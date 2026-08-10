# 04. Cache Thrashing

<div class="chapter-meta">
  <span class="meta-item meta-reading-time">⏱ 13 min read</span>
  <span class="meta-item meta-difficulty-advanced">🔴 Advanced</span>
  <span class="meta-item meta-prerequisites">📋 Prereq: Chapter 03</span>
</div>

> **Core idea:** Cache thrashing happens when the working set of keys that your application needs is larger than the cache can hold, causing a cycle of evictions and misses that defeats the purpose of caching and floods the backend database.

---

## What is Cache Thrashing?

A cache (Redis, Memcached) works by storing recently accessed data in fast memory. Cache thrashing occurs when:

1. The application needs more keys than the cache can hold
2. Newly written keys constantly evict recently used keys
3. The evicted keys are needed again immediately
4. Every access is a cache miss → every request hits the database

```
Working set needed: 100,000 keys (each 10KB) = ~1GB
Redis maxmemory:    512MB

Redis evicts older keys to make room for new ones.
Those evicted keys are needed again immediately.
→ Cache hit rate drops to 10-20%
→ 80-90% of requests hit the DB
→ DB overloads → slow queries → cascading failure
```

---

## LRU Cache Pollution

The most common form of cache thrashing is **LRU pollution**.

### Scenario: One Batch Job Pollutes the Cache

```
Application pattern:
  - 10,000 hot user sessions — accessed constantly
  - 1 nightly report — reads 500,000 different keys once each

Timeline:
  23:00 → Report starts
  23:01 → Report reads key_1, key_2, ... key_500000
           → Each read evicts a hot session key from LRU cache
  23:05 → Report finishes
  23:05 → All 10,000 hot session keys have been evicted from cache!
  23:05 → Normal traffic resumes → every session request = cache miss
           → All 10,000 session lookups hit the DB simultaneously
           → CACHE STAMPEDE + DB OVERLOAD
```

**This is a real production outage pattern.**

---

## Cache Key Eviction Loops

A more subtle form of cache thrashing:

```
Cache has 3 slots: [A, B, C]

Access sequence:
  Read A → [A, B, C]  hit
  Read D → [D, A, B]  miss, evict C
  Read E → [E, D, A]  miss, evict B
  Read C → [C, E, D]  miss, evict A  ← A just got evicted!
  Read A → [A, C, E]  miss, evict D
  Read D → [D, A, C]  miss, evict E
  Read B → [B, D, A]  miss, evict C
  Read C → [C, B, D]  miss, evict A  ← cycle repeats
```

If the access pattern cycles through more keys than the cache holds, **every access is a miss**. This is the worst case — hit rate = 0%.

---

## Redis Eviction Policies

Redis uses the `maxmemory-policy` setting to choose what to evict when memory is full:

| Policy | Behavior | Best For |
|---|---|---|
| `noeviction` | Returns error when full | Strict data integrity |
| `allkeys-lru` | Evicts LRU key from all keys | General caching |
| `volatile-lru` | Evicts LRU key with TTL set | Mixed cache + persistent data |
| `allkeys-lfu` | Evicts least frequently used key | Frequency-based hot/cold data |
| `volatile-ttl` | Evicts key with shortest TTL | TTL-managed caches |
| `allkeys-random` | Evicts random key | Rarely useful |

**For most caches:** Use `allkeys-lru` or `allkeys-lfu`.

**LFU (Least Frequently Used)** is better than LRU for preventing batch job cache pollution — because a key accessed 10,000 times won't be evicted by a batch key accessed once.

---

## High Read/Write Ratio Thrashing

When writes are as frequent as reads, the cache has no stable hot set:

```
Write-heavy app (every request writes a new unique key):
  write("session:user_1001", data)
  write("session:user_1002", data)
  write("session:user_1003", data)
  ...

Redis fills up → evicts old session keys
User 1001 comes back → session evicted → cache miss → DB hit
User 1002 comes back → session evicted → cache miss → DB hit
```

**Fix:** If your read/write ratio is < 3:1, a traditional LRU cache may not be effective. Consider TTL-based expiry, write-through caching, or a purpose-built session store.

---

## Cache Stampede (Thundering Herd)

When a popular cache key expires, all concurrent requests for that key miss simultaneously and all hit the database at once:

```
Key "trending_posts" TTL expires at 12:00:00.000

12:00:00.001 → Request 1: cache miss → DB query started
12:00:00.002 → Request 2: cache miss → DB query started
12:00:00.003 → Request 3: cache miss → DB query started
...
12:00:00.100 → 200 requests all running the same DB query simultaneously
               → DB CPU spikes to 100%
               → All 200 queries return same result
               → 199 of them wasted
```

### Prevention Strategies

| Strategy | How It Works |
|---|---|
| **Mutex / Lock** | Only one process fetches from DB; others wait for cache to fill |
| **Probabilistic early expiry** | Start refreshing before TTL expires using probability |
| **Staggered TTL** | `TTL = base_ttl + random(0, 30s)` to prevent synchronized expiry |
| **Background refresh** | Async worker refreshes cache before TTL expires |

---

## Detecting Cache Thrashing in Redis

```bash
# Connect to Redis
redis-cli

# Check hit/miss ratio
INFO stats

# Key metrics:
keyspace_hits:     4500
keyspace_misses:  95500
# Hit rate = 4500 / (4500 + 95500) = 4.5% → THRASHING!

# Check memory and evictions
INFO memory
INFO stats | grep evicted_keys
evicted_keys: 2400000   # 2.4 million keys evicted → cache under pressure
```

**Healthy hit rate:** > 90% (ideally > 95%)
**Thrashing:** < 50% hit rate

---

## Key Terms

<div class="key-terms">
<h4>📖 Glossary</h4>
<dl>
  <dt>Cache Hit Rate</dt>
  <dd>Percentage of requests served from cache vs. total requests. Higher is better.</dd>
  <dt>LRU (Least Recently Used)</dt>
  <dd>An eviction policy that removes the key not accessed for the longest time.</dd>
  <dt>LFU (Least Frequently Used)</dt>
  <dd>An eviction policy that removes the key accessed the fewest times overall.</dd>
  <dt>Cache Stampede</dt>
  <dd>When a popular cache key expires and all concurrent requests simultaneously miss and hit the DB.</dd>
  <dt>Working Set</dt>
  <dd>The set of cache keys your application actively needs at any given time.</dd>
</dl>
</div>

---

## Quick Revision

<div class="revision-box">

**30-second interview answer:**

> "Cache thrashing occurs when the working set of keys exceeds the cache capacity, causing constant evictions and misses. Common causes are LRU pollution from batch scans, key eviction loops, and cache stampedes on key expiry. Fix by increasing cache size, using LFU instead of LRU to resist pollution, adding a mutex to prevent stampedes, and using staggered TTLs to prevent synchronized expiry."

</div>

---

*Prev → [03. Database Thrashing](./03-database-thrashing.md) · Next → [05. Detection](./05-detection.md)*
