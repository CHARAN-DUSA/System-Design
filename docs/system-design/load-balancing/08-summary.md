# 08. Summary — Load Balancing

<div class="chapter-meta">
  <span class="meta-item meta-reading-time">⏱ 5 min read</span>
  <span class="meta-item meta-difficulty-beginner">🟢 All Levels</span>
</div>

> This is your final revision page before an interview. Read this once, then you are ready.

---

## The Complete Mental Model

```
Client Request
     │
     ▼
Global DNS LB (GeoDNS → nearest region)
     │
     ▼
L4 Load Balancer (TCP passthrough, high throughput)
     │
     ▼
L7 Load Balancer (URL routing, SSL termination, sticky sessions)
     │
     ├─────────────────┬─────────────────┐
     ▼                 ▼                 ▼
App Server A      App Server B      App Server C
(Stateless)       (Stateless)       (Stateless)
     │                 │                 │
     └────────────┬────┘                 │
                  ▼                      │
            Redis Cache ◄────────────────┘
                  │ (miss)
                  ▼
            PgBouncer (Connection Pool)
                  │
                  ▼
            Primary DB + Read Replicas
```

---

## Key Decisions Cheat Sheet

| Decision | Choose | When |
|---|---|---|
| L4 vs L7 | L4 | High throughput, non-HTTP, TLS passthrough |
| L4 vs L7 | L7 | Path routing, host routing, microservices |
| Algorithm | Round Robin | Equal servers, equal request cost |
| Algorithm | Least Connections | Variable request duration |
| Algorithm | IP Hash | Need stickiness, no cookie support |
| Hashing | Consistent | Distributed caches, sharded DBs |
| Hashing | Modulo | Simple apps, static server count |
| VNodes | Yes | Heterogeneous clusters, better failure recovery |
| Caching | Redis | Repeated reads, high QPS |
| Sessions | Stateless (JWT) | Prefer always — scales horizontally |

---

## Module Concepts Map

| Chapter | Core Concept |
|---|---|
| 01. Introduction | What, why, and where of load balancers |
| 02. Types | L4 vs L7, stateless vs stateful |
| 03. Algorithms | Round Robin, Least Connections, IP Hash |
| 04. Consistent Hashing | Hash ring, clockwise routing, 1/N remapping |
| 05. Virtual Nodes | Multiple ring positions, even distribution |
| 06. Production Architecture | Cache-aside, connection pooling, health checks |
| 07. Interview Questions | Q&A: all scenarios and deep answers |

---

## 5 Things to Say in Every Load Balancing Interview

1. **"I'd use an L7 LB for path-based routing across microservices."**
2. **"Health checks every 5–10 seconds ensure failed servers are removed from rotation."**
3. **"Stateless APIs with JWT allow any server to handle any request — no sticky sessions needed."**
4. **"Redis with cache-aside reduces DB load from 100K req/sec to ~5K req/sec (95% cache hit rate)."**
5. **"Consistent hashing + virtual nodes ensures minimal key remapping when the cluster changes."**

---

## Common Mistakes to Avoid

<div class="production-notes">
<h4>⚠️ Interview Pitfalls</h4>

- Saying "just add more servers" without mentioning the DB bottleneck
- Forgetting health checks — no LB works without them
- Confusing L4 and L7 capabilities (L4 cannot read HTTP headers)
- Not mentioning cache invalidation strategy (TTL? Event-driven?)
- Using modulo hashing for distributed caches (causes stampedes on server change)

</div>

---

## Revision Checklist

- [ ] I can explain what a load balancer does in 30 seconds
- [ ] I know the difference between L4 and L7
- [ ] I can name 4 load balancing algorithms and their use cases
- [ ] I can draw the hash ring and explain clockwise routing
- [ ] I can explain why virtual nodes improve consistent hashing
- [ ] I can describe the cache-aside pattern step by step
- [ ] I can draw the full production architecture from memory

---

*You finished the Load Balancing module! → Next module: [Thrashing](../thrashing/01-introduction.md)*
