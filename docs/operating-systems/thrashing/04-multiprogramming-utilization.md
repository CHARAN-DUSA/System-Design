---
title: CPU Utilization & Multiprogramming
description: Analyze the relationship between CPU Utilization and the Degree of Multiprogramming, and learn how overloading RAM triggers system crashes.
tags:
  - CPU Utilization
  - Multiprogramming
  - System Performance
  - Memory Management
  - OS Scheduling
---

# 📈 CPU Utilization & Multiprogramming

> To maximize system value, an Operating System tries to run multiple programs concurrently. However, loading too many processes into memory creates a tipping point where CPU utilization drops off a cliff.

---

<div class="chapter-meta" markdown>
<span class="meta-item meta-reading-time">⏱ 8 min read</span>
<span class="meta-item meta-difficulty-intermediate">🟡 Intermediate</span>
</div>

---

## ⚙️ Key Concepts

### 1. CPU Utilization
The percentage of time the CPU spends executing useful user processes (e.g., calculations, encoding, rendering).
* **High Utilization ($80\text{–}95\%$):** Good. The CPU is busy doing useful work.
* **Low Utilization ($<20\%$):** Bad. The CPU is mostly sitting idle, waiting for slow disk transfers.

### 2. Degree of Multiprogramming
The number of active processes loaded into RAM at the same time.
* **Low Degree:** Only 2 apps open (e.g., VS Code and Chrome).
* **High Degree:** 15 apps open at once (Chrome with 40 tabs, Spotify, VS Code, Slack, Docker, BGMI).

---

## 🏫 The Real-Life Metaphor: The Computer Lab

Imagine a school computer lab:

```
                  ┌──────────────────────┐
                  │     COMPUTER LAB     │
                  │  🖥️   🖥️   🖥️   🖥️  │ (RAM: 4 slots)
                  └──────────────────────┘
                             ▲
                Students Kicked Out / Swapping
                             ▼
                  ┌──────────────────────┐
                  │    WAITING ROOM      │
                  │  👨  👩  👨  👩  👨   │ (Suspended Processes)
                  └──────────────────────┘
```

* **10 Computers (RAM Capacity):** There are only 10 workstations.
* **30 Students (Processes):** 30 students want to complete their assignments.

### Scenario A: Too many students inside (Thrashing)
If the teacher allows all 30 students to pack into the lab simultaneously:
- Student 1 sits down, starts typing.
- Two minutes later, Student 2 taps Student 1 on the shoulder: *"My turn!"*
- Student 1 stands up, saves their work, and walks away. Student 2 sits down, loads their files.
- Two minutes later, Student 3 kicks out Student 2.
- **Result:** Everyone is constantly swapping seats. No student finishes their assignment. The output of the lab is close to **zero**.

### Scenario B: OS Intervention (Controlling Multiprogramming)
The teacher stands at the door and sets a rule:
- *"Only 10 students inside the lab at a time."*
- The remaining 20 students must wait in the hallway (**Suspended State**).
- The 10 students inside finish their work quickly without interruption, exit, and the next 10 enter.
- **Result:** Assignments are completed smoothly and quickly.

---

## 📊 The Multiprogramming Curve

In memory management, there is a famous curve representing the relationship between the Degree of Multiprogramming and CPU Utilization:

```
  CPU
Utilization
  100% |             *  Peak Performance
       |           *   *
       |         *       *
       |       *           *
       |     *               *  🚨 Collapse (Thrashing)
       |   *                   *
    0% +-----------------------------
       0                             Max
            Degree of Multiprogramming
```

### Explaining the Curve Stages:
1. **Steady Increase:** As you open more applications (increasing the degree of multiprogramming), the CPU gets busier because it can switch to another task if one is waiting. CPU utilization rises.
2. **The Tipping Point (Peak):** The RAM is fully utilized. Every process has just enough frames to hold its working set.
3. **The Collapse (Thrashing Cliff):** You open one more application. Now, there are not enough physical RAM frames to hold the working sets of all running processes. The OS spends all its time swapping pages in and out of storage. **CPU utilization plunges to near-zero** because the CPU is constantly idle, waiting for disk reads.

---

## 🧠 Self-Assessment Quiz

??? question "Question 1: Why does a scheduler sometimes swap entire processes out to disk (suspension)?"
    **Answer:** If the system is thrashing, the OS scheduler must decrease the degree of multiprogramming. By suspeding one or more processes and writing their entire memory space out to disk, it frees up physical frames for the remaining processes, allowing them to stop faulting and finish their work.

??? question "Question 2: What is 'Global Page Replacement' vs 'Local Page Replacement'?"
    **Answer:** 
    - **Global Replacement:** A process can steal a memory frame from any other process in the system. This can trigger thrashing across the entire system if one memory-hungry process starts stealing frames from healthy processes.
    - **Local Replacement:** A process can only replace frames allocated to *itself*. This limits thrashing to the misbehaving process, keeping the rest of the OS stable.

??? question "Question 3: How does the OS detect that it is thrashing?"
    **Answer:** The OS monitors the rate of page faults and page disk I/O operations. If page faults exceed a configured threshold and CPU utilization drops, the OS detects that it is spending too much time swapping and must intervene.

---

<div class="navigation-footer" markdown>

[⬅️ Page Faults & Thrashing](03-page-faults-thrashing.md)

[➡️ How to Prevent Thrashing](05-prevention-solutions.md)

</div>
