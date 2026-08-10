# 02. Types of Load Balancers

<div class="chapter-meta">
  <span class="meta-item meta-reading-time">⏱ 10 min read</span>
  <span class="meta-item meta-difficulty-intermediate">🟡 Intermediate</span>
  <span class="meta-item meta-prerequisites">📋 Prereq: Chapter 01</span>
</div>

> **Core idea:** Load balancers operate at different OSI layers. L4 works at the TCP level; L7 works at the HTTP level. L7 is smarter but costlier. Choosing the right one depends on what you need to inspect.

---

## L4 vs L7 — The Big Picture

| | L4 (Transport Layer) | L7 (Application Layer) |
|---|---|---|
| **OSI Layer** | Layer 4 — TCP/UDP | Layer 7 — HTTP/HTTPS |
| **Sees** | IP address + port | Full HTTP request (URL, headers, cookies, body) |
| **Speed** | Faster (less processing) | Slower (more inspection) |
| **Routing based on** | Source/destination IP, port | URL path, host header, cookie, method |
| **SSL** | Passes through (no decryption) | Terminates SSL (decrypts) |
| **Examples** | AWS NLB, HAProxy (TCP mode) | AWS ALB, NGINX, Envoy, Traefik |

---

## L4 Load Balancer — How It Works

L4 operates purely on TCP/UDP packets. It sees:
- Source IP → Destination IP
- Port number

It does **not** open the packet and look at the HTTP content.

```
Client → [TCP Packet: SRC=client, DST=LB:443]
            ↓  (rewrite destination IP only)
LB     → [TCP Packet: SRC=client, DST=Server2:8080]
```

**Use case:** Massive throughput (millions of connections/sec), non-HTTP protocols (gRPC, WebSockets, raw TCP, SMTP, DNS).

---

## L7 Load Balancer — How It Works

L7 terminates the TCP connection, reads the full HTTP request, makes a routing decision, then opens a **new** TCP connection to the backend.

```
Client → HTTPS → L7 LB (decrypts SSL) → inspects URL/headers → routes to backend
```

### Routing Examples

| Rule | Action |
|---|---|
| `GET /api/users/*` | → Route to Users Service |
| `GET /api/products/*` | → Route to Products Service |
| `Cookie: session=A1B2C3` | → Sticky to Server 3 |
| `Host: admin.example.com` | → Route to Admin Service |
| `Content-Type: application/grpc` | → Route to gRPC backend |

---

## Stateless vs Stateful APIs

Understanding stateless vs. stateful is essential to load balancing decisions.

### Stateless API

The server does **not** store any client session data. Every request carries everything the server needs (JWT token, payload, etc.).

```
Request 1 → Server A  (reads JWT, processes)
Request 2 → Server B  (reads JWT, processes)   ← Works! No state needed.
Request 3 → Server C  (reads JWT, processes)
```

- **Advantage:** Any request can go to any server — perfect for load balancing
- **Examples:** REST APIs with JWT, GraphQL

### Stateful API

The server **stores** session data in memory. Requests must return to the **same server**.

```
Request 1 → Server A  (session created in Server A's memory)
Request 2 → Server B  (no session found → error!)
```

Stateful apps require **sticky sessions** (session affinity) — the LB must route the same client to the same server.

- **Examples:** Legacy PHP apps, WebSocket connections, FTP sessions

---

## Sticky Sessions (Session Affinity)

When using L7, the LB can set a cookie to remember which backend served a client:

```
Response header: Set-Cookie: SERVERID=server-2; Path=/
```

Every subsequent request from that client includes the cookie, so the LB always routes to `server-2`.

!!! warning "Sticky Session Risk"
    If `server-2` goes down, all its stickied users lose their session. Prefer stateless designs wherever possible.

---

## Hardware vs Software Load Balancers

| | Hardware LB | Software LB |
|---|---|---|
| **Examples** | F5 BIG-IP, Citrix ADC | NGINX, HAProxy, Envoy, Traefik |
| **Cost** | Very expensive ($50k–$500k) | Free / cheap |
| **Throughput** | Extremely high | Very high |
| **Flexibility** | Low (vendor locked) | High (fully configurable) |
| **Used by** | Banks, telecoms | Most modern cloud apps |

---

## Global Load Balancing (GeoDNS)

For multi-region systems, DNS-based global load balancers route users to the **nearest data center**:

```
User in Mumbai   → DNS → Mumbai region LB → Mumbai servers
User in New York → DNS → US-East region LB → US servers
User in Tokyo    → DNS → Asia-Pacific LB   → Tokyo servers
```

**Examples:** AWS Route53 latency routing, Cloudflare Load Balancing, Azure Traffic Manager.

---

## Key Terms

<div class="key-terms">
<h4>📖 Glossary</h4>
<dl>
  <dt>L4 Load Balancer</dt>
  <dd>Routes traffic based on IP and port. Does not inspect HTTP content. Fast.</dd>
  <dt>L7 Load Balancer</dt>
  <dd>Routes traffic based on HTTP headers, URL paths, cookies. Terminates SSL.</dd>
  <dt>Sticky Session</dt>
  <dd>Routes a specific client to the same backend server on every request.</dd>
  <dt>SSL Termination</dt>
  <dd>Decrypting SSL/TLS at the load balancer layer.</dd>
  <dt>Stateless API</dt>
  <dd>An API where the server holds no session state. Any server can handle any request.</dd>
</dl>
</div>

---

## Quick Revision

<div class="revision-box">

**Interview answer — L4 vs L7:**

> "L4 load balancers route at the TCP/UDP level using IP and port. They're fast but can't route by URL or headers. L7 load balancers operate at the HTTP level — they can route by URL path, host header, cookie, or request method. L7 also terminates SSL. For microservices with path-based routing, you always want L7. For raw high-throughput TCP traffic, L4 is better."

</div>

---

*Prev → [01. Introduction](./01-introduction.md) · Next → [03. Algorithms](./03-algorithms.md)*
