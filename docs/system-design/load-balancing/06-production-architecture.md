# 06. Production Architecture

<div class="chapter-meta">
  <span class="meta-item meta-reading-time">⏱ 15 min read</span>
  <span class="meta-item meta-difficulty-advanced">🔴 Advanced</span>
  <span class="meta-item meta-prerequisites">📋 Prereq: Chapter 05</span>
</div>

> **Core idea:** Real systems layer multiple load balancing tiers, combine them with caching, connection pooling, and health checking to build an architecture that handles millions of requests per second.

---

## The Database Bottleneck Problem

As traffic scales, the backend database becomes the slowest component:

```
10,000 requests/sec → App Servers (scale easily) → Single DB → OVERLOADED
```

Databases are expensive to scale horizontally. Every API request hitting the DB directly is wasteful.

**Solution stack:**
1. Load balancer (distribute across app servers)
2. Cache layer (Redis / Memcached) — serve repeated reads from memory
3. Connection pooler (PgBouncer) — reuse DB connections
4. Read replicas — distribute read queries
5. Sharding — partition data across multiple DB instances

---

## Redis Cache-Aside Pattern

The most common caching pattern. The application checks the cache first:

```
Application logic:

  result = cache.get(key)          # 1. Check Redis
  if result is None:               # 2. Cache miss
      result = db.query(sql)       # 3. Hit the DB
      cache.set(key, result, ttl)  # 4. Store in Redis
  return result                    # 5. Return to client
```

```
Timeline:
  Request 1: Cache MISS → DB hit → store in cache (slow)
  Request 2: Cache HIT  → Redis  (fast, ~0.5ms)
  Request 3: Cache HIT  → Redis  (fast)
  ...
  Request N: Cache MISS → TTL expired → DB hit → refresh cache
```

---

## Multi-Tier Load Balancing Architecture

A production system at scale uses multiple layers:

```
┌─────────────────────────────────────────────────────────┐
│                    INTERNET                             │
└──────────────────────────┬──────────────────────────────┘
                           │
              ┌────────────▼────────────┐
              │     Global LB (DNS)     │  GeoDNS / Route 53
              │  Routes by region       │
              └────────────┬────────────┘
                           │
              ┌────────────▼────────────┐
              │       CDN Edge          │  Static assets
              │  (CloudFront / Fastly)  │
              └────────────┬────────────┘
                           │
              ┌────────────▼────────────┐
              │    L4 Load Balancer     │  AWS NLB / HAProxy
              │  (TLS passthrough)      │
              └────────────┬────────────┘
                           │
              ┌────────────▼────────────┐
              │    L7 Load Balancer     │  AWS ALB / NGINX
              │  (URL-based routing)    │
              └──┬──────────────────┬───┘
                 │                  │
     ┌───────────▼───┐     ┌───────▼───────┐
     │  API Servers  │     │  API Servers  │
     │  (Stateless)  │     │  (Stateless)  │
     └───────┬───────┘     └──────┬────────┘
             │                   │
     ┌───────▼───────────────────▼────────┐
     │         Redis Cache Cluster        │
     │    (Cache-aside, TTL-based)        │
     └───────────────┬────────────────────┘
                     │ Cache miss
     ┌───────────────▼────────────────────┐
     │         Connection Pooler          │
     │   (PgBouncer / ProxySQL)           │
     └───────────────┬────────────────────┘
                     │
     ┌───────────────▼────────────────────┐
     │         Primary Database           │
     │   + Read Replicas (scaling reads)  │
     └────────────────────────────────────┘
```

---

## Health Checks in Production

Every layer performs health checks:

| Layer | Health Check Type | Interval |
|---|---|---|
| L4 LB | TCP connect (can I open a socket?) | 5 sec |
| L7 LB | HTTP GET /health → expects 200 OK | 10 sec |
| Redis | PING command | 30 sec |
| DB Pooler | Test connection + query | 60 sec |

When a health check fails N times (e.g., 3), the node is removed from rotation. It is re-added after passing M consecutive checks.

---

## Connection Pooling — Why It Matters

Each DB connection consumes:
- ~5–10MB memory on PostgreSQL
- OS file descriptor
- CPU for TLS handshake

Without pooling:
```
10,000 app threads × 1 connection each = 10,000 DB connections = crash
```

With PgBouncer (connection pooler):
```
10,000 app threads → PgBouncer pool (50 connections) → DB
```

PgBouncer queues requests and reuses the same 50 connections.

---

## MediSphere Case Study: Scaling from 1K to 10M Users

| Stage | Users | Architecture Change |
|---|---|---|
| Startup | 1,000 | 1 app server + 1 DB |
| Growth | 10,000 | Add load balancer + 3 app servers |
| Scale | 100,000 | Add Redis cache, read replicas |
| Hyper-scale | 1,000,000 | Add CDN, PgBouncer, DB sharding |
| Production | 10,000,000 | Multi-region, consistent hashing, VNodes |

**The key insight:** Each scaling step targets the bottleneck of the current stage.

---

## Key Terms

<div class="key-terms">
<h4>📖 Glossary</h4>
<dl>
  <dt>Cache-Aside</dt>
  <dd>A caching pattern where the application checks the cache before hitting the DB and populates the cache on a miss.</dd>
  <dt>TTL (Time to Live)</dt>
  <dd>The duration a cached entry remains valid before being evicted.</dd>
  <dt>Connection Pooling</dt>
  <dd>Maintaining a pool of reusable DB connections to avoid the overhead of creating a new connection per request.</dd>
  <dt>Read Replica</dt>
  <dd>A copy of the primary database that handles read queries, reducing load on the primary.</dd>
</dl>
</div>

---

## Quick Revision

<div class="revision-box">

**30-second interview answer:**

> "In production, a typical high-scale system layers a global DNS-based load balancer, an L4 load balancer for TLS, an L7 load balancer for routing, stateless app servers, a Redis cache cluster using cache-aside, a connection pooler like PgBouncer, and a database with read replicas. Each layer solves a specific scaling problem. The key is to push as many reads as possible to Redis to avoid DB bottlenecks."

</div>

---

*Prev → [05. Virtual Nodes](./05-virtual-nodes.md) · Next → [07. Interview Questions](./07-interview-questions.md)*
