# 05. Detection

<div class="chapter-meta">
  <span class="meta-item meta-reading-time">⏱ 10 min read</span>
  <span class="meta-item meta-difficulty-intermediate">🟡 Intermediate</span>
  <span class="meta-item meta-prerequisites">📋 Prereq: Chapter 04</span>
</div>

> **Core idea:** Thrashing is detectable through a specific set of system signals. Knowing which metrics to look at — and what thresholds indicate danger — lets you catch it before it becomes a full outage.

---

## Detection by Thrashing Type

### OS / Memory Thrashing

| Tool | Command | Danger Signal |
|---|---|---|
| `vmstat` | `vmstat 1` | `wa` (iowait) > 30%, `si`+`so` > 0 sustained |
| `top` | `top` | CPU `%wa` column consistently high, low `%us` |
| `iostat` | `iostat -x 1` | `%util` near 100%, `await` > 100ms |
| `free` | `free -h` | Swap `used` growing steadily |
| `sar` | `sar -B 1` | `pgscand/s` (pages scanned) very high |

**Reading vmstat:**
```bash
vmstat 1 5

procs ----memory---- ---swap-- ---io--- --cpu--
 r  b  free   swap   si   so   bi   bo  us sy id wa
 0  0  8192   0       0    0    0    0  10  5 85  0  # Healthy
 0  5   512  45000  2800 2900 9500 9200   2  3  1 94  # THRASHING
```

Key: `si` (swap in) and `so` (swap out) should be **0** on a healthy system. Any sustained non-zero value is a warning.

---

### Database Buffer Thrashing

**PostgreSQL:**
```sql
-- Buffer hit rate per table
SELECT
  relname AS table,
  heap_blks_hit AS buffer_hits,
  heap_blks_read AS disk_reads,
  ROUND(100.0 * heap_blks_hit /
    NULLIF(heap_blks_hit + heap_blks_read, 0), 2) AS hit_rate_pct
FROM pg_statio_user_tables
ORDER BY disk_reads DESC
LIMIT 10;
```

| table | buffer_hits | disk_reads | hit_rate_pct |
|---|---|---|---|
| orders | 50,000 | 45,000 | 52.6% — THRASHING |
| users | 980,000 | 2,000 | 99.8% — Healthy |

**MySQL / InnoDB:**
```sql
SHOW ENGINE INNODB STATUS;
-- Look for: "Buffer pool hit rate 946/1000"
-- Means 94.6% hit rate — borderline acceptable

SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool%';
-- Innodb_buffer_pool_reads: high → disk reads are happening
-- Innodb_buffer_pool_read_requests: total requests
```

---

### Cache (Redis) Thrashing

```bash
redis-cli INFO stats | grep -E "keyspace_(hits|misses)|evicted"

keyspace_hits:500000
keyspace_misses:450000
evicted_keys:200000

# Hit rate = 500000 / (500000 + 450000) = 52.6% → CACHE THRASHING
```

**Redis latency:**
```bash
redis-cli --latency-history -i 5
# Healthy: < 1ms
# Warning: > 5ms
# Critical: > 20ms
```

**Redis memory:**
```bash
redis-cli INFO memory | grep -E "used_memory_human|maxmemory_human|mem_fragmentation"
used_memory_human: 450.00M
maxmemory_human: 512.00M   # Almost full → imminent eviction surge
mem_fragmentation_ratio: 2.5  # > 1.5 indicates memory fragmentation
```

---

## Thrashing Severity Scale

| Severity | Memory Thrashing | DB Thrashing | Cache Thrashing |
|---|---|---|---|
| **Healthy** | iowait < 5%, swap = 0 | Buffer hit > 99% | Cache hit > 95% |
| **Warning** | iowait 5–30%, swap growing | Buffer hit 95–99% | Cache hit 80–95% |
| **Critical** | iowait 30–70%, swap active | Buffer hit 90–95% | Cache hit 50–80% |
| **Thrashing** | iowait > 70%, swap loop | Buffer hit < 90% | Cache hit < 50% |

---

## Grafana / Monitoring Alerts to Set

### OS Level
```yaml
# Prometheus + Alertmanager rules
- alert: HighIOWait
  expr: node_cpu_seconds_total{mode="iowait"} > 0.30
  for: 5m
  labels:
    severity: critical

- alert: SwapInUse
  expr: node_memory_SwapTotal_bytes - node_memory_SwapFree_bytes > 1e9
  for: 2m
  labels:
    severity: warning
```

### Redis
```yaml
- alert: RedisCacheHitRateLow
  expr: redis_keyspace_hits_total / (redis_keyspace_hits_total + redis_keyspace_misses_total) < 0.90
  for: 5m

- alert: RedisEvictionsHigh
  expr: rate(redis_evicted_keys_total[5m]) > 100
  for: 2m
```

### PostgreSQL
```yaml
- alert: PostgresBufferHitRateLow
  expr: pg_stat_bgwriter_buffers_backend / (pg_stat_bgwriter_buffers_backend + pg_stat_bgwriter_buffers_clean) < 0.95
  for: 5m
```

---

## Quick Diagnosis Checklist

When a system is slow or unresponsive, run this checklist:

```
1. Check CPU: top / htop
   □ Is iowait > 30%?  → Disk I/O problem → memory thrashing

2. Check Memory: free -h / vmstat
   □ Is swap used > 0 and growing?  → Memory thrashing
   □ Is si/so > 0 sustained?         → Active swap loop

3. Check Disk: iostat -x 1
   □ Is %util near 100%?            → Disk saturated
   □ Is await > 100ms?              → Requests queuing on disk

4. Check Redis: redis-cli INFO stats
   □ Is hit rate < 90%?             → Cache thrashing
   □ Are evicted_keys growing fast? → Cache under pressure

5. Check Database: pg_statio / INNODB STATUS
   □ Is buffer hit rate < 99%?      → DB buffer thrashing
   □ Any slow queries > 1 second?   → Index or buffer issue
```

---

## Key Terms

<div class="key-terms">
<h4>📖 Glossary</h4>
<dl>
  <dt>iowait</dt>
  <dd>CPU time spent waiting for disk I/O. High iowait is the primary OS thrashing signal.</dd>
  <dt>Buffer Hit Rate</dt>
  <dd>Fraction of DB page requests served from memory (not disk). Target: > 99%.</dd>
  <dt>Cache Hit Rate</dt>
  <dd>Fraction of cache requests served from the cache (not DB). Target: > 90%.</dd>
  <dt>Evicted Keys</dt>
  <dd>Cache keys removed to make room for new ones. High eviction rate signals cache pressure.</dd>
</dl>
</div>

---

## Quick Revision

<div class="revision-box">

**30-second answer:**

> "To detect memory thrashing: check vmstat for si/so > 0 and iowait > 30%. To detect database thrashing: query pg_statio_user_tables and look for buffer hit rate < 99%. To detect cache thrashing: check Redis INFO stats for keyspace_misses/(hits+misses) > 10% or rapid evicted_keys growth. Set Prometheus alerts on these thresholds to catch it before it becomes an outage."

</div>

---

*Prev → [04. Cache Thrashing](./04-cache-thrashing.md) · Next → [06. Prevention](./06-prevention.md)*
