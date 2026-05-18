# Object Pooling for High-frequency Allocations

## Quick Reference

| Concept | Description |
|---|---|
| Object pool | Pre-allocated reservoir of reusable objects — borrow, use, return |
| GC pressure | Frequency of allocations that triggers garbage collection pauses |
| Hot path | Code executed many times per frame (render loop, event handlers) |
| Pool size | Fixed upper bound to avoid unbounded growth |

---

## Why Allocation in Hot Paths Matters

The garbage collector pauses JavaScript execution to reclaim memory. In most application code, this is imperceptible. In code that runs many times per frame — game loops, animation callbacks, particle systems, WebSocket message handlers — GC pauses cause visible frame drops.

The problem isn't any single allocation. It's the cumulative allocation rate: if your animation loop allocates 100 short-lived objects per frame at 60fps, that's 6,000 allocations per second constantly feeding the GC.

Object pooling sidesteps this: instead of allocating and discarding, you pre-allocate a fixed number of objects and reuse them. The GC has nothing to collect because nothing is being abandoned.

---

## When Pooling Is (and Isn't) Appropriate

**Use pooling when:**
- You're in a loop that runs 30–60+ times per second
- You're allocating many objects of the same shape per iteration
- You have measurable GC-induced frame drops (visible in the Performance timeline as GC events causing long tasks)

**Don't use pooling when:**
- The code runs occasionally (user events, route changes)
- Object count is low (< 100/second)
- Pool management complexity would outweigh the benefit

Pooling is premature optimization in most application code. It's appropriate in game loops, physics simulations, audio processing, and high-frequency data visualization.

---

## Basic Object Pool

```typescript
class Pool<T extends object> {
  private available: T[] = [];
  private factory: () => T;
  private reset: (obj: T) => void;
  private maxSize: number;

  constructor(factory: () => T, reset: (obj: T) => void, maxSize = 100) {
    this.factory = factory;
    this.reset = reset;
    this.maxSize = maxSize;
  }

  acquire(): T {
    if (this.available.length > 0) {
      return this.available.pop()!;
    }
    return this.factory();
  }

  release(obj: T): void {
    if (this.available.length >= this.maxSize) {
      return; // pool full — let it be GC'd rather than grow unbounded
    }
    this.reset(obj);
    this.available.push(obj);
  }

  get size(): number {
    return this.available.length;
  }
}
```

Usage:

```typescript
interface Particle {
  x: number;
  y: number;
  vx: number;
  vy: number;
  life: number;
}

const particlePool = new Pool<Particle>(
  () => ({ x: 0, y: 0, vx: 0, vy: 0, life: 0 }), // factory
  (p) => { p.x = 0; p.y = 0; p.vx = 0; p.vy = 0; p.life = 0; }, // reset
  500 // max pool size
);

class ParticleSystem {
  private active: Particle[] = [];

  emit(x: number, y: number): void {
    const p = particlePool.acquire(); // borrow from pool
    p.x = x;
    p.y = y;
    p.vx = (Math.random() - 0.5) * 4;
    p.vy = -Math.random() * 5;
    p.life = 1.0;
    this.active.push(p);
  }

  update(dt: number): void {
    for (let i = this.active.length - 1; i >= 0; i--) {
      const p = this.active[i];
      p.x += p.vx * dt;
      p.y += p.vy * dt;
      p.life -= dt;

      if (p.life <= 0) {
        this.active.splice(i, 1);
        particlePool.release(p); // return to pool — no GC
      }
    }
  }
}
```

---

## Pre-warming a Pool

If you know how many objects you'll need, pre-allocate them before the hot path begins:

```typescript
function prewarm<T extends object>(pool: Pool<T>, count: number): void {
  const temp: T[] = [];
  for (let i = 0; i < count; i++) {
    temp.push(pool.acquire());
  }
  for (const obj of temp) {
    pool.release(obj); // fill the pool with pre-allocated objects
  }
}

// Before starting the game loop:
prewarm(particlePool, 500);
```

Pre-warming ensures the factory function (which allocates new objects) only runs during initialization, never during the frame loop.

---

## TypedArray Pooling for Binary Data

For numeric data (coordinates, colors, matrices), `TypedArray`s are more memory-efficient than plain objects. You can pool `ArrayBuffer`s:

```typescript
class BufferPool {
  private pools = new Map<number, ArrayBuffer[]>();

  acquire(byteLength: number): ArrayBuffer {
    const available = this.pools.get(byteLength);
    if (available && available.length > 0) {
      return available.pop()!;
    }
    return new ArrayBuffer(byteLength);
  }

  release(buffer: ArrayBuffer): void {
    const available = this.pools.get(buffer.byteLength) ?? [];
    // Zero out the buffer before returning to pool
    new Uint8Array(buffer).fill(0);
    available.push(buffer);
    this.pools.set(buffer.byteLength, available);
  }
}

const bufferPool = new BufferPool();

// In animation loop — no new ArrayBuffer allocation:
function processFrame(dataSize: number): void {
  const buffer = bufferPool.acquire(dataSize * 4); // Float32: 4 bytes each
  const view = new Float32Array(buffer);

  // ... fill and process view ...

  bufferPool.release(buffer); // return for reuse
}
```

---

## Event Object Pooling

DOM events are created by the browser and can't be pooled, but synthetic event-like objects in your own event system can be:

```typescript
interface GameEvent {
  type: string;
  target: string;
  payload: unknown;
}

const eventPool = new Pool<GameEvent>(
  () => ({ type: '', target: '', payload: null }),
  (e) => { e.type = ''; e.target = ''; e.payload = null; }
);

class EventBus {
  private handlers = new Map<string, Array<(e: GameEvent) => void>>();

  emit(type: string, target: string, payload: unknown): void {
    const event = eventPool.acquire();
    event.type = type;
    event.target = target;
    event.payload = payload;

    const listeners = this.handlers.get(type) ?? [];
    for (const handler of listeners) {
      handler(event);
    }

    eventPool.release(event); // safe only if handlers are synchronous
    // Do NOT pool if handlers might be async or store the event
  }
}
```

> **Check yourself:** Why is it unsafe to release a pooled event object back to the pool if an event handler stores a reference to it?

---

## Common Mistakes

**Mistake 1: Storing references to released objects**

```typescript
// WRONG
const event = eventPool.acquire();
later.push(event); // store reference
eventPool.release(event); // release — now `later[i]` points to a recycled object
// `event` will be overwritten when someone else acquires it
```

Once you release an object, all references to it must be dropped. The pool may give that object to another caller immediately.

**Mistake 2: Unbounded pool growth**

```typescript
// WRONG — pool grows without limit if release rate > acquire rate
release(obj: T): void {
  this.available.push(obj); // no size check
}
```

Always cap pool size. If the pool is full, let the extra object be collected normally — it's rare and the GC handles it fine.

**Mistake 3: Partial reset**

```typescript
// WRONG — forgot to reset one field
const resetParticle = (p: Particle) => {
  p.x = 0;
  p.y = 0;
  // forgot: p.life = 0
};
// p.life still has the old value from the previous use
```

The reset function must clear every field that matters. Stale data from a previous use is a common source of subtle bugs.

**Mistake 4: Pooling when it doesn't help**

Object pools add complexity. Profile first — if the Performance timeline doesn't show GC as a significant contributor to frame time, pooling is not the right optimization. Most applications never need it.

---

## Self-Assessment

- [ ] I understand why high allocation rates cause GC pauses in animation loops
- [ ] I can implement a generic pool with factory, reset, and max-size cap
- [ ] I know why pre-warming eliminates factory calls from the hot path
- [ ] I understand why you must drop all references to a pooled object after releasing it
- [ ] I can identify when pooling is appropriate vs. when it's premature optimization
- [ ] I know that `TypedArray` pools are useful for numeric/binary data in tight loops

---

## Interview Q&A

**Q: Why do frequent allocations cause frame drops, and how does object pooling address this?** `High`

JavaScript's garbage collector runs periodically to reclaim memory from objects that are no longer reachable. When the GC runs, it pauses JavaScript execution — this is the "GC pause." In code that allocates many short-lived objects per frame (like a particle system or physics loop), the allocation rate is high enough that the GC runs frequently, causing visible frame drops. Object pooling keeps a reservoir of pre-allocated objects and reuses them. Since released objects aren't discarded, there's nothing for the GC to collect, eliminating GC pauses in the hot path.

---

**Q: What's the most critical rule when returning an object to a pool?** `High`

All external references to the object must be dropped before or immediately after releasing it. The pool may give that object to another caller on the very next `acquire()` call. If any part of the code still holds a reference and reads from it, it will see stale or overwritten data from the new use. This is why pooling works cleanest with synchronous, scope-limited patterns: acquire at the top of a block, use within the block, release at the end — never storing the reference elsewhere.

---

**Q: When should you NOT use object pooling?** `Medium`

When the allocation frequency is low enough that GC pauses are not measurable. Pooling adds complexity: you need to maintain the pool, write reset functions, guard against reference aliasing after release, and cap pool size. For most application code — user events, route changes, data fetching — allocations happen rarely enough that the GC handles them imperceptibly. Pool only when you have profiler evidence that GC is causing frame drops, typically in animation loops, game engines, or high-frequency data visualization.

---

**Q: Why would you pre-warm a pool before the game loop starts?** `Medium`

Pre-warming fills the pool by calling the factory function during initialization. Once warm, all `acquire()` calls return existing objects from the pool without allocating. The factory function — which does allocate — only runs during startup, never in the frame loop. Without pre-warming, the pool starts empty and the factory runs during early frames, causing allocations precisely when you're trying to avoid them.

---

**Q: How do `TypedArray`s compare to plain objects for pooling in numeric hot paths?** `Low`

`TypedArray`s (like `Float32Array`) store values in contiguous memory with no per-element overhead — just the raw numeric bytes. Plain objects have per-property overhead and are stored as heap objects with type tags and property descriptors. For numeric data (vectors, colors, matrices), `TypedArray`s have better memory density and CPU cache behavior since contiguous memory access is faster than pointer chasing through object properties. Pooling `ArrayBuffer`s is straightforward: acquire a buffer, create a typed view over it, use it, zero it out, release back to pool.
