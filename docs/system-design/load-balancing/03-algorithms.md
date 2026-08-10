# 03. Load Balancing Algorithms

<div class="chapter-meta">
  <span class="meta-item meta-reading-time">⏱ 12 min read</span>
  <span class="meta-item meta-difficulty-intermediate">🟡 Intermediate</span>
  <span class="meta-item meta-prerequisites">📋 Prereq: Chapter 02</span>
</div>

> **Core idea:** A load balancer's algorithm decides *which* server gets the next request. Different algorithms suit different workloads. Picking the wrong one creates hot spots and overloads.

---

## Overview of Algorithms

| Algorithm | Best For | Awareness |
|---|---|---|
| Round Robin | Homogeneous servers, equal request cost | None |
| Weighted Round Robin | Different server capacities | Server weight |
| Least Connections | Long-lived or variable requests | Active connections |
| Least Response Time | Latency-sensitive apps | Response time + connections |
| IP Hash | Sticky sessions without cookies | Client IP |
| Random | Simple, fast, stateless | None |
| Resource Based | Complex heterogeneous clusters | CPU, memory, queue depth |

---

## 1. Round Robin

Distributes requests in a rotating cycle: Server 1 → Server 2 → Server 3 → Server 1 → ...

```
Request 1  → Server A
Request 2  → Server B
Request 3  → Server C
Request 4  → Server A   ← cycles back
Request 5  → Server B
```

**Pros:** Simple, no state, works well when all servers are identical.  
**Cons:** Ignores actual server load. Server A might be slow (heavy request) while Server B and C are idle.

---

## 2. Weighted Round Robin

Each server gets a weight. Higher-weight servers receive proportionally more requests.

```
Server A: weight 3  → gets 3 out of every 5 requests
Server B: weight 1  → gets 1 out of every 5 requests
Server C: weight 1  → gets 1 out of every 5 requests
```

**Use case:** Server A has 16 cores, Server B and C have 4 cores each.

---

## 3. Least Connections

Routes the new request to the server with the **fewest active connections**.

```
Server A: 150 connections  ← skip
Server B: 80  connections  ← pick this one
Server C: 200 connections  ← skip
```

**Pros:** Great for variable request duration (DB queries, file uploads).  
**Cons:** Requires the LB to track connection counts (state).

---

## 4. Least Response Time

Routes to the server with the **lowest combination of active connections + fastest response time**.

```
Score = active_connections + avg_response_time_ms
Server A: 10 conn + 120ms = 130  ← pick
Server B: 5 conn  + 200ms = 205
Server C: 8 conn  + 180ms = 188
```

**Use case:** Latency-critical APIs where response time varies greatly between servers.

---

## 5. IP Hash

The client's IP is hashed to determine which server always handles it.

```
hash(client_ip) % num_servers = server_index
```

```
Client 10.0.0.1 → hash → always Server A
Client 10.0.0.2 → hash → always Server B
Client 10.0.0.3 → hash → always Server C
```

**Pros:** Provides sticky sessions without cookies.  
**Cons:** Uneven distribution if few clients generate most traffic. Server removal shifts many clients.

---

## 6. Random

Picks a random server from the pool for each request.

**Pros:** Extremely simple, zero state.  
**Cons:** Can create temporary hot spots. Inferior to round robin for most cases.

---

## Comparison Table

| Algorithm | State Required | Session Stickiness | Best Scenario |
|---|---|---|---|
| Round Robin | No | No | Equal servers, equal request cost |
| Weighted Round Robin | No | No | Servers with different capacity |
| Least Connections | Yes (counters) | No | Variable duration requests |
| Least Response Time | Yes (metrics) | No | Latency-sensitive workloads |
| IP Hash | No | Yes | Apps needing stickiness |
| Random | No | No | Ultra-simple, low-scale |

---

## What AWS Uses in Practice

| AWS Service | Default Algorithm |
|---|---|
| **ALB (L7)** | Round Robin (per target group) |
| **ALB + LOR** | Least Outstanding Requests (like Least Connections) |
| **NLB (L4)** | Flow Hash (IP + Port + Protocol) |
| **Route 53** | Weighted / Latency / Geolocation |

---

## The Hot Spot Problem

Round Robin assumes equal request weight. In reality:

```
Request A → 2ms (reads cached data)
Request B → 4,000ms (runs a complex DB report)
```

Server A handles Request B and is stuck for 4 seconds. Meanwhile, Server B and C get easy requests. Round Robin sends the next request to Server A — which is still busy.

**Solution:** Use **Least Connections** or **Least Response Time** for workloads with variable request duration.

---

## Key Terms

<div class="key-terms">
<h4>📖 Glossary</h4>
<dl>
  <dt>Round Robin</dt>
  <dd>Rotates requests equally across all servers in order.</dd>
  <dt>Least Connections</dt>
  <dd>Sends new requests to the server with the fewest active connections.</dd>
  <dt>IP Hash</dt>
  <dd>Deterministically maps a client's IP to a specific backend server.</dd>
  <dt>Hot Spot</dt>
  <dd>When one server receives disproportionately more load than others.</dd>
</dl>
</div>

---

## Quick Revision

<div class="revision-box">

**30-second interview answer:**

> "The most common algorithms are Round Robin, Weighted Round Robin, Least Connections, and IP Hash. Round Robin is simple and works when request cost is equal. Least Connections is better when request duration varies. IP Hash provides stickiness. In AWS, ALB uses Round Robin by default but you can enable Least Outstanding Requests for variable workloads."

</div>

---

*Prev → [02. Types](./02-types.md) · Next → [04. Consistent Hashing](./04-consistent-hashing.md)*
