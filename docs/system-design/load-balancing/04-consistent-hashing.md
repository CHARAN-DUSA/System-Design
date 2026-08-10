# 04. Consistent Hashing

<div class="chapter-meta">
  <span class="meta-item meta-reading-time">⏱ 15 min read</span>
  <span class="meta-item meta-difficulty-advanced">🔴 Advanced</span>
  <span class="meta-item meta-prerequisites">📋 Prereq: Chapter 03</span>
</div>

> **Core idea:** Consistent hashing solves the problem of redistributing all keys when servers are added or removed. Instead of remapping everything, only the keys owned by the changed server move.

---

## The Problem With Modulo Hashing

A naive approach to routing requests to servers uses modulo:

```
server_index = hash(key) % num_servers
```

For 3 servers:

```
hash("user_123") % 3 = 1  → Server 1
hash("user_456") % 3 = 2  → Server 2
hash("user_789") % 3 = 0  → Server 0
```

This works — **until you add or remove a server.**

### Adding a 4th server:

```
hash("user_123") % 4 = 3  → Server 3  (was Server 1!)
hash("user_456") % 4 = 0  → Server 0  (was Server 2!)
hash("user_789") % 4 = 1  → Server 1  (was Server 0!)
```

**Almost every key remaps!** For a cache, this means a **cache stampede** — every request hits the database simultaneously.

---

## The Solution: The Hash Ring

Consistent hashing arranges both **servers** and **keys** on a circular ring (0 to 2^32 − 1).

```
              0 / 2^32
                 │
         ┌───────┴───────┐
    S3 ──┤               ├── S1
         │   Hash Ring   │
    S2 ──┤               │
         └───────────────┘
               2^32/2
```

### How Keys Are Routed

1. Hash the key → get a position on the ring
2. Walk **clockwise** from that position
3. The first server you hit is the one that handles the key

```
Key "user_123" hashes to position 45 on the ring.
Walking clockwise, the first server is S1 at position 90.
→ Server S1 handles "user_123".
```

---

## Why Servers and Keys Are on the Same Ring

This is the most common point of confusion:

> "Why are servers and requests on the same circle?"

**Answer:** Both servers and keys are run through the **same hash function** to produce a position on the same 0→2^32 number line. This is what makes the ring consistent — the key always goes to the nearest server in clockwise direction, regardless of how many servers exist.

The ring is **not** a physical structure. It is just a sorted data structure (a sorted array or tree) of positions. A key's server is found with a binary search for the next server position >= key's position.

---

## Adding a Server — Minimal Disruption

Add Server S4 at position 60 on the ring.

**Before:**
```
Key at position 45 → walks clockwise → hits S1 at 90
```

**After:**
```
Key at position 45 → walks clockwise → hits S4 at 60  (new!)
```

Only keys that fall between S3's position and S4's new position are remapped. Everything else stays the same.

**Typical redistribution:** `1/N` of keys move (where N is the number of servers). For 10 servers, only ~10% of keys move.

---

## Removing a Server — Same Principle

If S1 fails, its keys are absorbed by the **next server clockwise (S2)**. Only S1's keys move. All other keys are unaffected.

---

## Real-World Comparison

| Approach | Keys Remapped on Change |
|---|---|
| Modulo Hashing | ~100% (catastrophic) |
| Consistent Hashing | ~1/N (minimal) |

---

## Where Is Consistent Hashing Used?

| System | Use |
|---|---|
| **Amazon DynamoDB** | Partition key routing |
| **Apache Cassandra** | Token-based ring partitioning |
| **Redis Cluster** | Hash slot distribution (16,384 slots) |
| **Memcached** | Client-side consistent hashing |
| **CDN edge routing** | Route content to the nearest cache node |
| **Distributed caches** | Cache key routing with minimal invalidation |

---

## Key Terms

<div class="key-terms">
<h4>📖 Glossary</h4>
<dl>
  <dt>Hash Ring</dt>
  <dd>A circular space where both server and key positions are computed via the same hash function.</dd>
  <dt>Clockwise Routing</dt>
  <dd>A key is routed to the first server found by walking clockwise from the key's hash position.</dd>
  <dt>Cache Stampede</dt>
  <dd>When many keys simultaneously miss the cache and flood the backend database.</dd>
  <dt>Token</dt>
  <dd>A position on the hash ring assigned to a server.</dd>
</dl>
</div>

---

## Quick Revision

<div class="revision-box">

**30-second interview answer:**

> "Consistent hashing maps both servers and keys onto a circular ring using a hash function. A key is routed to the first server found by walking clockwise from the key's position. When a server is added or removed, only 1/N of the keys need to be remapped — compared to modulo hashing where nearly all keys remap. It is used in Cassandra, DynamoDB, Redis Cluster, and CDN routing."

</div>

---

*Prev → [03. Algorithms](./03-algorithms.md) · Next → [05. Virtual Nodes](./05-virtual-nodes.md)*
