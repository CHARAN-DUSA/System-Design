# 07. Interview Questions

<div class="chapter-meta">
  <span class="meta-item meta-reading-time">⏱ 20 min read</span>
  <span class="meta-item meta-difficulty-advanced">🔴 Advanced</span>
</div>

> Use this chapter to quickly revise before a system design interview. Each question has a 30-second answer and a deep answer.

---

## Core Concepts

### Q1. What is a load balancer and why do we need it?

**30-sec answer:**
> A load balancer distributes incoming requests across multiple servers. It ensures high availability (re-routes if a server fails), enables horizontal scaling (add more servers without changing clients), and reduces latency by avoiding overloaded servers.

**Deep answer:**
Without a load balancer, all traffic hits one server. A single server has CPU, memory, and network limits. A load balancer solves this by splitting traffic, doing health checks, terminating SSL, and providing a single stable entry point (IP/DNS) regardless of how many backend servers exist.

---

### Q2. What is the difference between L4 and L7 load balancing?

**30-sec answer:**
> L4 routes by IP + port (TCP layer), is very fast but cannot read HTTP content. L7 routes by URL, headers, cookies — it can do path-based routing and sticky sessions but requires SSL termination.

| | L4 | L7 |
|---|---|---|
| OSI Layer | 4 (Transport) | 7 (Application) |
| Sees | IP, Port | URL, headers, cookies |
| SSL | Passthrough | Terminates |
| Examples | AWS NLB | AWS ALB, NGINX |

---

### Q3. What is consistent hashing and why is it better than modulo hashing?

**30-sec answer:**
> Modulo hashing (`hash(key) % N`) causes ~100% key remapping when N changes. Consistent hashing places servers on a ring and remaps only ~1/N of keys when a server is added or removed — critical for distributed caches.

---

### Q4. What are virtual nodes and why do we use them?

**30-sec answer:**
> Basic consistent hashing gives each server one ring position, causing uneven load. Virtual nodes give each server multiple positions, statistically distributing load evenly. When a server fails, its load is absorbed by multiple servers instead of just one.

---

### Q5. What is the difference between stateless and stateful APIs?

**30-sec answer:**
> Stateless APIs carry all needed info in each request (JWT token) — any server can handle any request. Stateful APIs store session in server memory — requests must return to the same server (sticky sessions). Stateless is preferred for scalability.

---

## Scenario Questions

### Q6. Design a URL shortener that handles 10M requests/day

**Key components:**
1. L7 LB → routes `/shorten` and `/r/{code}` separately
2. Redis cache → cache redirect targets (high read volume)
3. Stateless API servers behind the LB
4. Primary DB + read replicas

```
User → LB → API Server → Redis cache hit → 302 Redirect
                       → Cache miss → DB → Cache set → 302 Redirect
```

---

### Q7. How does Netflix handle load balancing?

Netflix uses:
- **Eureka** (service registry) for service discovery
- **Ribbon** (client-side load balancer) — each service picks its own backend
- **Zuul / API Gateway** — edge L7 load balancing
- **Cassandra** (consistent hashing) — data distribution across regions

---

### Q8. What happens when a server behind the load balancer crashes?

1. Health check fails (HTTP 500 or TCP timeout)
2. LB marks server as unhealthy after N consecutive failures
3. LB stops routing traffic to that server
4. Remaining servers absorb the load
5. If server recovers, health checks pass again → re-added to rotation

---

### Q9. What is a cache stampede? How do you prevent it?

**Problem:** When a popular cache key expires, thousands of requests simultaneously go to the DB.

**Prevention techniques:**
1. **Mutex / lock** — only one request fetches from DB; others wait
2. **Jitter** — randomize TTL (e.g., TTL = 300 ± 30 seconds) to stagger expiry
3. **Background refresh** — async refresh before TTL expires
4. **Consistent hashing** — avoids mass key migration on server changes

---

### Q10. What is the CAP theorem and how does it relate to load balancing?

In distributed systems, you can only guarantee 2 of 3:
- **Consistency** — all nodes see the same data
- **Availability** — every request gets a response
- **Partition Tolerance** — system works despite network splits

Load balancers operate at the availability layer. When a node fails, traffic is rerouted — choosing **Availability over Consistency** (AP system). If your DB requires strong consistency, you must accept lower availability during partitions.

---

## Quick Reference Card

| Topic | Key Point |
|---|---|
| Round Robin | Simple rotation, equal servers |
| Least Connections | Best for variable-duration requests |
| IP Hash | Sticky sessions without cookies |
| L4 LB | Fast, TCP-only, no SSL termination |
| L7 LB | Smart routing, SSL termination |
| Consistent Hashing | 1/N keys remap on server change |
| Virtual Nodes | Evens out ring distribution |
| Cache-Aside | App checks cache, fetches from DB on miss |
| Health Check | TCP / HTTP check to detect server failure |
| Sticky Session | Cookie-based affinity to same server |

---

*Prev → [06. Production Architecture](./06-production-architecture.md) · Next → [08. Summary](./08-summary.md)*
