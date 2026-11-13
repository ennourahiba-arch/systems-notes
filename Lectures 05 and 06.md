# Systems and Networking I — Part II: CPU Scheduling (Lectures 5–6)

Study notes for **Lecture 5 (CPU Scheduling 1)** and **Lecture 6 (CPU Scheduling 2)**.

> Exam format: multiple choice, ~20 questions, **+3** correct, **−1** wrong, **0** blank. Don't guess blindly.
> Scheduling calculations (FCFS, RR, SJF, SRTF, waiting and turnaround times) come up in **every** past exam.

## Table of contents

1. [Basic concepts](#1-basic-concepts)
2. [When scheduling happens: preemptive vs non-preemptive](#2-when-scheduling-happens-preemptive-vs-non-preemptive)
3. [The dispatcher](#3-the-dispatcher)
4. [Definitions and metrics](#4-definitions-and-metrics)
5. [How to solve a scheduling exercise](#5-how-to-solve-a-scheduling-exercise)
6. [First-Come-First-Serve (FCFS)](#6-first-come-first-serve-fcfs)
7. [Round Robin (RR)](#7-round-robin-rr)
8. [Shortest-Job-First (SJF) and SRTF](#8-shortest-job-first-sjf-and-srtf)
9. [Priority scheduling](#9-priority-scheduling)
10. [Multilevel queues: MLQ and MLFQ](#10-multilevel-queues-mlq-and-mlfq)
11. [Lottery scheduling](#11-lottery-scheduling)
12. [Summary of algorithms](#12-summary-of-algorithms)
13. [Worked example: one job set, four algorithms](#13-worked-example-one-job-set-four-algorithms)
14. [Exam traps (Lectures 5–6)](#14-exam-traps-lectures-56)
15. [Practice questions (Rounds 1–4)](#15-practice-questions-rounds-14)
16. [Exam-style questions](#16-exam-style-questions)
17. [Oral exam questions](#17-oral-exam-questions)

---

## 1. Basic concepts

**CPU scheduling** = the **policy** that decides which process runs on the CPU.

- Almost every program alternates between **CPU computation** and **waiting for I/O**. Even a fetch from main memory is slow compared to the CPU.
- **Assumptions:** a **multiprogrammed, uniprocessor** system.
- With a single process, the time spent waiting for I/O is wasted: those CPU cycles are lost forever.
- Scheduling lets **one process use the CPU while another waits for I/O**, maximizing system utilization.
- **Challenge:** be as **efficient** and **fair** as possible under changing conditions.

Processes managed by the OS are always in **exactly one** state queue.

### CPU burst vs I/O burst

Every process alternates between two phases in a continuing cycle:

- **CPU burst:** performing calculations.
- **I/O burst:** waiting for data to be transferred in or out.

The distribution of CPU burst lengths is **highly skewed**: **most** processes have **short** CPU bursts, and **few** have very long ones.

### Long-term vs short-term scheduling

| | Long-term scheduling | Short-term scheduling (**CPU scheduler**) |
| --- | --- | --- |
| Question | How many processes to load in memory (the **level of multiprogramming**) | Which process in the **ready queue** runs next |

### CPU scheduler

- Whenever the CPU becomes **idle**, it picks another process from the **ready queue**.
- The ready queue is **not necessarily FIFO**: its data structure and the selection algorithm depend on the policy (policy vs mechanism again).

---

## 2. When scheduling happens: preemptive vs non-preemptive

CPU scheduling decisions take place under **4 conditions**:

| # | Transition | Cause | Choice? |
| --- | --- | --- | --- |
| 1 | Running → **Waiting** | I/O request or `wait` system call | **No choice:** a new process must be selected |
| 2 | Running → **Ready** | An interrupt (e.g. timer) | Continue the current process or pick a new one |
| 3 | Waiting → **Ready** | I/O completion or return from `wait` | Continue the current process or pick a new one |
| 4 | Process **created** or **terminates** | — | **No choice** (on termination): a new process must be selected |

| | Non-preemptive scheduling | Preemptive scheduling |
| --- | --- | --- |
| Runs on | **Only** conditions **1 and 4** (no choice) | Conditions 1 and 4 **and also 2 and 3** |
| Behavior | Once a process starts, it runs until it **voluntarily blocks or finishes** | The OS can take the CPU away (e.g. timer interrupt) |

> ⚠️ **A non-preemptive scheduler still runs** on running → waiting and on termination. "Non-preemptive" doesn't mean "never scheduled".

| | Windows | Mac | UNIX-like |
| --- | --- | --- | --- |
| Non-preemptive | Up to Windows 3.x | Up to Mac OS 9.x | — |
| Preemptive | Since Windows 95 | Since Mac OS X | Since forever |

### Problems with preemption

Preemption can cause trouble if it happens while:

- the **kernel is executing a system call** (e.g. updating critical kernel data structures)
- **two processes share data** and one is interrupted in the middle of updating it

Countermeasures:

| Countermeasure | Drawback |
| --- | --- |
| Wait until the system call has **completed or blocked** before preempting | Bad for **real-time systems**: real-time response can't be guaranteed |
| **Disable interrupts** before a critical section and re-enable them right after | Only for **rare** situations and **very short** pieces of code |

---

## 3. The dispatcher

The **dispatcher** is the module that **gives control of the CPU** to the process selected by the scheduler. Its functions:

1. **Switching context**
2. **Switching to user mode**
3. **Jumping to the proper location** in the newly loaded program

It runs on **every context switch**, so its time — the **dispatch latency** — must be as short as possible.

> Scheduler = **decides** who runs. Dispatcher = **hands over** the CPU.

---

## 4. Definitions and metrics

### Time definitions

| Term | Definition |
| --- | --- |
| **Arrival time** | When the process arrives in the **ready queue** |
| **Completion time** | When the process **finishes** its execution |
| **Burst time** | CPU time the process needs |
| **Turnaround time** | `completion − arrival` |
| **Waiting time** | `turnaround − burst` |

> In these calculations, **I/O waiting time is not counted** in the waiting time.

```
turnaround_i = completion_i − arrival_i
waiting_i    = turnaround_i − burst_i
avg waiting  = (1/N) · Σ (turnaround_i − burst_i)
```

Unless stated otherwise, assume **all jobs arrive at time 0**. Then turnaround = completion time.

### Scheduling criteria

| Metric | Meaning | Goal |
| --- | --- | --- |
| **CPU utilization** | % of time the CPU is busy. Ideally 100%; real systems: 40% (light load) to 90% (heavy load) | **Maximize** |
| **Throughput** | Number of processes **completed per unit of time** (from 10/s to 1/hour) | **Maximize** |
| **Turnaround time** | Time from **submission to completion**, including all waiting | **Minimize** |
| **Waiting time** | Time spent in the **ready queue** waiting for the CPU | **Minimize** |
| **Response time** | Time from issuing a command to the **beginning** of the response (mostly interactive processes) | **Minimize** |

> ⚠️ **Waiting time ≠ the waiting state.** Waiting time counts time in the **ready** queue. Processes in the waiting state aren't under the CPU scheduler's control: they simply aren't ready to run.

### Trade-off and policies

No scheduler optimizes all metrics at once. **Choose the algorithm according to the policy** you want:

| Policy | Typical of |
| --- | --- |
| Minimize **average** response time (output to the user quickly) | Interactive systems |
| Minimize the **maximum** response time (worst-case bound) | Interactive systems |
| Minimize the **variance** of response time (users prefer a predictable system) | Interactive systems |
| Maximize **throughput**: minimize overhead (context switches), use CPU and I/O efficiently | Batch systems |
| Minimize **waiting time**: give every process the same CPU time (may increase average response time) | Batch systems |

### Terminology and assumptions

- **Job / task:** general unit of CPU execution. **Process:** the actual unit of execution. **Thread:** coming soon.
- Assumptions: **one process per user**, processes are **independent** (no communication), **one thread per process**.

### The algorithms

FCFS · Round Robin · SJF (and SRTF) · Priority · Multilevel Queue (MLQ) · Multilevel Feedback Queue (MLFQ) · Lottery.

---

## 5. How to solve a scheduling exercise

1. **Write the table:** job, arrival, burst.
2. **Draw the Gantt chart** step by step. At every decision point, ask: *who is in the ready queue right now, and what does the algorithm pick?*
3. **Read the completion time** of each job from the chart.
4. **Compute** `turnaround = completion − arrival` and `waiting = turnaround − burst`.
5. **Average** over all jobs.

> ✅ **Self-check:** for every job, waiting ≥ 0, and the chart's total length = sum of bursts (if the CPU is never idle).

**RR convention for ties** (the slides don't state one; it's the common textbook convention): if a new job **arrives** at the same instant a job's quantum **expires**, put the **new arrival** in the queue **first**, then the preempted job at the back. If an exam question specifies otherwise, follow the question.

---

## 6. First-Come-First-Serve (FCFS)

- A plain **FIFO queue**, like customers in line at the post office.
- Jobs run **to completion in arrival order**.
- The scheduler takes over **only** when the running job asks for **I/O** or **finishes**. A job can keep the CPU indefinitely (until it blocks). → **Non-preemptive.**

### Scenario I: order A, B, C (arrival 0, no I/O)

| Job | Burst |
| --- | --- |
| A | 5 |
| B | 2 |
| C | 3 |

```
| A         | B   | C     |
0           5     7       10
```

Waiting: A = 0, B = 5, C = 7 → **avg = (0 + 5 + 7) / 3 = 4**

### Scenario II: same jobs, order B, C, A

```
| B   | C     | A         |
0     2       5           10
```

Waiting: B = 0, C = 2, A = 5 → **avg = (5 + 0 + 2) / 3 ≈ 2.3**

Same jobs, different order → very different average. **FCFS waiting time depends heavily on arrival order.**

### Scenario III: A also does I/O

A runs 2 units, then makes an I/O request.

```
| A   | B   | A       | C       |
0     2     4         7         10
```

- At t = 2, A goes to **waiting** (I/O); B runs.
- A's I/O may finish **before** B does, but FCFS is **non-preemptive**: B keeps the CPU until t = 4.
- At t = 4, **A** runs, not C. Why? FCFS only cares about **arrival order in the ready queue**, and A came back before C got its turn.
- avg waiting = (2 + 2 + 7) / 3 ≈ **3.7**. Strictly, the time A spent **doing I/O** should be removed from its waiting time.

### Pros and cons

- ✅ **Very simple.**
- ❌ **Highly variable** average waiting time: short jobs may sit behind very long ones.
- ❌ **Convoy effect:** poor CPU/I/O overlap, because CPU-bound jobs force I/O-bound jobs to wait.

---

## 7. Round Robin (RR)

- Like FCFS, but each turn on the CPU is limited by a **time quantum** (time slice).
- When a job gets the CPU, a **timer** is set:
  - job finishes **before** the quantum expires → swapped out like in FCFS
  - **timer goes off first** → job is swapped out and moved to the **back** of the ready queue
- Used in **time-sharing** systems, with **timer interrupts**. → **Preemptive.**
- The ready queue is a **circular queue**: when every job has had a turn, the first gets another, and so on.
- ✅ **Fair:** shares the CPU equally.
- ❌ **Average waiting time can be long.**

### The time quantum

RR's performance is **sensitive to the quantum**:

| Quantum | Effect |
| --- | --- |
| **Too large** | Jobs are never preempted → **degenerates into FCFS** (high average waiting time) |
| **Too small** | Many context switches, whose overhead **dominates** useful work → **low throughput** |

**Trade-off:** context-switch overhead must be **small compared to the time slice**. Example: time slice = 10–100 ms, context switch = 0.01–0.1 ms.

- Smaller quantum → **more context switches**.
- RR gives an **upper bound** on the time before a job is scheduled for the first time. Worst case: every job ahead of it uses its whole slice.

### Example: quantum = 2, context switch = 0

Same jobs as FCFS Scenario I (A = 5, B = 2, C = 3, all arrive at 0):

```
| A   | B   | C   | A   | C | A |
0     2     4     6     8   9   10
```

- A: completes at 10 → waiting = 10 − 5 = **5**
- B: completes at 4 → waiting = 4 − 2 = **2**
- C: completes at 9 → waiting = 9 − 3 = **6**

**avg waiting = (5 + 2 + 6) / 3 ≈ 4.3** (worse than FCFS Scenario I's 4)

### FCFS vs RR: 5 jobs with **equal** bursts (100 each, quantum = 1, arrival 0)

| | Avg turnaround | Avg waiting |
| --- | --- | --- |
| FCFS | (100 + 200 + 300 + 400 + 500) / 5 = **300** | (0 + 100 + 200 + 300 + 400) / 5 = **200** |
| RR | (496 + 497 + 498 + 499 + 500) / 5 = **498** | **398** |

FCFS looks better on both averages… but is it **fair**? Look at the **variance**, not just the average: under FCFS the last job waits 400, the first waits 0.

### FCFS vs RR: 5 jobs with **different** bursts (quantum = 1, arrival 0)

| Job | Burst | Turnaround FCFS | Turnaround RR | Waiting FCFS | Waiting RR |
| --- | --- | --- | --- | --- | --- |
| A | 50 | 50 | 150 | 0 | 100 |
| B | 40 | 90 | 140 | 50 | 100 |
| C | 30 | 120 | 120 | 90 | 90 |
| D | 20 | 140 | 90 | 120 | 70 |
| E | 10 | 150 | 50 | 140 | 40 |
| **Avg** | | **110** | **110** | **80** | **80** |

Same averages, but RR treats short jobs much better (E: 40 instead of 140).

---

## 8. Shortest-Job-First (SJF) and SRTF

**Idea:** run the job with the **least expected amount of work** (CPU burst) until its next I/O or termination.

### Example (all arrive at 0)

| Job | Burst |
| --- | --- |
| A | 6 |
| B | 8 |
| C | 7 |
| D | 3 |

```
| D     | A           | C             | B               |
0       3             9               16                24
```

Waiting: A = 3, B = 16, C = 9, D = 0 → **avg = (3 + 16 + 9 + 0) / 4 = 7**

### Pros and cons

- ✅ **Provably optimal** for minimizing **average waiting time**.
- ✅ Works with both non-preemptive and preemptive schedulers. **Preemptive SJF = SRTF** (Shortest Remaining Time First).
- ❌ It's **almost impossible to know** a job's CPU time in advance.
- ❌ Long CPU-bound jobs can **starve**: I/O-bound (short-burst) jobs implicitly have higher priority.

### Estimating the next CPU burst: exponential smoothing

Predict the next burst from the process's recent history, using a **weighted average** of the **previous observation** and the **previous prediction**:

```
x_t     = actual length of the t-th CPU burst
s_(t+1) = predicted length of the (t+1)-th CPU burst
0 ≤ α ≤ 1

s_1     = x_0                          (bootstrap)
s_(t+1) = α · x_t + (1 − α) · s_t
```

| α | Result | Meaning |
| --- | --- | --- |
| **0** | `s_(t+1) = s_t` | Observed bursts are **ignored**: a constant burst is assumed |
| **1** | `s_(t+1) = x_t` | The next burst = the **last** observed burst; older history doesn't count |
| **0.5** (usual) | Equal weight to the last observation and the last prediction | |

**Slide example (α = 0.5):**

| t | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Observation `x_t` | 10 | 6 | 4 | 6 | 4 | 13 | 13 | 13 |
| Prediction `s_(t+1)` | 10 | 8 | 6 | 6 | 5 | 9 | 11 | 12 |

E.g. `s_6 = 0.5 · 13 + 0.5 · 5 = 9`.

Expanding the formula, each older observation's weight shrinks **geometrically**:

```
s_(t+1) = α·x_t + α(1−α)·x_(t−1) + α(1−α)²·x_(t−2) + … + (1−α)^t · x_0
```

### SJF vs SRTF

| | SJF | SRTF |
| --- | --- | --- |
| Type | **Non-preemptive** | **Preemptive** |
| Rule | Once given the CPU, a process runs until its CPU burst ends | Preempt when a **new arrival's burst < remaining burst** of the running process |

### SRTF example

| Job | Arrival | Burst |
| --- | --- | --- |
| A | 0 | 8 |
| B | 1 | 4 |
| C | 2 | 9 |
| D | 3 | 5 |

```
| A | B       | D         | A               | C                   |
0   1         5           10                17                    26
```

- t = 1: B arrives (4) < A's remaining (8 − 1 = 7) → **B preempts A** and runs to 5.
- t = 5: ready: A (7 left), C (9), D (5) → **D** runs to 10.
- t = 10: A (7) vs C (9) → **A** runs to 17.
- t = 17: **C** runs to 26.

Waiting (`completion − arrival − burst`):
A = 17 − 0 − 8 = 9, B = 5 − 1 − 4 = 0, C = 26 − 2 − 9 = 15, D = 10 − 3 − 5 = 2

**avg = 26 / 4 = 6.5**

### FCFS vs RR vs SJF (5 jobs, quantum = 1, arrival 0)

| Job | Burst | TA FCFS | TA RR | TA SJF | W FCFS | W RR | W SJF |
| --- | --- | --- | --- | --- | --- | --- | --- |
| A | 50 | 50 | 150 | 150 | 0 | 100 | 100 |
| B | 40 | 90 | 140 | 100 | 50 | 100 | 60 |
| C | 30 | 120 | 120 | 60 | 90 | 90 | 30 |
| D | 20 | 140 | 90 | 30 | 120 | 70 | 10 |
| E | 10 | 150 | 50 | 10 | 140 | 40 | 0 |
| **Avg** | | **110** | **110** | **70** | **80** | **80** | **40** |

SJF wins on averages: as expected, it's optimal.

---

## 9. Priority scheduling

- Generalization of SJF: each job has a **priority**, and the **highest-priority** job runs first.
- **SJF is priority scheduling** where priority = predicted next CPU burst.
- Priorities are integers in a fixed range. No universal convention, but **usually low numbers = high priority** (0 = highest).
- **Internal** priorities: set by the OS (average burst time, CPU/I/O ratio, resource use…).
- **External** priorities: set by users (importance, fees paid, politics…).
- Can be **preemptive or non-preemptive**.

### Starvation and aging

- **Starvation (indefinite blocking):** a low-priority job may wait **forever** because others always have higher priority. It might only run when the load gets lighter, or after a crash and reboot.
- **Aging:** **increase a job's priority the longer it waits**, until it is eventually scheduled. → Solves starvation.

---

## 10. Multilevel queues: MLQ and MLFQ

### Multilevel Queue (MLQ)

- **Several separate queues**, one per **job category**.
- Each queue uses **its own algorithm**, the best fit for that job type.
- Scheduling is also needed **between** queues:
  - **Strict priority:** no job in a lower-priority queue runs until all higher-priority queues are **empty**.
  - **Round robin:** each queue gets a time slice in turn (possibly of different sizes).
- ⚠️ **Jobs cannot move between queues.**
- The time slice usually **increases (exponentially) as priority gets lower**.

### Multilevel Feedback Queue (MLFQ)

Like MLQ, **but jobs can move between queues**. A job may move when:

- its behavior changes between **CPU-intensive** and **I/O-intensive**
- it has waited a long time → bumped up to a higher-priority queue for a while (**aging**)

**Adjusting priority:**

| Event | Effect |
| --- | --- |
| New job | Starts in the **highest**-priority queue |
| **Time slice expires** | Priority **drops one level** |
| Time slice **does not** expire (it blocked for I/O) | Priority **rises one level** (up to the top) |

→ **CPU-bound** jobs quickly sink; **I/O-bound** jobs stay at high priority.

MLFQ is the **most flexible** but also the **most complex**. Parameters: number of queues, algorithm per queue, how jobs are promoted/demoted, which queue a job enters first.

### MLFQ Example I: 3 queues, strict priority, no I/O

Jobs: A = 30, B = 20, C = 10. Notation: `A^k_t` = A has received **k** total units of CPU at time **t**.

| Queue | Time slice | Progress |
| --- | --- | --- |
| 1 | 1 | `A^1_1`, `B^1_2`, `C^1_3` |
| 2 | 2 | `A^3_5`, `B^3_7`, `C^3_9` |
| 3 | 4 | `A^7_13`, `B^7_17`, `C^7_21`, `A^11_25`, `B^11_29`, `C^10_32` (C done) … |

Each job uses its whole slice → demoted until it reaches the bottom queue.

### MLFQ Example II: 2 queues, C alternates 1 unit of CPU with 1 unit of I/O

| Queue | Time slice | Progress |
| --- | --- | --- |
| 1 | 1 | `A^1_1`, `B^1_2`, `C^1_3`, `C^2_6`, `C^3_9`, `C^4_12`, …, `C^10_30` |
| 2 | 2 | `A^3_5`, `B^3_8`, `A^5_11`, `B^5_14`, …, `B^12_32`, `A^14_34`, … |

C never uses its whole slice (it blocks for I/O), so it **stays in the top queue** and gets the CPU as soon as it's ready. A and B (CPU-bound) sink to queue 2.

### Fairness

- MLFQ tries to **mimic SJF** (good average waiting time) by **favoring short (I/O-bound) jobs**.
- Problem: SJF and MLFQ can be **unfair** (unlike RR). Any fairness gained by giving long jobs CPU time increases waiting time.
- Ways to improve fairness:
  - give **each queue a fraction** of the CPU (fair only if jobs are evenly spread across queues)
  - **raise the priority** of jobs that don't get scheduled (avoids starvation, but when overloaded all jobs reach the top queue and waiting time rises)

---

## 11. Lottery scheduling

- Each job gets some **lottery tickets**.
- On each time slice, a winning ticket is drawn **uniformly at random**.
- In the long run (**law of large numbers**), CPU time is split according to the ticket distribution.

```
m_i = tickets of job i
M   = Σ m_i (total tickets)
P(i) = m_i / M   = probability that job i is scheduled
```

**Policy:**

- **More tickets to short jobs**, few to long jobs → **simulates SJF**.
- **Every job gets at least one ticket** → **no starvation**.
- **Degrades gracefully** as load changes: adding or removing a job affects all others **proportionally**.

**The key difference:** lottery is the only **randomized** scheduler; all the others are **deterministic**.

**Example:** short jobs get 10 tickets, long jobs get 1.

| # short / # long | % CPU per short job | % CPU per long job |
| --- | --- | --- |
| 1 / 1 | 10/11 ≈ 91% | 1/11 ≈ 9% |
| 0 / 2 | — | 1/2 = 50% |
| 2 / 0 | 10/20 = 50% | — |
| 10 / 1 | 10/101 ≈ 9.9% | 1/101 ≈ 0.99% |
| 1 / 10 | 10/20 = 50% | 1/20 = 5% |

---

## 12. Summary of algorithms

| Algorithm | Preemptive? | Strength | Weakness |
| --- | --- | --- | --- |
| **FCFS** | No | Very simple | Generally unfair; highly variable waiting time; convoy effect |
| **RR** | Yes (timer) | **Fair** | Average waiting time may be long; sensitive to the quantum |
| **SJF / SRTF** | SJF no, SRTF yes | **Optimal average waiting time** (if bursts are predicted correctly) | Unfair; possible **starvation**; bursts hard to predict |
| **Priority** | Either | General | Starvation (fixed by **aging**) |
| **MLQ / MLFQ** | — | Approximates SJF; MLFQ is very flexible | Complex; may be unfair |
| **Lottery** | — | Fairer, low average waiting time, no starvation | Less predictable (randomized) |

---

## 13. Worked example: one job set, four algorithms

Not from the slides: a practice set in exam style, **with arrival times**.

| Job | Arrival | Burst |
| --- | --- | --- |
| P1 | 0 | 5 |
| P2 | 1 | 3 |
| P3 | 2 | 7 |
| P4 | 4 | 2 |

### FCFS

```
| P1        | P2    | P3            | P4  |
0           5       8               15    17
```

| Job | Completion | Turnaround | Waiting |
| --- | --- | --- | --- |
| P1 | 5 | 5 | 0 |
| P2 | 8 | 7 | 4 |
| P3 | 15 | 13 | 6 |
| P4 | 17 | 13 | 11 |
| **Avg** | | **9.5** | **5.25** |

### SJF (non-preemptive)

At t = 0 only P1 is there, so it runs to 5. At t = 5, ready: P2 (3), P3 (7), P4 (2) → P4, then P2, then P3.

```
| P1        | P4  | P2    | P3            |
0           5     7       10              17
```

| Job | Completion | Turnaround | Waiting |
| --- | --- | --- | --- |
| P1 | 5 | 5 | 0 |
| P2 | 10 | 9 | 6 |
| P3 | 17 | 15 | 8 |
| P4 | 7 | 3 | 1 |
| **Avg** | | **8** | **3.75** |

### SRTF (preemptive)

- t = 1: P2 (3) < P1's remaining (4) → P2 preempts.
- t = 2: P3 (7) > P2's remaining (2) → P2 continues; done at 4.
- t = 4: P1 (4), P3 (7), P4 (2) → P4 runs to 6.
- t = 6: P1 (4) < P3 (7) → P1 runs to 10. Then P3 to 17.

```
| P1 | P2    | P4  | P1      | P3            |
0    1       4     6         10              17
```

| Job | Completion | Turnaround | Waiting |
| --- | --- | --- | --- |
| P1 | 10 | 10 | 5 |
| P2 | 4 | 3 | 0 |
| P3 | 17 | 15 | 8 |
| P4 | 6 | 2 | 0 |
| **Avg** | | **7.5** | **3.25** |

### Round Robin (quantum = 3)

- t = 0–3: P1 (2 left). P2 and P3 arrived meanwhile → queue: P2, P3, P1.
- t = 3–6: P2 done at 6. P4 arrived at 4 → queue: P3, P1, P4.
- t = 6–9: P3 (4 left) → queue: P1, P4, P3.
- t = 9–11: P1 done. t = 11–13: P4 done. t = 13–16: P3 (1 left). t = 16–17: P3 done.

```
| P1    | P2    | P3    | P1  | P4  | P3    | P3 |
0       3       6       9     11    13      16   17
```

| Job | Completion | Turnaround | Waiting |
| --- | --- | --- | --- |
| P1 | 11 | 11 | 6 |
| P2 | 6 | 5 | 2 |
| P3 | 17 | 15 | 8 |
| P4 | 13 | 9 | 7 |
| **Avg** | | **10** | **5.75** |

### Comparison

| | FCFS | SJF | SRTF | RR (q = 3) |
| --- | --- | --- | --- | --- |
| Avg turnaround | 9.5 | 8 | **7.5** | 10 |
| Avg waiting | 5.25 | 3.75 | **3.25** | 5.75 |

SRTF is best on averages (optimal among these); RR is worst on averages but fairest in response.

---

## 14. Exam traps (Lectures 5–6)

- **Turnaround = completion − arrival**, not completion alone (unless arrival = 0).
- **Waiting = turnaround − burst**; I/O time is not counted.
- Waiting **time** = time in the **ready** queue, not time in the waiting **state**.
- Non-preemptive schedulers still run on **running → waiting** and on **termination**.
- Preemptive schedulers also run on **running → ready** (interrupt) and **waiting → ready**.
- Preemption (via the timer) puts an **upper bound** on how long a process holds the CPU continuously.
- FCFS is **non-preemptive**: a CPU-bound job with no I/O keeps the CPU until it finishes.
- In FCFS with I/O, a job returning from I/O does **not** preempt the running job.
- RR with a **huge quantum** = FCFS. With a **tiny quantum**, context switches dominate.
- Scheduler **decides**; dispatcher **switches context, switches to user mode, jumps** to the program.
- **SJF** is optimal for **average waiting time**, but needs burst prediction and can starve long jobs.
- **SRTF** = preemptive SJF: preempt only when a **new arrival** has a shorter burst than the **remaining** time.
- Exponential smoothing: α = 0 ignores observations; α = 1 uses only the last burst.
- Priority scheduling: starvation → **aging**. Usually **low number = high priority**.
- MLQ: jobs **can't** change queues. MLFQ: jobs **can**.
- MLFQ: use the whole slice → **demoted**; block before it expires → **promoted**.
- Lottery: the only **randomized** scheduler; each job gets ≥ 1 ticket so **no starvation**.

---

## 15. Practice questions (Rounds 1–4)

### Round 1: Basics, metrics, preemption

1. Under which of the 4 scheduling conditions does a **non-preemptive** scheduler make a decision?
2. A process arrives at t = 2, has a burst of 4, and completes at t = 10. What are its turnaround and waiting times?
3. A process is blocked waiting for the disk. Does that time count in its "waiting time" metric?
4. Name the 3 functions of the dispatcher.
5. Which metrics do we maximize, and which do we minimize?

<details>
<summary>Answers (Round 1)</summary>

1. **Conditions 1 and 4**: running → waiting, and process termination (creation too).
2. Turnaround = 10 − 2 = **8**; waiting = 8 − 4 = **4**.
3. **No.** The waiting-time metric counts time in the **ready** queue; I/O time is excluded.
4. **Switching context**, **switching to user mode**, **jumping to the proper location** in the program.
5. Maximize **CPU utilization** and **throughput**; minimize **turnaround**, **waiting** and **response** time.

</details>

### Round 2: FCFS and RR

1. Jobs A = 6, B = 3, C = 1, all arriving at 0 in that order. Average waiting time under FCFS?
2. Same jobs in order C, B, A under FCFS?
3. Same jobs (order A, B, C), RR with quantum = 2. Draw the Gantt chart and compute the average waiting time.
4. What happens to RR when the quantum is very large? Very small?
5. What is the convoy effect?

<details>
<summary>Answers (Round 2)</summary>

1. Waiting 0, 6, 9 → **15 / 3 = 5**.
2. Waiting 0, 1, 4 → **5 / 3 ≈ 1.67**.
3. `A 0–2 | B 2–4 | C 4–5 | A 5–7 | B 7–8 | A 8–10`. Completions: A 10, B 8, C 5. Waiting: A 4, B 5, C 4 → **13 / 3 ≈ 4.33**.
4. Very large → behaves like **FCFS**. Very small → **too many context switches**, low throughput.
5. Short or I/O-bound jobs get stuck waiting behind a long CPU-bound job, so CPU and I/O overlap poorly.

</details>

### Round 3: SJF, SRTF, smoothing, priority

1. Jobs with bursts 6, 2, 8, 3, all arriving at 0. Average waiting time under SJF?
2. α = 0.5, previous prediction = 10, last actual burst = 6. What is the next prediction?
3. What makes SRTF preempt the running process?
4. Which jobs can starve under SJF, and why?
5. What is aging?

<details>
<summary>Answers (Round 3)</summary>

1. Order 2, 3, 6, 8 → waiting 0, 2, 5, 11 → **18 / 4 = 4.5**.
2. `0.5 · 6 + 0.5 · 10 =` **8**.
3. A **new arrival** whose burst is **shorter than the remaining** time of the running process.
4. **Long CPU-bound** jobs: short jobs keep getting picked first.
5. **Raising a job's priority the longer it waits**, so it eventually runs (fixes starvation).

</details>

### Round 4: MLQ, MLFQ, lottery

1. In MLFQ, what happens to a job that uses its whole time slice? And to one that blocks for I/O first?
2. What is the main difference between MLQ and MLFQ?
3. Three jobs have 5, 3 and 2 tickets. What is the probability that the first is scheduled in a given slice?
4. 2 short jobs (10 tickets each) and 2 long jobs (1 ticket each). What % of CPU does each get?
5. Which scheduling algorithm is randomized?

<details>
<summary>Answers (Round 4)</summary>

1. Whole slice → **demoted** one level. Blocks first → **promoted** one level.
2. In MLQ jobs **can't** move between queues; in MLFQ they **can**.
3. `5 / 10 =` **50%**.
4. Total = 22. Each short job: 10/22 ≈ **45.5%**; each long job: 1/22 ≈ **4.5%**.
5. **Lottery scheduling.**

</details>

---

## 16. Exam-style questions

> **Reconstructed** in exam format from the scheduling topics that appeared in past exams and in the question bank (when the scheduler runs, preemption, RR/FCFS/SJF/SRTF calculations, turnaround vs completion, comparing algorithms, starvation). They are not the verbatim exam text. For RR, the tie convention in section 5 applies.

1. A non-preemptive CPU scheduler makes a scheduling decision:
   (a) only when a process terminates
   (b) when a process switches from running to waiting, or terminates
   (c) at every timer interrupt
   (d) never

2. Preemptive scheduling ensures that:
   (a) every process finishes in a fixed time
   (b) there is an upper bound on how long a process can keep the CPU continuously
   (c) no context switch ever happens
   (d) I/O-bound processes are always starved

3. Round Robin, quantum = 3. Compute the average waiting time.

   | Job | Arrival | Burst |
   | --- | --- | --- |
   | A | 0 | 4 |
   | B | 1 | 6 |
   | C | 2 | 2 |
   | D | 5 | 5 |

   (a) 4.5 (b) 5.75 (c) 6 (d) 7

4. FCFS, all jobs arrive at 0 in order A, B, C, with bursts 8, 4, 2. Average waiting time?
   (a) 2.67 (b) 4.67 (c) 6.67 (d) 8

5. Same jobs as question 4 (bursts 8, 4, 2, all arrive at 0), scheduled with SJF. Average turnaround time?
   (a) 7.33 (b) 8.67 (c) 11.33 (d) 14

6. SRTF. Compute the average waiting time.

   | Job | Arrival | Burst |
   | --- | --- | --- |
   | A | 0 | 7 |
   | B | 2 | 4 |
   | C | 4 | 1 |
   | D | 5 | 4 |

   (a) 3 (b) 4 (c) 6.5 (d) 8.75

7. A process arrives at t = 3, needs 5 units of CPU and completes at t = 15. Its turnaround and waiting times are:
   (a) 15 and 10 (b) 12 and 7 (c) 12 and 5 (d) 10 and 7

8. Which algorithm is provably optimal for minimizing the average waiting time (given correct burst predictions)?
   (a) FCFS (b) RR (c) SJF (d) Lottery

9. In priority scheduling, starvation of low-priority processes is solved by:
   (a) disabling interrupts (b) aging (c) a larger time quantum (d) the dispatcher

10. Round Robin with a very large time quantum behaves like:
    (a) SJF (b) FCFS (c) SRTF (d) priority scheduling

11. The "waiting time" scheduling metric measures:
    (a) the time a process spends in the waiting state
    (b) the time a process spends in the ready queue
    (c) the total time from arrival to completion
    (d) the time spent doing I/O

12. Under FCFS, a CPU-bound process that never performs I/O:
    (a) is preempted at every timer interrupt
    (b) keeps the CPU until it finishes
    (c) is moved to a lower-priority queue
    (d) runs for one quantum, then goes to the back of the queue

13. In an MLFQ, a job that blocks for I/O before its time slice expires:
    (a) is demoted one level
    (b) is promoted one level
    (c) is removed from the system
    (d) stays in its queue forever

<details>
<summary>Answers (Exam-style)</summary>

1. **(b)**
2. **(b)**
3. **(c) 6.** Gantt: `A 0–3 | B 3–6 | C 6–8 | A 8–9 | D 9–12 | B 12–15 | D 15–17`. Completions: A 9, B 15, C 8, D 17. Waiting: A 9−0−4 = 5, B 15−1−6 = 8, C 8−2−2 = 4, D 17−5−5 = 7 → 24 / 4 = **6**.
4. **(c)** Waiting 0, 8, 12 → 20 / 3 ≈ **6.67**.
5. **(a)** Order C, B, A → completions 2, 6, 14 → 22 / 3 ≈ **7.33**.
6. **(a) 3.** Gantt: `A 0–2 | B 2–4 | C 4–5 | B 5–7 | D 7–11 | A 11–16`. Waiting: A 16−0−7 = 9, B 7−2−4 = 1, C 5−4−1 = 0, D 11−5−4 = 2 → 12 / 4 = **3**.
7. **(b)** Turnaround = 15 − 3 = 12; waiting = 12 − 5 = 7.
8. **(c)**
9. **(b)**
10. **(b)**
11. **(b)**
12. **(b)**
13. **(b)**

</details>

---

## 17. Oral exam questions

Open questions on these lectures from the oral exam list. Answer them out loud using the sections above.

1. What is the difference between preemptive and non-preemptive scheduling? (The scheduler intervenes on running → waiting in **both** cases.)
2. Why do we need scheduling at all?
3. How does Round Robin work?
4. How does FCFS work?
5. How does SRTF work?
6. Which metrics can we use to compare scheduling algorithms? What are the pros and cons of each algorithm? *(a big question: prepare a full answer)*
7. How can we avoid starvation in priority scheduling? (Aging.)
