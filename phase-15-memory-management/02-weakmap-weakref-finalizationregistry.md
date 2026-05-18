# WeakMap, WeakRef & FinalizationRegistry

## Quick Reference

| API | Key Type | Value Held | Use Case |
|---|---|---|---|
| `WeakMap` | Objects only | Strongly | Private per-object metadata without preventing GC |
| `WeakSet` | Objects only | Strongly | Tracking a set of objects without preventing GC |
| `WeakRef` | Objects only | Weakly | Optional cache — access if still alive, else recompute |
| `FinalizationRegistry` | Objects only | — | Cleanup callback after GC, debugging, releasing resources |

---

## Why These APIs Exist

Regular `Map` and `Set` hold **strong references** to their keys and values — the referenced objects can't be collected as long as the collection exists. This is the right default most of the time.

But sometimes you want to attach data to an object for the *lifetime of that object*, not longer. If you store metadata about DOM nodes, for example, a regular Map keeps those nodes alive even after they're removed from the document. `WeakMap` solves this: when the node goes out of scope and has no other strong references, the GC can collect it and the `WeakMap` entry disappears automatically.

`WeakRef` and `FinalizationRegistry` go further — they let you hold optional references to objects and react when the GC reclaims them.

---

## WeakMap

A `WeakMap` maps objects to values. The key difference from `Map`: `WeakMap` **does not prevent its keys from being garbage collected**. When the key object is collected, the entry is automatically removed.

Constraints that enable this behavior:
- Keys must be objects (not primitives)
- Not iterable — you can't list all keys (because they might be collected at any time)
- No `size` property

```typescript
const metadata = new WeakMap<HTMLElement, { clicks: number; created: number }>();

function trackElement(el: HTMLElement): void {
  metadata.set(el, { clicks: 0, created: Date.now() });
}

function onClick(el: HTMLElement): void {
  const data = metadata.get(el);
  if (data) data.clicks++;
}

// When `el` is removed from the DOM and no other references exist,
// both `el` and its metadata entry are collected.
// No manual cleanup needed.
```

**Primary use case: private per-instance data**

Before ES private fields (`#field`), `WeakMap` was the standard pattern for true privacy:

```typescript
type Point = { x: number; y: number };

const _internals = new WeakMap<object, { secret: string }>();

class SecureBox {
  constructor(secret: string) {
    _internals.set(this, { secret });
  }

  reveal(): string {
    return _internals.get(this)!.secret;
  }
}

const box = new SecureBox('hidden');
box.reveal(); // 'hidden'
// No way to access _internals from outside the module
```

**Secondary use case: associating data with DOM nodes without leaking**

```typescript
// Store component state alongside DOM nodes in a library
const componentState = new WeakMap<Element, { mounted: boolean; props: Record<string, unknown> }>();

function mount(el: Element, props: Record<string, unknown>): void {
  componentState.set(el, { mounted: true, props });
}

function unmount(el: Element): void {
  const state = componentState.get(el);
  if (state) state.mounted = false;
  el.remove();
  // No need to call componentState.delete(el) — when el is GC'd,
  // the entry disappears automatically.
}
```

> **Check yourself:** Why can't you iterate over a `WeakMap`? What would happen if you could?

---

## WeakSet

A `WeakSet` is a set of objects where membership doesn't prevent GC. Like `WeakMap`, it's not iterable and has no `size`.

**Primary use case: membership tracking without retention**

```typescript
const processing = new WeakSet<Request>();

async function processRequest(req: Request): Promise<void> {
  if (processing.has(req)) return; // already in flight
  processing.add(req);

  try {
    await handleRequest(req);
  } finally {
    processing.delete(req);
    // When req goes out of scope, it's collected regardless
  }
}
```

```typescript
// Track which nodes have been initialized without holding them alive
const initialized = new WeakSet<HTMLElement>();

function initOnce(el: HTMLElement): void {
  if (initialized.has(el)) return;
  initialized.add(el);
  // setup...
}
```

---

## WeakRef

`WeakRef` holds a **weak reference** to an object — it doesn't prevent GC. You dereference it with `.deref()`, which returns the object if it's still alive, or `undefined` if it's been collected.

```typescript
class ImageCache {
  private cache = new Map<string, WeakRef<HTMLImageElement>>();

  set(url: string, img: HTMLImageElement): void {
    this.cache.set(url, new WeakRef(img));
  }

  get(url: string): HTMLImageElement | undefined {
    const ref = this.cache.get(url);
    if (!ref) return undefined;

    const img = ref.deref();
    if (img === undefined) {
      // GC collected the image — evict the stale entry
      this.cache.delete(url);
      return undefined;
    }

    return img;
  }
}
```

**Important caveats about `WeakRef`:**

1. **GC timing is non-deterministic.** `.deref()` may return the object on one call and `undefined` on the next — you have no control over when the GC runs.
2. **Use only as a cache fallback.** You must be prepared to recompute/reload the value when `.deref()` returns `undefined`.
3. **Not a replacement for strong references** when you need the object to stay alive.

```typescript
// Correct pattern: always have a fallback
function getOrLoad(cache: ImageCache, url: string): HTMLImageElement {
  const cached = cache.get(url);
  if (cached) return cached;

  // Fallback: create fresh
  const img = new Image();
  img.src = url;
  cache.set(url, img);
  return img;
}
```

> **Check yourself:** If `.deref()` returns a value, does that guarantee the object won't be collected before you use it in the next line?

---

## FinalizationRegistry

`FinalizationRegistry` lets you register a cleanup callback that fires **after** an object is garbage collected.

```typescript
const registry = new FinalizationRegistry<string>((heldValue) => {
  console.log(`Object with token "${heldValue}" was collected`);
  // Free associated native resources, close file handles, etc.
});

class Resource {
  constructor(name: string) {
    registry.register(this, name); // register with a held value (not the object itself)
  }
}

{
  const r = new Resource('db-connection');
  // r goes out of scope here
}
// At some future point: "Object with token "db-connection" was collected"
```

**Critical constraints:**

- **Callbacks are not guaranteed.** If the tab closes or the process exits, registered callbacks may never fire.
- **Callbacks are non-deterministic.** They run at GC time, which can be long after the object becomes unreachable.
- **Don't store strong references to the registered object in the callback.** The whole point is that the object is being collected.
- **Not a substitute for explicit cleanup.** Use explicit `dispose()` / `destroy()` patterns as the primary cleanup mechanism. `FinalizationRegistry` is a safety net, not a design pattern.

**Practical use: releasing native/external resources**

```typescript
// Hypothetical WebGPU texture wrapper
const gpuRegistry = new FinalizationRegistry<GPUTexture>((texture) => {
  texture.destroy(); // release GPU memory if JS wrapper was GC'd without explicit cleanup
});

class ManagedTexture {
  private texture: GPUTexture;

  constructor(device: GPUDevice, descriptor: GPUTextureDescriptor) {
    this.texture = device.createTexture(descriptor);
    gpuRegistry.register(this, this.texture);
  }

  dispose(): void {
    this.texture.destroy();
    // ideally also call registry.unregister(this) to cancel the callback
  }
}
```

**Use `FinalizationRegistry` for:**
- Debugging: tracking which objects are being collected and when
- Safety-net cleanup for external/native resources
- Detecting leaks by observing when expected objects are *not* collected

---

## When to Use Each

```
Need to attach data to an object for its lifetime?
  → WeakMap

Need to track membership without holding the object alive?
  → WeakSet

Need a cache that automatically evicts stale entries when memory is needed?
  → WeakRef + Map (with deref() fallback)

Need a callback when an object is collected?
  → FinalizationRegistry (as safety net, never as primary cleanup)
```

---

## Self-Assessment

- [ ] I can explain why `WeakMap` keys cannot be primitives
- [ ] I understand why `WeakMap` is not iterable and why that's necessary
- [ ] I know the primary use case for `WeakMap` (per-instance metadata without retention)
- [ ] I understand that `WeakRef.deref()` can return `undefined` at any call
- [ ] I can implement a GC-friendly cache using `WeakRef`
- [ ] I know why `FinalizationRegistry` callbacks are a safety net, not a primary cleanup strategy

---

## Interview Q&A

**Q: What problem does `WeakMap` solve that a regular `Map` doesn't?** `High`

A regular `Map` holds strong references to its keys. If you use a `Map` to attach metadata to objects (e.g., DOM nodes), those objects can never be garbage collected as long as the `Map` exists — even if you've removed them from the DOM and have no other references. `WeakMap` holds weak references to its keys: when the key object has no other strong references, the GC can collect it and the entry disappears automatically. This makes `WeakMap` the right choice for per-object metadata that should live only as long as the object itself.

---

**Q: Why can't you iterate over a `WeakMap` or get its `size`?** `High`

Because iteration would require enumerating all keys, but at any point during iteration the GC could collect a key (since `WeakMap` doesn't prevent this). The result would be non-deterministic — the set of keys could change mid-iteration. Not supporting iteration is the spec's way of making the weak semantics consistent: you can only access an entry if you already have a strong reference to the key, which is by design.

---

**Q: When would you use `WeakRef` and what's the most important thing to remember about it?** `Medium`

`WeakRef` is for optional caches where the cached object doesn't need to stay alive. You hold a weak reference to something expensive to compute/load; if the GC collects it, you reload it. The most important thing: `.deref()` can return `undefined` at any call, and you have zero control over when. You must always have a fallback path. Never use `WeakRef` when you need the object to stay alive — use a strong reference. Never use it as the primary reference to something the user is actively interacting with.

---

**Q: Can you rely on `FinalizationRegistry` to always fire?** `Medium`

No. The spec explicitly does not guarantee that cleanup callbacks will run — if the page closes or the process exits, they may not. They also run at an unpredictable time after collection, potentially much later. `FinalizationRegistry` is a safety net for releasing external resources that the GC can't automatically free (GPU buffers, file handles, native objects). Primary resource cleanup must always be explicit — a `dispose()` method, a `using` declaration (ES2023), or framework lifecycle hooks.

---

**Q: How does `WeakMap` enable private class fields before ES private syntax?** `Low`

A module-level `WeakMap` keyed by `this` (or any object instance) is inaccessible from outside the module — it's a closure variable. Code outside the module can't access the map, and therefore can't read the stored values, even with full access to the instance. Unlike properties on `this`, there's no way to enumerate or access these values through the object itself. This was the standard pattern before `#privateField` syntax was introduced, and it still appears in library code that needs to support environments without private fields.
