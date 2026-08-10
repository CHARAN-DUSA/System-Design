# 02. Memory Pressure

<div class="chapter-meta">
  <span class="meta-item meta-reading-time">⏱ 12 min read</span>
  <span class="meta-item meta-difficulty-intermediate">🟡 Intermediate</span>
  <span class="meta-item meta-prerequisites">📋 Prereq: Chapter 01</span>
</div>

> **Core idea:** When physical RAM fills up, the OS uses virtual memory — extending RAM onto disk using a swap space. If too many processes compete for too little RAM, the system falls into a swap loop. That loop is OS-level thrashing.

---

## Virtual Memory — The Illusion of Infinite RAM

Every process sees a **virtual address space** — as if it has all the RAM it wants. The OS maps these virtual addresses to **physical RAM frames** using a **page table**.

```
Process A thinks it has 8GB RAM:
  Virtual Address 0x0000 → Physical Frame 42 (in RAM)
  Virtual Address 0x1000 → Physical Frame 91 (in RAM)
  Virtual Address 0x2000 → Disk swap space  (not in RAM!)
                                              ↑
                                          Page Fault here
```

---

## Pages and Frames

| Term | Definition |
|---|---|
| **Page** | A fixed-size block of virtual memory (4 KB on x86) |
| **Frame** | A fixed-size block of physical RAM (same size as a page) |
| **Page Table** | OS data structure mapping virtual pages → physical frames |
| **Swap Space** | A section of disk used to store pages not currently in RAM |

The OS keeps the **most recently used** pages in RAM and moves **least recently used** pages to swap.

---

## Page Fault — Step by Step

A page fault occurs when a process accesses a virtual page that is **not currently in physical RAM**.

```
Step 1:  Process accesses address 0x2000
Step 2:  CPU checks page table → not in RAM (page fault!)
Step 3:  CPU generates a page fault interrupt
Step 4:  OS page fault handler takes over
Step 5:  OS finds a victim page to evict from RAM (LRU)
Step 6:  OS writes the victim page to swap (if dirty)
Step 7:  OS reads the required page from swap into RAM
Step 8:  OS updates the page table
Step 9:  Process resumes from the faulting instruction
```

**Timing:**
- RAM access: ~100 nanoseconds
- Page fault (SSD swap): ~100 microseconds = **1,000× slower**
- Page fault (HDD swap): ~10 milliseconds = **100,000× slower**

---

## The Kitchen Tomato Analogy

Imagine you are cooking and your kitchen counter (RAM) holds 10 items at a time. Your fridge (disk) holds 1,000 items.

- You are cooking pasta → keep flour, water, pasta on counter (no problem)
- You are cooking 50 dishes simultaneously → counter overflows
- To start each dish, you fetch ingredients from fridge (page in)
- To make room, you put other ingredients back in fridge (page out)
- You spend 80% of time walking to and from the fridge

**That walking back and forth = thrashing.**

The moment the fridge walking time dominates your cooking time, you are no longer cooking — you are just shuffling ingredients.

---

## The Swap Loop — How Thrashing Escalates

```
Too many processes
       │
       ▼
RAM fills up completely
       │
       ▼
Process A needs page → page fault → OS evicts page B to swap
       │
       ▼
Process B needs its evicted page → page fault → OS evicts page A to swap
       │
       ▼
Process A needs its evicted page → page fault → OS evicts page B again...
       ↑                                                          │
       └─────────────── Infinite loop of page faults ────────────┘
```

Result:
- **CPU iowait** → 90%+ (waiting for disk I/O)
- **CPU user** → near 0% (no useful work)
- **Disk I/O** → maxed out
- **Application** → completely frozen or extremely slow

---

## CPU Utilization vs Degree of Multiprogramming

This is the classical OS graph:

```
CPU Utilization (%)
100%│              ▲ Peak
    │           ╱    ╲
    │         ╱        ╲
    │       ╱            ╲ THRASHING ZONE
 50%│     ╱                ╲ ╲ ╲ ╲
    │   ╱                             ╲ ╲ ╲
  0%└──────────────────────────────────────────
    0   2   4   6   8   10  12  14  16  18
              Degree of Multiprogramming (# processes)
```

| Zone | Description |
|---|---|
| **Rising** | Adding more processes increases CPU utilization |
| **Peak** | Optimal point — all processes fit in RAM |
| **Declining (Thrashing)** | Too many processes → RAM overflow → swap loop |

---

## vmstat — Reading Thrashing Signals

```bash
vmstat 1

procs -----------memory---------- ---swap-- -----io---- --cpu--
 r  b   swpd   free   buff  cache   si   so    bi    bo   us sy id wa
 3  8  45000   2048   1024  4096  2800 2900  8500  9200   2  3  1 94
```

| Column | Value | Meaning |
|---|---|---|
| `swpd` | 45000 KB | 45MB in swap — pages have been evicted |
| `si` | 2800 KB/s | Swap in: 2.8MB/sec coming from disk to RAM |
| `so` | 2900 KB/s | Swap out: 2.9MB/sec going from RAM to disk |
| `wa` | 94% | **iowait: 94% — CPU waiting on disk I/O = THRASHING** |
| `us` | 2% | Only 2% CPU doing real work |

---

## Key Terms

<div class="key-terms">
<h4>📖 Glossary</h4>
<dl>
  <dt>Virtual Memory</dt>
  <dd>An abstraction that gives each process the illusion of a large, contiguous memory space.</dd>
  <dt>Page Fault</dt>
  <dd>An interrupt triggered when a process accesses a page not currently in physical RAM.</dd>
  <dt>Swap Space</dt>
  <dd>A disk partition or file used to store pages evicted from RAM.</dd>
  <dt>iowait</dt>
  <dd>The percentage of CPU time spent waiting for disk I/O to complete.</dd>
  <dt>Working Set</dt>
  <dd>The set of pages a process actively uses in a given time window.</dd>
</dl>
</div>

---

## Quick Revision

<div class="revision-box">

**30-second answer:**

> "Memory pressure thrashing happens when too many processes compete for limited RAM. The OS uses virtual memory — mapping pages to disk (swap) when RAM is full. When a process needs its swapped page, it generates a page fault, evicting another process's page. If two processes keep evicting each other, the swap loop never ends — that is thrashing. Signs: iowait >80%, si/so >0 in vmstat, frozen system."

</div>

---

*Prev → [01. Introduction](./01-introduction.md) · Next → [03. Database Thrashing](./03-database-thrashing.md)*
