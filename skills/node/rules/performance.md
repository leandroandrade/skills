---
name: performance
description: Performance optimization techniques
metadata:
  tags: performance, optimization, memory, worker-threads
---

# Performance in Node.js

## Avoid Blocking the Event Loop

Never perform CPU-intensive operations synchronously:

```javascript
// BAD - blocks event loop
function hashPasswordSync(password) {
  return crypto.pbkdf2Sync(password, salt, 100000, 64, 'sha512').toString('hex');
}

// GOOD - async operation
async function hashPassword(password) {
  return new Promise((resolve, reject) => {
    crypto.pbkdf2(password, salt, 100000, 64, 'sha512', (err, key) => {
      if (err) reject(err);
      else resolve(key.toString('hex'));
    });
  });
}
```

## Worker Threads with Piscina

Use [piscina](https://github.com/piscinajs/piscina) for CPU-intensive tasks:

```javascript
// worker.js
export default function heavyComputation(data) {
  // CPU-intensive work here
  return result;
}
```

```javascript
// main.js
import Piscina from 'piscina';

const piscina = new Piscina({
  filename: new URL('./worker.js', import.meta.url).href,
});

const result = await piscina.run({ input: 'data' });
```

Piscina handles worker pool management, task queuing, and load balancing automatically.

## Connection Pooling

Always use connection pools for databases:

```javascript
import { Pool } from 'pg';

const pool = new Pool({
  max: 20,
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 2000,
});

async function query(sql, params) {
  const client = await pool.connect();
  try {
    const result = await client.query(sql, params);
    return result.rows;
  } finally {
    client.release();
  }
}
```

## Avoid Memory Leaks

Common memory leak patterns to avoid:

```javascript
// BAD - unbounded cache
const cache = new Map();
function addToCache(key, value) {
  cache.set(key, value); // Never cleaned up
}

// GOOD - LRU cache with max size
import { LRUCache } from 'lru-cache';

const cache = new LRUCache({
  max: 500,
  ttl: 1000 * 60 * 5,
});

// BAD - listener leak
function subscribe(emitter) {
  emitter.on('event', handler); // Never removed
}

// GOOD - cleanup listeners
function subscribe(emitter) {
  emitter.on('event', handler);
  return () => emitter.off('event', handler);
}
```

## Lazy Loading

Load modules only when needed:

```javascript
let heavyModule = null;

async function getHeavyModule() {
  if (!heavyModule) {
    const { HeavyModule } = await import('./heavy-module.js');
    heavyModule = new HeavyModule();
  }
  return heavyModule;
}
```

## Related

- [caching.md](./caching.md) - Caching patterns and libraries
- [profiling.md](./profiling.md) - Profiling and benchmarking tools
