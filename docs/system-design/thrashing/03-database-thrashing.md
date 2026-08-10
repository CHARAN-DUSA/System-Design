# 03. Database Thrashing

<div class="chapter-meta">
  <span class="meta-item meta-reading-time">⏱ 14 min read</span>
  <span class="meta-item meta-difficulty-advanced">🔴 Advanced</span>
  <span class="meta-item meta-prerequisites">📋 Prereq: Chapter 02</span>
</div>

> **Core idea:** Databases maintain an in-memory **buffer pool** (like a RAM cache for disk pages). When queries access more data than fits in the buffer pool, the database constantly swaps pages in and out — this is database-level thrashing, and it kills query performance.

---

## The Database Buffer Pool

A relational database never reads directly from disk on every query. It maintains a **buffer pool** (also called shared buffers) — an in-memory cache of disk pages:

```
Buffer Pool (in RAM):
  ┌────────┬────────┬────────┬────────┬────────┐
  │ Page 1 │ Page 4 │ Page 9 │ Page 2 │ Page 7 │
  └────────┴────────┴────────┴────────┴────────┘
               ↑
        Hot pages — frequently accessed rows

Disk (Data Files):
  Page 1, 2, 3, 4, 5, 6, 7, 8, 9, ... 10,000 pages
```

When a query needs **Page 5**, the DB checks:
1. Is Page 5 in the buffer pool? → **Buffer Hit** (microseconds)
2. Not in buffer pool → **Buffer Miss** → read from disk → evict an old page (milliseconds)

---

## Buffer Pool Thrashing — The Scenario

Imagine a table with **10 GB of data** but you set `shared_buffers = 128MB` in PostgreSQL.

A query runs a full table scan:
```sql
SELECT * FROM orders WHERE year = 2023;
-- Scans all 10GB of data
```

The scan reads **page after page** into the buffer pool. Since 10GB >> 128MB, each new page **evicts a recently loaded page**. By the time the scan finishes the first loop, all those pages it evicted might be needed again.

```
Buffer pool: [P1, P2, P3, ... P32] (128MB = 32,768 pages of 4KB)
Query reads P1, P2, P3, ... P32, P33 → P1 evicted!
Query next needs P1 again → P33 evicted, P1 loaded!
→ Infinite eviction loop = DATABASE THRASHING
```

---

## InnoDB Buffer Pool (MySQL)

In MySQL with InnoDB:

```ini
# my.cnf
innodb_buffer_pool_size = 8G   # Recommended: 70-80% of total RAM
```

When `innodb_buffer_pool_size` is too small relative to your working set:

| Symptom | Value |
|---|---|
| `Innodb_buffer_pool_reads` | High (physical disk reads per second) |
| `Innodb_buffer_pool_read_requests` | Total buffer read requests |
| **Buffer Pool Hit Rate** | `< 99%` → problem. `< 95%` → serious thrashing |

**Formula:**
```
Hit Rate = 1 - (Innodb_buffer_pool_reads / Innodb_buffer_pool_read_requests)
```

---

## PostgreSQL Shared Buffers

PostgreSQL has two caching layers:
1. `shared_buffers` — PostgreSQL's own buffer pool
2. OS page cache — the kernel caches disk I/O too

```
postgresql.conf:
  shared_buffers = 256MB     # Too small for large datasets
```

**Recommendation:** Set `shared_buffers` to 25% of RAM. Let the OS page cache use the rest.

```bash
# Check PostgreSQL buffer hit rate:
SELECT
  sum(heap_blks_hit) as hits,
  sum(heap_blks_read) as misses,
  100 * sum(heap_blks_hit) / (sum(heap_blks_hit) + sum(heap_blks_read)) as hit_rate
FROM pg_statio_user_tables;
```

A `hit_rate < 99%` indicates frequent disk reads → potential thrashing.

---

## Read-Eviction Thrashing — The Full Loop

This is the most dangerous form of database thrashing:

```
Query A (full table scan, 2GB table):
  Read page 1 into buffer → evict oldest page
  Read page 2 into buffer → evict oldest page
  Read page 3 into buffer → evict oldest page
  ...
  (Scan complete — but all hot pages evicted)

Query B (a simple index lookup on a frequently accessed row):
  Needed page was evicted by Query A's scan!
  → Page fault → disk read → slow

Query A runs again (scheduled report):
  Evicts Query B's pages again
  ...
```

**The fix:** Use `enable_seqscan = off` for large tables in PostgreSQL, or partition the large table and add proper indexes. Never allow unlimited full table scans in production.

---

## Index-Related Buffer Thrashing

An index with poor cardinality or a missing index causes:
1. Full table scans that flood the buffer pool
2. Eviction of hot index pages
3. Next query needing those index pages → disk read

**Example:**
```sql
-- BAD: Full scan, floods buffer pool
SELECT * FROM events WHERE status = 'active';   -- 90% of rows are 'active'!

-- GOOD: Use a partial index
CREATE INDEX idx_events_active ON events (id) WHERE status = 'active';
```

---

## N+1 Query Problem and Buffer Thrashing

The N+1 problem generates thousands of tiny queries:

```python
# BAD: N+1 queries
users = db.query("SELECT * FROM users LIMIT 100")
for user in users:
    orders = db.query(f"SELECT * FROM orders WHERE user_id = {user.id}")
    # 100 extra queries → 100 random page reads → buffer churn
```

Each of the 100 queries reads a different random page. If those pages do not fit together in the buffer pool, you get constant eviction churn.

```python
# GOOD: Single JOIN
result = db.query("""
  SELECT u.*, o.* FROM users u
  JOIN orders o ON u.id = o.user_id
  LIMIT 100
""")
```

---

## Key Terms

<div class="key-terms">
<h4>📖 Glossary</h4>
<dl>
  <dt>Buffer Pool</dt>
  <dd>An in-memory cache of disk pages maintained by the database engine (InnoDB, PostgreSQL).</dd>
  <dt>Buffer Hit Rate</dt>
  <dd>The percentage of page requests served from the buffer pool vs. from disk. Target >99%.</dd>
  <dt>Full Table Scan</dt>
  <dd>Reading every page of a table because no usable index exists for the query.</dd>
  <dt>Working Set</dt>
  <dd>The set of pages a database actually needs to serve its current query workload.</dd>
  <dt>N+1 Problem</dt>
  <dd>Loading N parent records then making N separate queries for their children.</dd>
</dl>
</div>

---

## Quick Revision

<div class="revision-box">

**30-second interview answer:**

> "Database thrashing occurs when the database's buffer pool is too small to hold the working set of frequently accessed pages. Queries constantly evict each other's pages, causing disk reads on every query. Key indicators are buffer pool hit rate below 99%, high I/O wait, and slow queries despite good indexes. Fix by increasing buffer pool size, using proper indexes to avoid full scans, and eliminating N+1 query patterns."

</div>

---

*Prev → [02. Memory Pressure](./02-memory-pressure.md) · Next → [04. Cache Thrashing](./04-cache-thrashing.md)*
