---
name: caching
description: Caching patterns and libraries
metadata:
  tags: caching, memoization, performance, async-cache-dedupe
---

# Caching in Node.js

## Memoization with mnemoist

Use [mnemoist](https://github.com/Yomguithereal/mnemonist) for synchronous memoization:

```javascript
const { LRUCache } = require('mnemonist');

const cache = new LRUCache(1000);

function getUser(id) {
  if (cache.has(id)) {
    return cache.get(id);
  }
  const user = fetchUserSync(id);
  cache.set(id, user);
  return user;
}
```

## Async Caching with async-cache-dedupe

Use [async-cache-dedupe](https://github.com/mcollina/async-cache-dedupe) for async operations with request deduplication:

```javascript
const { createCache } = require('async-cache-dedupe');

const cache = createCache({
  ttl: 60, // seconds
  stale: 5, // serve stale while revalidating
  storage: { type: 'memory' },
});

cache.define('getUser', async (id) => {
  return await db.users.findById(id);
});

cache.define('getPost', {
  ttl: 300,
  stale: 30,
}, async (id) => {
  return await db.posts.findById(id);
});

// Usage - concurrent calls are deduplicated
const user = await cache.getUser('123');
const post = await cache.getPost('456');
```

### Request Deduplication

async-cache-dedupe automatically deduplicates concurrent requests:

```javascript
// These three concurrent calls result in only ONE database query
const [user1, user2, user3] = await Promise.all([
  cache.getUser('123'),
  cache.getUser('123'),
  cache.getUser('123'),
]);
```

### Redis Storage

For distributed caching across multiple instances:

```javascript
const { createCache } = require('async-cache-dedupe');
const Redis = require('ioredis');

const redis = new Redis();

const cache = createCache({
  ttl: 60,
  storage: {
    type: 'redis',
    options: { client: redis },
  },
});
```

## LRU Cache

Use [lru-cache](https://github.com/isaacs/node-lru-cache) for bounded in-memory caching:

```javascript
const { LRUCache } = require('lru-cache');

const cache = new LRUCache({
  max: 500,           // Maximum items
  ttl: 1000 * 60 * 5, // 5 minutes
  updateAgeOnGet: true,
});

cache.set('user:123', user);
const cached = cache.get('user:123');
```

## Cache Invalidation Patterns

### Time-Based Expiration

```javascript
const cache = createCache({
  ttl: 60,    // Fresh for 60 seconds
  stale: 30,  // Serve stale for 30 more seconds while revalidating
});
```

### Manual Invalidation

```javascript
// Invalidate single entry
await cache.invalidate('getUser', '123');

// Invalidate all entries for a function
await cache.clear('getUser');

// Clear entire cache
await cache.clear();
```

### Reference-Based Invalidation

```javascript
const cache = createCache({
  ttl: 60,
  storage: { type: 'memory' },
});

cache.define('getUser', {
  references: (args, key, result) => [`user:${result.id}`],
}, async (id) => {
  return await db.users.findById(id);
});

cache.define('getUserPosts', {
  references: (args, key, result) => [`user:${args[0]}`],
}, async (userId) => {
  return await db.posts.findByUserId(userId);
});

// Invalidate all cache entries referencing this user
await cache.invalidateAll(`user:123`);
```

## When to Cache

- Database query results
- External API responses
- Computed values that are expensive to calculate
- Configuration that rarely changes

## When NOT to Cache

- User-specific sensitive data (without proper isolation)
- Rapidly changing data
- Data that must always be consistent
- Large objects that would exhaust memory
