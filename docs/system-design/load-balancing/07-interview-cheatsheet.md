---
title: "Interview Cheat Sheet (Mana Style)"
description: Master System Design interviews with this quick-revision guide, featuring the complete MediSphere story, key takeaways, and a Python consistent hashing implementation.
tags:
  - System Design
  - Interview Prep
  - Cheat Sheet
  - Consistent Hashing Implementation
  - Python
---

# 📝 Interview Cheat Sheet (Mana Style)

> This chapter is your go-to revision checklist before interviews. It explains the core concepts in a story format ("Mana Style"), lists the golden interview guidelines, and provides a Python implementation of Consistent Hashing with VNodes.

---

<div class="chapter-meta" markdown>
<span class="meta-item meta-reading-time">⏱ 8 min read</span>
<span class="meta-item meta-difficulty-beginner">🟢 All Levels</span>
</div>

---

## 🎯 One-Line Revision (Mana Style)

| Concept | "Mana Style" Metaphor | Real-World Job |
|---|---|---|
| **Load Balancer** | "Traffic Police" 🚦 | Routes incoming traffic to active servers. |
| **API Server** | "Manager" 👨‍💼 | Runs business logic, auth, and validation. |
| **Database** | "Godown" 📦 | Stores persistent, transactional data. |
| **Redis** | "Fridge" 🧊 | Temporary fast cache for hot reads. |
| **Stateless API**| "Carry your own ID" 🪪 | Uses client-side tokens (JWT); no local session RAM. |
| **Consistent Hashing**| "Ring Routing" 🎯 | Maps database keys to nodes minimizing rehashing. |
| **Virtual Nodes**| "Multiple Desks per Worker" 🪑 | Distributes data load evenly across storage units. |
| **Replication** | "Backup Copies" 📑 | Syncs primary writes to read-only replica instances. |
| **Cloud Auto-Scaling**| "Temp Agency" 🤖 | Spins up new instances if servers crash or load peaks. |

---

## 🚀 Complete System Design Story (Mana Style)

Imagine we launched **MediSphere** today.

Initially, we only have **1,000 requests/hour**. The architecture is simple:

$$\text{Users} \longrightarrow \text{ASP.NET API} \longrightarrow \text{PostgreSQL DB}$$

Users submit requests. The API handles authentication (JWT validation) and business logic, then writes or reads from the database. The API server stores no data in local memory; SQL Server stores all state.

A few months later, MediSphere becomes popular. Traffic reaches **10,000 requests/hour**. A single API server starts crashing. We provision 3 API instances and place them behind a **Load Balancer**.

```
                 ┌───► API Server 1 ───┐
                 │                     │
Users ──► Load Balancer ──► API Server 2 ───┼──► Shared Database
                 │                     │
                 └───► API Server 3 ───┘
```

The Load Balancer acts like a traffic controller. It checks server health and forwards requests (using algorithms like Round Robin or Least Connections). To make this work, the APIs must be **Stateless**. The user's token (JWT) is stored in the browser. Whether a request lands on API 1 or API 3, the response is identical because they both fetch persistent data from the same SQL Database.

Traffic hits **100,000 requests/hour**. The API tier is fine, but the SQL Database disk I/O spikes. The database is slowing down. First, we optimize queries and create **indexes**. If that is not enough, we introduce **Redis Caching**.

```
                                  ┌───► Redis Cache (Fast RAM Reads)
                                  │
Load Balancer ──► API Servers ────┤
                                  │
                                  └───► PostgreSQL Database (Fallback)
```

Redis is our fast RAM-based cache. It uses the **Cache-Aside Pattern**:
- If the data is in Redis (Cache Hit), return it immediately.
- If not (Cache Miss), query the SQL database, save the result to Redis, and return it.

We cache static info (specialty departments, doctor schedules) but bypass Redis for transactional data (payment gateways, booking confirmations).

Traffic grows to **500,000 requests/hour**. Read operations saturate our database. We set up **Read Replicas** with a Primary database handling writes and Replicas handling reads.

Finally, we hit **Millions of users**. A single SQL Database cannot store all patient records. We must implement **Database Sharding**. To determine which shard holds a patient's record, we use **Consistent Hashing**:

```
                       [0 / 2^32 - 1]
                       
                   S1 (Hash: 1,000,000)
                         /        \
                        /          \
   🔑 User_C (15M) ──► /            \ ◄── 🔑 User_A (2.5M)
                      |   HASH RING  |
                       \            /
                        \          /
                   S3 (Hash: 12M) ─── S2 (Hash: 5M)
                         ▲
                         │
                   🔑 User_B (8M)
```

We map both our database shards (servers) and patient IDs (keys) onto the same circular **Hash Ring** by hashing their IDs. To route a request, we move **clockwise** from the key's hash position until we hit the nearest server. 

If a server crashes, only its keys migrate to the next clockwise node. To prevent overloading that single neighbor, we implement **Virtual Nodes** (assigning multiple logical locations on the ring to each physical machine). This distributes the failure load evenly across all surviving shards.

---

## 💻 Consistent Hashing Python Implementation

Here is a complete, executable Python script demonstrating Consistent Hashing with Virtual Nodes:

```python
import hashlib
import bisect

class ConsistentHashRing:
    def __init__(self, replicas=100):
        """
        replicas: Number of virtual nodes (VNodes) per physical server.
        """
        self.replicas = replicas
        self.ring = []        # Sorted list of active VNode hash keys
        self.vnode_map = {}  # Map: VNode Hash -> Physical Server Name

    def _hash(self, key: str) -> int:
        """Helper to generate a 32-bit integer hash from a string key."""
        sha = hashlib.sha1(key.encode('utf-8')).hexdigest()
        return int(sha, 16) % (2**32)

    def add_node(self, node: str):
        """Adds a physical server and its virtual nodes to the ring."""
        for i in range(self.replicas):
            vnode_key = f"{node}-vnode-{i}"
            vnode_hash = self._hash(vnode_key)
            
            # Map the VNode hash position to the physical server name
            self.vnode_map[vnode_hash] = node
            
            # Keep the ring sorted
            bisect.insort(self.ring, vnode_hash)
        print(f"Added node '{node}' with {self.replicas} VNodes to the ring.")

    def remove_node(self, node: str):
        """Removes a physical server and its virtual nodes from the ring."""
        for i in range(self.replicas):
            vnode_key = f"{node}-vnode-{i}"
            vnode_hash = self._hash(vnode_key)
            
            # Remove from ring list
            if vnode_hash in self.vnode_map:
                self.ring.remove(vnode_hash)
                del self.vnode_map[vnode_hash]
        print(f"Removed node '{node}' from the ring.")

    def get_node(self, key: str) -> str:
        """Finds the nearest clockwise node handling the given key."""
        if not self.ring:
            return None
            
        key_hash = self._hash(key)
        
        # Binary search: find the first VNode hash >= key_hash
        idx = bisect.bisect_right(self.ring, key_hash)
        
        # If we reach the end of the ring list, wrap around to index 0
        if idx == len(self.ring):
            idx = 0
            
        target_vnode_hash = self.ring[idx]
        return self.vnode_map[target_vnode_hash]

# --- Verification and Execution Demo ---
if __name__ == "__main__":
    # Create a Consistent Hash Ring
    ring = ConsistentHashRing(replicas=3) # small replica size for display trace
    
    # Add servers
    ring.add_node("Database_Shard_1")
    ring.add_node("Database_Shard_2")
    ring.add_node("Database_Shard_3")
    
    # Route keys
    users = ["User_100", "User_500", "User_999", "User_1234"]
    print("\n--- Key Routing Trace ---")
    for user in users:
        print(f"Key: '{user}' maps to Node: '{ring.get_node(user)}'")
        
    # Simulate database crash
    print("\n--- Failover Simulation: Shard 2 Crashes ---")
    ring.remove_node("Database_Shard_2")
    for user in users:
        print(f"Key: '{user}' now routes to Node: '{ring.get_node(user)}'")
```

---

## 🧠 Self-Assessment Quiz

??? question "Question 1: What is the Golden Rule of Scaling?"
    **Answer:** "Measure $\to$ Identify Bottleneck $\to$ Optimize $\to$ Scale $\to$ Repeat". Do not guess where the slowdown is, and do not over-engineer by using complex systems (like sharding or message streams) before exhausting query indexing and caching.

??? question "Question 2: How do you answer the interview question: 'Your platform grows from 1K users to 1M users, what do you do?'?"
    **Answer:** Explain the scaling path systematically:
    1. Check bottlenecks first.
    2. Scaled the APIs horizontally using a load balancer.
    3. Transition to stateless APIs.
    4. Implement indexes, query caching, and connection poolers.
    5. Add Redis cache (Cache-Aside pattern).
    6. Implement Read/Write splitting with replicas.
    7. Partition tables internally or implement Sharding.
    8. Introduce Consistent Hashing with VNodes for database distribution.
    9. Move processing tasks to async background jobs (queues).

---

<div class="navigation-footer" markdown>

[⬅️ Case Study: MediSphere Roadmap](06-medisphere-case-study.md)

[🏠 System Design Home](../index.md)

</div>
