# 01. Introduction to Thrashing

<div class="chapter-meta">
  <span class="meta-item meta-reading-time">⏱ 10 min read</span>
  <span class="meta-item meta-difficulty-beginner">🟢 Beginner</span>
</div>

> **Core idea:** Thrashing is when a system spends more time moving data between memory and disk — or between cache and database — than doing useful work. The CPU utilization collapses. Everything grinds to a halt.

---

## What is Thrashing?

Thrashing occurs when a system becomes so overloaded that it is constantly swapping data in and out — but making no real progress. It is a pathological feedback loop:

```
More processes → More page faults → More swapping → Less CPU for work
     ↑                                                        │
     └────────────── System appears "stuck" ←────────────────┘
```

---

## Programs, Processes, and Threads

Before understanding thrashing, you need to understand what runs in memory.

### Program

A **program** is a static set of instructions stored on disk (SSD/HDD). It is inactive.

```
Chrome installer on your SSD = Program (not using RAM or CPU)
```

Real-life analogy: A **recipe book** on the shelf.

### Process

A **process** is a program that is actively running. The OS loads it into RAM and assigns:

| Resource | Purpose |
|---|---|
| **PID** | Unique identifier |
| **RAM (Heap + Stack)** | Working memory |
| **CPU time** | Execution slots |
| **File descriptors** | Open files/sockets |
| **Virtual address space** | Memory map |

```
You open Chrome → OS creates a Process:
  PID: 4827
  RAM: 250MB loaded
  CPU: Running
```

Real-life analogy: A **chef actively cooking** using the recipe.

### Thread

A **thread** is the smallest unit of CPU execution inside a process. Threads inside the same process share memory.

```
Chrome Process (PID 4827)
  ├── Thread 1: UI rendering
  ├── Thread 2: Tab A (YouTube)
  ├── Thread 3: Tab B (Gmail)
  └── Thread 4: Downloads
```

Real-life analogy: **Workers in the kitchen** — same kitchen (process), each doing a different task (thread).

---

## Memory Hierarchy: The Foundation

| Level | Location | Speed | Size |
|---|---|---|---|
| **CPU Registers** | Inside CPU | ~0.3ns | Bytes |
| **L1 Cache** | On-chip | ~1ns | 32–64 KB |
| **L2 Cache** | On-chip | ~4ns | 256 KB – 1 MB |
| **L3 Cache** | On-chip (shared) | ~10ns | 4–32 MB |
| **RAM (DRAM)** | On motherboard | ~100ns | 4–256 GB |
| **SSD (NVMe)** | Slot on board | ~100µs | 500GB – 4TB |
| **HDD** | Mechanical | ~10ms | 1–20 TB |

**The critical ratio:**
- L1 Cache access: 1ns
- RAM access: 100ns → **100× slower**
- SSD access: 100,000ns → **100,000× slower**
- HDD access: 10,000,000ns → **10,000,000× slower**

---

## The Wardrobe vs Study Table Analogy

| Analogy | Actual System | Speed |
|---|---|---|
| Study Table (small, fast) | RAM (working memory) | ~100ns |
| Wardrobe (big, slow) | SSD/HDD (storage) | ~100µs–10ms |
| What you're reading NOW | CPU Registers + Cache | ~1ns |

When your study table is full and you need a new book, you **put one book back in the wardrobe** (swap out) and **fetch the new one** (swap in). If you're doing this constantly — more swapping than studying — you're **thrashing**.

---

## Types of Thrashing

| Type | Where it happens | Cause |
|---|---|---|
| **Memory Thrashing** | OS / RAM | Too many processes competing for physical RAM |
| **Database Thrashing** | DB buffer pool | Too many queries evicting each other's pages |
| **Cache Thrashing** | Redis / Memcached | Working set larger than cache capacity |
| **CPU Cache Thrashing** | L1/L2/L3 cache | Frequent context switches evict cache lines |

We cover each type in depth in the following chapters.

---

## How to Recognize Thrashing

| Signal | What you see |
|---|---|
| CPU iowait % | Spikes to 80–100% |
| Disk reads/writes | Extremely high, constantly |
| CPU user % | Drops to near zero |
| Application | Extremely slow or frozen |
| `vmstat` output | High `si` (swap in) + `so` (swap out) |
| Database | Slow queries despite good indexes |
| Cache | Cache hit rate drops below 50% |

---

## Key Terms

<div class="key-terms">
<h4>📖 Glossary</h4>
<dl>
  <dt>Thrashing</dt>
  <dd>A state where a system spends more resources moving data than performing useful computation.</dd>
  <dt>Process</dt>
  <dd>A running instance of a program with its own memory space.</dd>
  <dt>Thread</dt>
  <dd>The smallest schedulable unit inside a process. Threads share the process's memory.</dd>
  <dt>RAM</dt>
  <dd>Random Access Memory — the fast, volatile working memory of a computer.</dd>
  <dt>Page</dt>
  <dd>A fixed-size block of virtual memory (typically 4KB) used by the OS.</dd>
</dl>
</div>

---

## Quick Revision

<div class="revision-box">

**30-second answer:**

> "Thrashing is when a system spends more time managing memory than doing useful work. For OS-level thrashing, too many processes compete for RAM, causing constant page faults and swapping. The CPU appears busy but does no real work. The fix is to reduce the degree of multiprogramming (run fewer processes), add RAM, or improve memory access patterns."

</div>

---

*Next → [02. Memory Pressure](./02-memory-pressure.md)*
