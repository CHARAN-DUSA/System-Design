# 01. Introduction to Load Balancing

<div class="chapter-meta">
  <span class="meta-item meta-reading-time">⏱ 8 min read</span>
  <span class="meta-item meta-difficulty-beginner">🟢 Beginner</span>
</div>

> **Core idea:** A load balancer distributes incoming traffic across multiple servers so that no single server becomes a bottleneck. It is the traffic cop of distributed systems.

---

## What is Load Balancing?

Load balancing is the process of distributing incoming network requests across a pool of backend servers to ensure:

- **High availability** — if one server fails, traffic is routed to healthy ones
- **Scalability** — add more servers to handle more traffic
- **Low latency** — requests are sent to servers with available capacity
- **No single point of failure** — no individual server can bring down the system

---

## The Restaurant Analogy 🍽️

Imagine a restaurant with one chef:

```
100 Customers → One Chef → Overloaded → Slow, errors, crash
```

The owner hires 5 chefs and adds a **manager (host)** at the front:

```
100 Customers → Manager (Load Balancer) → Chef 1
                                        → Chef 2
                                        → Chef 3
                                        → Chef 4
                                        → Chef 5
```

The manager decides **which chef** handles each customer based on who is free. That manager is your **load balancer**.

---

## What Does a Load Balancer Actually Do?

| Function | Description |
|---|---|
| **Traffic distribution** | Splits requests across multiple servers |
| **Health checking** | Pings servers to detect failures |
| **Session management** | Optionally sticks users to the same server |
| **SSL termination** | Decrypts HTTPS at the LB so backends serve plain HTTP |
| **Rate limiting** | Blocks clients sending too many requests |
| **Caching** | Can cache static responses at the edge |

---

## Where Does a Load Balancer Sit?

```
Internet → DNS → CDN → Load Balancer
                              |
           ┌──────────────────┼──────────────────┐
           ▼                  ▼                  ▼
        Server A           Server B           Server C
       (App logic)        (App logic)        (App logic)
           └──────────────────┼──────────────────┘
                              ▼
                        Shared Database
```

---

## Why Not Just Use DNS?

DNS round-robin can distribute load — but it:

- Has **no health checking** (points to dead servers)
- Has **slow propagation** (TTL delays)
- Has **no sticky sessions**
- Has **no SSL termination**

A proper load balancer solves all of these.

---

## Key Terms

<div class="key-terms">
<h4>📖 Glossary</h4>
<dl>
  <dt>Load Balancer</dt>
  <dd>A server or service that distributes incoming traffic across multiple backends.</dd>
  <dt>Health Check</dt>
  <dd>Periodic ping to each backend to verify it is alive and accepting requests.</dd>
  <dt>SSL Termination</dt>
  <dd>Decrypting HTTPS at the load balancer so backend servers receive plain HTTP.</dd>
  <dt>Upstream</dt>
  <dd>The pool of backend servers that a load balancer forwards traffic to.</dd>
</dl>
</div>

---

## Quick Revision

<div class="revision-box">

**30-second answer for interviews:**

> "A load balancer sits between the client and the server pool. It distributes incoming requests using an algorithm (round-robin, least connections, etc.), performs health checks to avoid sending traffic to dead servers, handles SSL termination, and enables horizontal scaling without the client knowing how many servers exist behind it."

</div>

---

*Next → [02. Types of Load Balancers](./02-types.md)*
