# 05. Virtual Nodes

<div class="chapter-meta">
  <span class="meta-item meta-reading-time">⏱ 12 min read</span>
  <span class="meta-item meta-difficulty-advanced">🔴 Advanced</span>
  <span class="meta-item meta-prerequisites">📋 Prereq: Chapter 04</span>
</div>

> **Core idea:** A basic hash ring gives each server one position, causing uneven load distribution. Virtual nodes (VNodes) give each physical server *multiple* positions on the ring, spreading load evenly and enabling graceful failure handling.

---

## The Problem With Basic Consistent Hashing

With only 3 servers and one position each, the ring can be highly unbalanced:

```
Ring positions:
  S1 at position 10
  S2 at position 15   ← very close to S1
  S3 at position 900  ← very far from S2
```

The arc from S2 (15) to S3 (900) is enormous. S3 ends up owning ~98% of the keyspace — a massive hot spot.

---

## The Solution: Virtual Nodes

Instead of placing each physical server once, assign each server **V virtual positions** on the ring.

```
Physical server S1 → virtual nodes: S1_v1, S1_v2, S1_v3, S1_v4, S1_v5
Physical server S2 → virtual nodes: S2_v1, S2_v2, S2_v3, S2_v4, S2_v5
Physical server S3 → virtual nodes: S3_v1, S3_v2, S3_v3, S3_v4, S3_v5
```

The ring now has 15 positions spread across the 0–2^32 space:

```
Ring: S2_v3, S1_v1, S3_v2, S1_v4, S2_v1, S3_v5, S1_v2, ...
```

A key hashes to a position, walks clockwise to the nearest virtual node (e.g., `S2_v3`), and is routed to physical server S2.

---

## Benefits of Virtual Nodes

### 1. Even Load Distribution

With V=150 virtual nodes per server, each server statistically owns ~33% of the keyspace (for 3 servers). The more virtual nodes, the smoother the distribution.

### 2. Better Failure Handling

When S1 fails, its **virtual nodes are scattered** across the ring. The keys it owned are absorbed by multiple different servers (S2 and S3) rather than dumping all of S1's load onto just S2.

```
Without VNodes (S1 fails):
  All of S1's keys → S2 only  ← S2 gets overloaded

With VNodes (S1 fails):
  S1_v1 keys → S3_v2 (absorbed by S3)
  S1_v2 keys → S2_v5 (absorbed by S2)
  S1_v3 keys → S2_v1 (absorbed by S2)
  S1_v4 keys → S3_v4 (absorbed by S3)
  S1_v5 keys → S2_v3 (absorbed by S2)
```

The load distributes across the cluster evenly.

### 3. Heterogeneous Server Support

Servers with more hardware capacity can be given more virtual nodes:

```
High-capacity server (32 cores, 128GB RAM) → 300 virtual nodes
Standard server    (8 cores, 32GB RAM)     → 100 virtual nodes
```

This is exactly how Cassandra handles heterogeneous clusters.

---

## Choosing the Right Number of Virtual Nodes

| V (VNodes per server) | Distribution Quality | Memory Overhead |
|---|---|---|
| 1 | Poor (highly uneven) | Minimal |
| 10 | Moderate | Low |
| 150 | Good | Medium |
| 1000 | Excellent | Higher |

**Cassandra default:** 256 virtual nodes per physical node.

---

## Virtual Nodes in Production Systems

| System | VNode Approach |
|---|---|
| **Apache Cassandra** | 256 VNodes per physical node (configurable) |
| **Amazon DynamoDB** | Implicit via partition splitting |
| **Redis Cluster** | 16,384 hash slots distributed across nodes |
| **Riak** | 64 VNodes per node by default |

---

## Visualization

```
Without VNodes (3 servers, 3 positions):

    0────S1(10)────S2(15)─────────────────────────S3(900)────2^32

S3 owns 885 units of the ring. S1 and S2 together own 15 units.
```

```
With VNodes (3 servers × 5 VNodes = 15 positions):

    0──S2_v3─S1_v1─S3_v2─S1_v4─S2_v1─S3_v5─S1_v2─S2_v4─S3_v1─S1_v3─S2_v2─S3_v4─S1_v5─S2_v5─S3_v3──2^32

Each server owns roughly 1/3 of the ring.
```

---

## Key Terms

<div class="key-terms">
<h4>📖 Glossary</h4>
<dl>
  <dt>Virtual Node (VNode)</dt>
  <dd>A logical token on the hash ring representing a physical server. Each physical server has multiple VNodes.</dd>
  <dt>Hot Spot</dt>
  <dd>A server or partition receiving disproportionately high load.</dd>
  <dt>Hash Slot</dt>
  <dd>Redis Cluster's term for a VNode position (16,384 total slots).</dd>
  <dt>Token</dt>
  <dd>Cassandra's term for a VNode position on the ring.</dd>
</dl>
</div>

---

## Quick Revision

<div class="revision-box">

**30-second interview answer:**

> "Virtual nodes solve the uneven load distribution problem in basic consistent hashing. Instead of one ring position per server, each server gets multiple positions (virtual nodes). This statistically distributes load evenly across all servers. When a server fails, its virtual nodes' load is absorbed by multiple servers rather than dumping everything onto one server. Cassandra uses 256 VNodes per physical node by default."

</div>

---

*Prev → [04. Consistent Hashing](./04-consistent-hashing.md) · Next → [06. Production Architecture](./06-production-architecture.md)*
