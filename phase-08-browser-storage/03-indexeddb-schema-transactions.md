# IndexedDB: Schema Design & Transactions

## Quick Reference

| Concept | What it is | Key fact |
|---|---|---|
| Database | Named, versioned container for object stores | Each origin can have many; version triggers `upgradeneeded` |
| Object Store | Schemaless collection of records | Analogous to a table; records keyed by a `keyPath` or auto-increment |
| Index | Alternative lookup path over an object store | Enables queries by non-key fields; can enforce uniqueness |
| Transaction | Atomic unit of reads/writes | `readonly` or `readwrite`; auto-commits when all requests settle |
| Cursor | Iterator over a range of records | For iterating large result sets without loading all into memory |

---

## What Is This?

IndexedDB is the browser's full-featured, asynchronous, transactional database. It stores structured JavaScript values (objects, arrays, typed arrays, Blobs, Files, Dates — most things except functions) organized into **object stores** that can be indexed and queried by key range.

```js
// Minimal read flow using raw IDB
const request = indexedDB.open('my-db', 1);

request.onupgradeneeded = (e) => {
  const db = e.target.result;
  const store = db.createObjectStore('users', { keyPath: 'id' });
  store.createIndex('by-email', 'email', { unique: true });
};

request.onsuccess = (e) => {
  const db = e.target.result;
  const tx = db.transaction('users', 'readwrite');
  const store = tx.objectStore('users');

  store.put({ id: 1, name: 'Alice', email: 'alice@example.com' });

  tx.oncomplete = () => console.log('Saved');
  tx.onerror = (e) => console.error(e.target.error);
};
```

IndexedDB is the right tool when localStorage is too small, too slow (for large iteration), or lacks the structure you need.

> **Check yourself:** IndexedDB is asynchronous but doesn't use Promises natively. Why, and how does the ecosystem address this?

---

## Why Does It Exist?

localStorage peaked at ~5MB of strings. The web needed client-side storage for:
- **Large structured data**: offline documents, product catalogs, cached API responses
- **Binary data**: images, audio, video blobs
- **Indexed lookups**: "find all orders placed in the last 30 days" without loading everything into memory

WebSQL (SQLite in the browser) was the earlier attempt — it was deprecated because it tied the browser to a specific SQL implementation, making the spec unmaintainable. IndexedDB replaced it with a more abstract, lower-level API that browsers could implement without locking into SQLite.

The price of that abstraction: IndexedDB's raw API is famously verbose and callback-hell-prone. The ecosystem answered with wrapper libraries (Dexie.js, idb) that layer Promises on top.

---

## How It Works

### Opening a database and versioning

```js
const request = indexedDB.open('app-db', 3); // name, version
```

If the database doesn't exist, it's created. If the version number is higher than the stored version, `onupgradeneeded` fires — the only place where schema changes (creating/deleting object stores, indexes) are allowed.

```js
request.onupgradeneeded = (event) => {
  const db = event.target.result;
  const oldVersion = event.oldVersion; // 0 if new database

  if (oldVersion < 1) {
    db.createObjectStore('users', { keyPath: 'id', autoIncrement: false });
  }
  if (oldVersion < 2) {
    const store = event.target.transaction.objectStore('users');
    store.createIndex('by-email', 'email', { unique: true });
  }
  if (oldVersion < 3) {
    db.createObjectStore('sessions', { autoIncrement: true });
  }
};
```

Version migrations are additive — each `if (oldVersion < N)` block applies only the changes needed for that version. This pattern handles upgrades from any version, not just the immediate previous one.

### Object Stores

An object store is a collection of records. Records are identified by a **key**:

```js
// keyPath: a property in the stored object becomes the key
db.createObjectStore('users', { keyPath: 'id' });
// store.put({ id: 1, name: 'Alice' }) — 'id' is the key

// autoIncrement: a numeric key is generated automatically
db.createObjectStore('logs', { autoIncrement: true });
// store.add({ message: 'error' }) — key is auto-generated

// Out-of-line key: key is provided separately from the value
db.createObjectStore('blobs'); // no keyPath, no autoIncrement
// store.put(blob, 'avatar-123') — key is the second argument
```

### Indexes

An index lets you query records by a field other than the primary key:

```js
// In upgradeneeded:
const store = db.createObjectStore('users', { keyPath: 'id' });
store.createIndex('by-email', 'email', { unique: true });
store.createIndex('by-city', 'address.city', { unique: false });
store.createIndex('by-tags', 'tags', { multiEntry: true }); // array field
```

`multiEntry: true` means if `tags` is `['frontend', 'react']`, the record is indexed under both tags. Querying the `by-tags` index for `'frontend'` returns all records that have `'frontend'` in their `tags` array.

### Transactions

All reads and writes happen inside a transaction. Transactions are:
- **Scoped to specific object stores** — you declare which stores you need upfront
- **Typed as `readonly` or `readwrite`** — multiple readonly transactions run concurrently; readwrite transactions are serialized
- **Auto-committing** — they commit automatically when all requests complete and no new requests are added. You can abort explicitly.

```js
const tx = db.transaction(['users', 'orders'], 'readwrite');

const userStore = tx.objectStore('users');
const orderStore = tx.objectStore('orders');

// Queue requests — they execute atomically
userStore.put({ id: 1, balance: 100 });
orderStore.add({ userId: 1, total: 50, date: new Date() });

tx.oncomplete = () => console.log('Both saved atomically');
tx.onerror = () => tx.abort(); // roll back
```

**Transaction lifetime gotcha**: a transaction auto-commits as soon as all its queued requests complete. If you `await` something inside a transaction (a `fetch`, a timer, even a resolved promise that yields to the microtask queue after the current task), the transaction may auto-commit before you add the next request.

### Promises with the `idb` library

The raw IDB API uses events. The `idb` package (by Jake Archibald, ~1KB) wraps it in Promises:

```js
import { openDB } from 'idb';

const db = await openDB('app-db', 1, {
  upgrade(db) {
    db.createObjectStore('users', { keyPath: 'id' });
  },
});

await db.put('users', { id: 1, name: 'Alice' });
const user = await db.get('users', 1);
await db.delete('users', 1);

// Transactions still exist
const tx = db.transaction('users', 'readwrite');
await tx.store.put({ id: 2, name: 'Bob' });
await tx.done; // await transaction completion
```

Dexie.js is a heavier wrapper that adds a full query DSL, relationship support, and LiveQuery (reactive queries that re-run when data changes).

### Cursors — iterating large datasets

`getAll()` loads all matching records into memory. Cursors iterate one record at a time:

```js
const tx = db.transaction('users', 'readonly');
const store = tx.objectStore('users');
let cursor = await store.openCursor();

while (cursor) {
  console.log(cursor.key, cursor.value);
  cursor = await cursor.continue();
}
```

Key ranges limit the iteration:

```js
// Only users with id between 10 and 20
const range = IDBKeyRange.bound(10, 20);
let cursor = await store.openCursor(range);

// Iterate by index — users with email starting with 'a'
const idx = store.index('by-email');
const range = IDBKeyRange.bound('a', 'b', false, true); // ['a', 'b')
let cursor = await idx.openCursor(range);
```

> **Check yourself:** Why might `db.getAll('users')` cause problems in a large offline-first app?

---

## Schema Design Patterns

### Relational data via indexes

IDB is not relational — no joins. Model relationships through stored IDs and index lookups:

```js
// Object stores: users, orders
// orders has a 'userId' field, indexed
const orderIndex = db.transaction('orders').objectStore('orders').index('by-userId');
const userOrders = await orderIndex.getAll(IDBKeyRange.only(userId));
```

### Compound keys

IDB supports array compound keys natively:

```js
// Primary key is [userId, date]
db.createObjectStore('events', { keyPath: ['userId', 'date'] });

// Range queries on compound keys
const range = IDBKeyRange.bound([userId, startDate], [userId, endDate]);
store.getAll(range);
```

### Versioned records (audit log pattern)

```js
// keyPath: ['entityId', 'version']
db.createObjectStore('document-history', { keyPath: ['id', 'version'] });
// Allows querying all versions of a document efficiently
```

---

## Gotchas

**1. Transaction auto-commit on async yield**
If you `await fetch()` inside a transaction, the transaction commits before the fetch resolves — your subsequent `store.put()` throws `TransactionInactiveError`. Keep transactions synchronous; do I/O before or after, not during.

**2. Version must increase monotonically**
You cannot lower the version number. If you accidentally deploy version 5 and revert to version 4, users with version 5 in their browser will get an error on open. Migrations are one-way.

**3. Schema changes only in `upgradeneeded`**
`createObjectStore` and `createIndex` throw if called outside `upgradeneeded`. You cannot add an index in a regular transaction — you must bump the version.

**4. Blocking upgrades**
If a user has the DB open in another tab when a new version tries to open, `onblocked` fires. The upgrade can't proceed until all old-version connections are closed. Handle this: listen for `versionchange` events and close the DB, or show a "please close other tabs" prompt.

**5. Large values are still stored by value, not reference**
Storing a 50MB Blob in IDB stores 50MB of data. Writing a Blob doesn't transfer it — a copy is made. For very large files, store only metadata in IDB and use the File System Access API or Cache API for the actual bytes.

**6. Safari IDB bugs**
Safari has had historical IDB bugs — particularly around transactions completing prematurely and `openCursor` in certain patterns. Always test offline-first apps in Safari. The `idb` library papers over some of these bugs.

---

## Interview Questions

**Q (High): What is IndexedDB and when would you use it over localStorage?**

Answer: IndexedDB is the browser's transactional, async, structured data store. It stores typed JavaScript objects (not just strings), supports indexes for non-key field queries, handles binary data (Blobs, ArrayBuffers), has essentially no practical size limit (quota-governed, typically GBs), and is accessible from Workers. Use IndexedDB when: data is larger than localStorage's 5MB, you need to query by non-key fields, you're building offline-first features with large datasets, or you need to store binary files. Use localStorage when: you need a simple key-value store for small string data (preferences, settings) and don't need async access or Workers.

**Q (High): Explain IndexedDB transactions — what are they, and what is the auto-commit trap?**

Answer: A transaction is an atomic unit of reads and writes over a declared set of object stores. It's typed as `readonly` or `readwrite`. Transactions auto-commit when all queued requests settle and no new requests are added — this is the trap. If you `await` any asynchronous operation inside a transaction (including `fetch`, timers, or even an already-resolved Promise that yields via microtasks to a new task), the browser may commit the transaction before you add the next request. Subsequent `store.put()` calls throw `TransactionInactiveError`. The fix: complete all I/O before starting the transaction, or keep transaction code synchronous with no awaits except for the IDB requests themselves.

**Q (High): What is `onupgradeneeded` and how do you handle multi-version migrations?**

Answer: `onupgradeneeded` fires when the database version increases. It's the only place schema operations (creating/deleting object stores, indexes) are allowed. For multi-version migrations, use conditional `if (oldVersion < N)` blocks — each block applies changes for that schema version upgrade, and they're cumulative. This handles upgrades from any old version to the current one, not just version-minus-one. `event.oldVersion` is 0 for a brand-new database. Example: if you deploy version 3 and a user was on version 1, their upgrade runs all three blocks in sequence. Never assume the user is upgrading from the immediately previous version.

**Q (Medium): How do indexes work in IndexedDB? What is `multiEntry` for?**

Answer: An index is a separate lookup structure over an object store keyed by a specific property path. `createIndex('by-email', 'email', { unique: true })` builds an index keyed on the `email` property — queries on the index are efficient (B-tree lookup), not full scans. `multiEntry: true` is for array-valued fields: if a record has `tags: ['frontend', 'react']`, it's indexed once under each tag value. Without `multiEntry`, the array itself is the key — a range query for `'frontend'` wouldn't match it. With `multiEntry`, querying the index for `'frontend'` finds all records containing that tag in their array.

**Q (Medium): Why is `getAll()` potentially problematic in an offline-first app?**

Answer: `getAll()` loads the entire result set into memory at once. If the object store contains thousands of large records (cached documents, images as DataURLs), `getAll()` allocates all of it in the JS heap simultaneously. For large datasets, use a cursor: `openCursor()` iterates one record at a time, keeping only one record in memory. Combine with `openCursor(range)` to limit to a key range. For counts without loading data, use `store.count(range)`. The pattern is to load only what you need, paginated by key range.

**Q (Low): What happens when a user has IndexedDB open in Tab A and your app tries to open a new version in Tab B?**

Answer: Tab B fires `onblocked` — the version upgrade cannot proceed while an older version has open connections. The database stays at the old version in Tab A. To handle this gracefully: listen for the `versionchange` event on the open database (this fires in Tab A when a newer version is requested elsewhere) and close the connection. Show a "please reload your other tabs" prompt. Without this handler, the user in Tab B is stuck. Libraries like idb and Dexie expose this as a `blocking` callback option on `openDB`.

---

## Self-Assessment

- [ ] Describe the `onupgradeneeded` pattern for multi-version migration from memory
- [ ] Explain the transaction auto-commit trap and how to avoid it
- [ ] Write the correct pattern to iterate all records with a cursor using the `idb` library
- [ ] Explain what `multiEntry: true` does on an index over an array field
- [ ] Name three types of data that IndexedDB can store that localStorage cannot

---
*Next: Cache API — the service worker layer's programmable HTTP cache, and how it differs from the browser's native HTTP cache.*
