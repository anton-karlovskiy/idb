# IndexedDB with usability.

This is a tiny (~1.16k) library that mostly mirrors the IndexedDB API, but with small improvements that make a big difference to usability.

1. [Installation](#installation)
1. [Changes](#changes)
1. [API](#api)
    1. [`openDB`](#opendb)
    1. [`deleteDB`](#deletedb)
    1. [`unwrap`](#unwrap)
    1. [`wrap`](#wrap)
    1. [General enhancements](#general-enhancements)
    1. [`IDBDatabase` enhancements](#idbdatabase-enhancements)
    1. [`IDBTransaction` enhancements](#idbtransaction-enhancements)
    1. [`IDBCursor` enhancements](#idbcursor-enhancements)
    1. [Async iterators](#async-iterators)
1. [Examples](#examples)
1. [TypeScript](#typescript)

# Installation

```sh
npm install idb
```

Then, assuming you're using a module-compatible system (like webpack, Rollup etc):

```js
import { openDB, deleteDB, wrap, unwrap } from 'idb';

async function doDatabaseStuff() {
  const db = await openDB(…);
}
```

Or, use it directly via unpkg:

```html
<script type="module">
import { openDB, deleteDB, wrap, unwrap } from 'https://unpkg.com/idb?module';

async function doDatabaseStuff() {
  const db = await openDB(…);
}
</script>
```

# Changes

This is a rewrite from 3.x with substantial changes. [See details](changes.md).

# API

## `openDB`

This method opens a database, and returns a promise for an enhanced [`IDBDatabase`](https://w3c.github.io/IndexedDB/#database-interface).

```js
const db = await openDB(name, version, {
  upgrade(db, oldVersion, newVersion, transaction) {
    // …
  },
  blocked() {
    // …
  },
  blocking() {
    // …
  }
});
```

* `name`: Name of the database.
* `version`: Schema version.
* `upgrade` (optional): Called if this version of the database has never been opened before. Use it to specify the schema for the database. This is similar to the [`upgradeneeded` event](https://developer.mozilla.org/en-US/docs/Web/API/IDBOpenDBRequest/upgradeneeded_event) in plain IndexedDB.
    * `db`: An enhanced `IDBDatabase`.
    * `oldVersion`: Last version of the database opened by the user.
    * `newVersion`: Whatever new version you provided.
    * `transaction`: An enhanced transaction for this upgrade. This is useful if you need to get data from other stores as part of a migration.
* `blocked` (optional): Called if there are older versions of the database open on the origin, so this version cannot open. This is similar to the [`blocked` event](https://developer.mozilla.org/en-US/docs/Web/API/IDBOpenDBRequest/blocked_event) in plain IndexedDB.
* `blocking` (optional): Called if this connection is blocking a future version of the database from opening. This is similar to the [`versionchange` event](https://developer.mozilla.org/en-US/docs/Web/API/IDBDatabase/versionchange_event) in plain IndexedDB.

## `deleteDB`

Deletes a database.

```js
await deleteDB(name);
```

* `name`: Name of the database.

## `unwrap`

Takes an enhanced IndexedDB object and returns the plain unmodified one.

```js
const unwrapped = unwrap(wrapped);
```

This is useful if, for some reason, you want to drop back into plain IndexedDB. Promises will also be converted back into `IDBRequest` objects.

## `wrap`

Takes an IDB object and returns a version enhanced by this library.

```js
const wrapped = wrap(unwrapped);
```

This is useful if some third party code gives you an `IDBDatabase` object and you want it to have the features of this library.

This doesn't work with `IDBCursor`, [due to missing primitives](https://github.com/w3c/IndexedDB/issues/255). Also, if you wrap an `IDBTransaction`, `tx.store` and `tx.objectStoreNames` won't work in Edge. To avoid these issues, wrap the `IDBDatabase` object, and use the wrapped object to create a new transaction.

## General enhancements

Once you've opened the database the API is the same as IndexedDB, except for a few changes to make things easier.

Firstly, any method that usually returns an `IDBRequest` object will now return a promise for the result.

```js
const store = db.transaction(storeName).objectStore(storeName);
const value = await store.get(key);
```

### Promises & throwing

The library turns all `IDBRequest` objects into promises, but it doesn't know in advance which methods may return promises.

As a result, methods such as `store.put` may throw instead of returning a promise.

If you're using async functions, there's no observable difference.

### Transaction lifetime

TL;DR: **Do not `await` other things between the start and end of your transaction**, otherwise the transaction will close before you're done.

An IDB transaction auto-closes if it doesn't have anything left do once microtasks have been processed. As a result, this works fine:

```js
const tx = db.transaction('keyval', 'readwrite');
const store = tx.objectStore('keyval');
const val = await store.get('counter') || 0;
store.put(val + 1, 'counter');
await tx.done;
```

But this doesn't:

```js
const tx = db.transaction('keyval', 'readwrite');
const store = tx.objectStore('keyval');
const val = await store.get('counter') || 0;
// This is where things go wrong:
const newVal = await fetch('/increment?val=' + val);
// And this throws an error:
store.put(newVal, 'counter');
await tx.done;
```

In this case, the transaction closes while the browser is fetching, so `store.put` fails.

## `IDBDatabase` enhancements

### Shortcuts to get/set from an object store

It's common to create a transaction for a single action, so helper methods are included for this:

```js
// Get a value from a store:
const value = await db.get(storeName, key);
// Set a value in a store:
await db.put(storeName, value, key);
```

The shortcuts are: `get`, `getKey`, `getAll`, `getAllKeys`, `count`, `put`, `add`, `delete`, and `clear`. Each method takes a `storeName` argument, the name of the object store, and the rest of the arguments are the same as the equivalent `IDBObjectStore` method.

These methods depend on the same methods on `IDBObjectStore`, therefore `getKey`, `getAll`, and `getAllKeys` are missing in Edge.

### Shortcuts to get from an index

The shortcuts are: `getFromIndex`, `getKeyFromIndex`, `getAllFromIndex`, `getAllKeysFromIndex`, and `countFromIndex`.

```js
// Get a value from an index:
const value = await db.getFromIndex(storeName, indexName, key);
```

Each method takes `storeName` and `indexName` arguments, followed by the rest of the arguments from the equivalent `IDBIndex` method. Again, these methods depend on the equivalent methods on `IDBIndex`, so Edge does not support `getKeyFromIndex`, `getAllFromIndex`, or `getAllKeysFromIndex`.

## `IDBTransaction` enhancements

### `tx.store`

If a transaction involves a single store, the `store` property will reference that store.

```js
const tx = db.transaction('whatever');
const store = tx.store;
```

If a transaction involves multiple stores, `tx.store` is undefined, you need to use `tx.objectStore(storeName)` to get the stores.

### `tx.done`

Transactions have a `.done` promise which resolves when the transaction completes successfully, and otherwise rejects with the [transaction error](https://developer.mozilla.org/en-US/docs/Web/API/IDBTransaction/error).

```js
const tx = db.transaction(storeName, 'readwrite');
tx.store.put('foo', 'bar');
await tx.done;
```

## `IDBCursor` enhancements

Cursor advance methods (`advance`, `continue`, `continuePrimaryKey`) return a promise for the cursor, or null if there are no further values to provide.

```js
let cursor = await db.transaction(storeName).store.openCursor();

while (cursor) {
  console.log(cursor.key, cursor.value);
  cursor = await cursor.continue();
}
```

## Async iterators

Async iterator support isn't included by default (Edge doesn't support them). To include them, import `idb/with-async-ittr.js` instead of `idb` (this increases the library size to ~1.38k):

```js
import { openDB } from 'idb/with-async-ittr.js';
```

Now you can iterate over stores, indexes, and cursors:

```js
const tx = db.transaction(storeName);

for await (const cursor of tx.store) {
  // …
}
```

Each yielded object is an `IDBCursor`. You can optionally use the advance methods to skip items (within an async iterator they return void):

```js
const tx = db.transaction(storeName);

for await (const cursor of tx.store) {
  console.log(cursor.value);
  // Skip the next item
  cursor.advance(2);
}
```

If you don't manually advance the cursor, `cursor.continue()` is called for you.

Stores and indexes also have an `iterate` method which has the same signature as `openCursor`, but returns an async iterator:

```js
const index = db.transaction('books').store.index('author');

for await (const cursor of index.iterate('Douglas Adams')) {
  console.log(cursor.value);
}
```

# Examples

## Keyval store

This is very similar to `localStorage`, but async. If this is *all* you need, you may be interested in [idb-keyval](https://www.npmjs.com/package/idb-keyval). You can always upgrade to this library later.

```js
const dbPromise = openDB('keyval-store', 1, {
  upgrade(db) {
    db.createObjectStore('keyval');
  }
});

const idbKeyval = {
  async get(key) {
    return (await dbPromise).get('keyval', key);
  },
  async set(key, val) {
    return (await dbPromise).put('keyval', val, key);
  },
  async delete(key) {
    return (await dbPromise).delete('keyval', key);
  },
  async clear() {
    return (await dbPromise).clear('keyval');
  },
  async keys() {
    return (await dbPromise).getAllKeys('keyval');
  },
};
```

## Article store

```js
async function demo() {
  const db = await openDB('Articles', 1, {
    upgrade(db) {
      // Create a store of objects
      const store = db.createObjectStore('articles', {
        // The 'id' property of the object will be the key.
        keyPath: 'id',
        // If it isn't explicitly set, create a value by auto incrementing.
        autoIncrement: true,
      });
      // Create an index on the 'date' property of the objects.
      store.createIndex('date', 'date');
    },
  });

  // Add an article:
  await db.add('articles', {
    title: 'Article 1',
    date: new Date('2019-01-01'),
    body: '…',
  });

  // Add multiple articles in one transaction:
  {
    const tx = db.transaction('articles', 'readwrite');
    tx.store.add({
      title: 'Article 2',
      date: new Date('2019-01-01'),
      body: '…',
    });
    tx.store.add({
      title: 'Article 3',
      date: new Date('2019-01-02'),
      body: '…',
    });
    await tx.done;
  }

  // Get all the articles in date order:
  console.log(await db.getAllFromIndex('articles', 'date'));

  // Add 'And, happy new year!' to all articles on 2019-01-01:
  {
    const tx = db.transaction('articles', 'readwrite');
    const index = tx.store.index('date');

    for await (const cursor of index.iterate(new Date('2019-01-01'))) {
      const article = { ...cursor.value };
      article.body += ' And, happy new year!';
      cursor.update(article);
    }

    await tx.done;
  }
}
```

# TypeScript

This library is fully typed, and you can improve things by providing types for your database:

```ts
import { openDB, DBSchema } from 'idb';

interface MyDB extends DBSchema {
  'favourite-number': {
    key: string,
    value: number,
  },
  'products': {
    value: {
      name: string,
      price: number,
      productCode: string,
    },
    key: string,
    indexes: { 'by-price': number },
  }
}

async function demo() {
  const db = await openDB<MyDB>('my-db', 1, {
    upgrade(db) {
      db.createObjectStore('favourite-number');

      const productStore = db.createObjectStore('products', { keyPath: 'productCode' });
      productStore.createIndex('by-price', 'price');
    }
  });

  // This works
  await db.put('favourite-number', 7, 'Jen');
  // This fails at compile time, as the 'favourite-number' store expects a number.
  await db.put('favourite-number', 'Twelve', 'Jake');
}
```

To define types for your database, extend `DBSchema` with an interface where the keys are the names of your object stores.

For each value, provide an object where `value` is the type of values within the store, and `key` is the type of keys within the store.

Optionally, `indexes` can contain a map of index names, to the type of key within that index.

Provide this interface when calling `openDB`, and from then on your database will be strongly typed. This also allows your IDE to autocomplete the names of stores and indexes.

## Opting out of types

If you call `openDB` without providing types, your database will use basic types. However, sometimes you'll need to interact with stores that aren't in your schema, perhaps during upgrades. In that case you can cast.

Let's say we were renaming the 'favourite-number' store to 'fave-nums':

```ts
import { openDB, DBSchema, IDBPDatabase } from 'idb';

interface MyDBV1 extends DBSchema {
  'favourite-number': { key: string, value: number },
}

interface MyDBV2 extends DBSchema {
  'fave-num': { key: string, value: number },
}

const db = await openDB<MyDBV2>('my-db', 2, {
  async upgrade(db, oldVersion) {
    // Cast a reference of the database to the old schema.
    const v1Db = db as unknown as IDBPDatabase<MyDBV1>;

    if (oldVersion < 1) {
      v1Db.createObjectStore('favourite-number');
    }
    if (oldVersion < 2) {
      const store = v1Db.createObjectStore('favourite-number');
      store.name = 'fave-num';
    }
  }
});
```

You can also cast to a typeless database by omiting the type, eg `db as IDBPDatabase`.

Note: Types like `IDBPDatabase` are used by TypeScript only. The implementation uses proxies under the hood.
