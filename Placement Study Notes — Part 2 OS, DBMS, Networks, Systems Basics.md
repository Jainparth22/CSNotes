# Placement Study Notes — Part 2: OS, DBMS, Computer Networks and Systems Basics

*Companion to Part 1 (DSA). Code is given in C++17 and Python 3 wherever code helps.*

## 0. How to use these notes

Every topic is written for two uses: **learning** (read top to bottom once) and **revision** (jump to the *Interview answer* and *Loopholes* lines).

- **Definition**: the one-sentence answer you should be able to say cold.
- **Key points**: what sits underneath the definition, with diagrams.
- **Worked example**: numbers you can reproduce on paper (these are what OAs and written rounds ask).
- **Interview answer**: the concise spoken version.
- **Loopholes**: common wrong statements, edge cases and follow-up traps.

**Contents:** Part A Operating Systems (A1 basics, A2 process vs thread, A3 context switching, A4 CPU scheduling, A5 synchronization and concurrency, A6 deadlocks, A7 memory management and virtual memory, A8 page replacement and thrashing, A8b system calls, A9 thread safety, CAS and TOCTOU) · Part B DBMS (B1 basics, B2 ACID and transactions, B3 keys and constraints, B4 normalization, B5 ER diagrams, B6 indexing and B-trees, B7 isolation levels, B8 serializability, B9 transaction deadlocks, B10 query processing and optimization, B11 SQL vs NoSQL) · Part C Computer Networks (C1 models, C2 TCP vs UDP, C3 IP and subnetting, C4 ARP and DHCP, C5 DNS, C6 HTTP and HTTPS, C7 TLS, C8 status codes, C9 HTTP/2 and HTTP/3, C10 REST, C11 cookies vs sessions vs JWT, C12 what happens when you type a URL) · Part D Programming and systems basics (C++ pointers and references, RAII, smart pointers, STL, Linux, compilers, computer architecture) · Part E Rapid-fire revision sheet.

---

# PART A: OPERATING SYSTEMS

## A1. Operating-system basics

**Definition:** an operating system is system software that manages hardware (CPU, memory, disk, I/O) and gives programs a safe, uniform interface to it. It does two jobs: **resource manager** (fair, efficient sharing) and **abstraction layer** (files instead of disk blocks, processes instead of raw CPU).

```
+-----------------------------------------+
|  Applications (browser, IDE, games)     |   user mode
+-----------------------------------------+
|  Libraries (libc, STL)                  |
+=========================================+  <- system call interface (trap)
|  KERNEL: process mgmt, memory mgmt,     |   kernel mode
|  file system, device drivers, networking|
+-----------------------------------------+
|  Hardware: CPU, RAM, disk, NIC          |
+-----------------------------------------+
```

**Key points**

- **Dual mode (user/kernel):** a mode bit in the CPU. Privileged instructions (I/O, changing page tables, halting the CPU) run only in kernel mode. A user program asks the kernel for service via a **system call**, which raises a software **trap** and switches mode.
- **Kernel types:** *monolithic* (everything in kernel space: fast, one bug can crash all; Linux), *microkernel* (only minimal services in the kernel, the rest as user processes: safer, more IPC overhead; Minix, QNX), *hybrid* (Windows NT, macOS).
- **Interrupt vs trap vs exception:** an **interrupt** is asynchronous, from hardware (timer, keyboard, disk completion). A **trap/exception** is synchronous, caused by the running instruction (system call, divide by zero, page fault). The CPU saves state, jumps to a handler through the interrupt vector table, then resumes.
- **Boot sequence:** firmware (BIOS/UEFI) → bootloader (GRUB) → kernel loads → init/systemd (PID 1) → services and login.
- **OS types:** batch, multiprogramming, time-sharing (preemptive, short quantum), real-time (hard: deadlines must be met; soft: should), distributed, embedded.
- **Multiprogramming vs multitasking vs multiprocessing:** several programs in memory so the CPU never idles on I/O; rapid switching so each user feels continuous service; more than one physical CPU/core running in parallel.

**Interview answer:** "The OS is the resource manager and abstraction layer between hardware and programs. It runs in kernel mode; user programs enter it only through system calls."

**Loopholes**

- The **shell** and GUI are not the OS; they are user programs. The kernel is the core of the OS.
- Concurrency (interleaving on one core) is not parallelism (simultaneous execution on several cores).
- Interrupts are asynchronous; system calls and page faults are synchronous.

---

## A2. Process vs thread

**Process:** a program in execution, with its own address space. **Thread:** the unit of CPU execution *inside* a process; threads of one process share the address space.

```
Process address space (high to low)         Process Control Block (PCB)
+----------------+                          - PID, parent PID
|  Stack         | grows down (locals,      - state, program counter, registers
|  ...           |  return addresses)       - scheduling info, priority
|  Heap          | grows up (malloc/new)    - page table pointer / memory limits
|  .bss          | uninitialised globals    - open file table, I/O status
|  .data         | initialised globals
|  .text (code)  | read-only
+----------------+
```

**Process state diagram**

```
 New --admit--> Ready --dispatch--> Running --exit--> Terminated
                  ^                  |   |
                  +---- timeout -----+   +--I/O or event wait--> Waiting
                  ^                                               |
                  +------------ I/O or event complete ------------+
```

**Process vs thread**

|  | Process | Thread |
| --- | --- | --- |
| Memory | own address space | shares code, data, heap, open files with its process |
| Own resources | everything | stack, registers, program counter, thread-local storage |
| Creation cost | heavy (copy page tables, PCB) | light |
| Switch cost | high (switch page tables, TLB flush) | low (same address space) |
| Communication | IPC (pipes, sockets, shared memory, message queues) | shared variables (needs synchronization) |
| Isolation | a crash affects only that process | a crash or memory corruption can bring down the whole process |

**Threading models:** *user-level* threads (library-managed; cheap, but one blocking call blocks all, no multicore use), *kernel-level* threads (OS schedules each; real parallelism), and mappings **1:1** (Linux, Windows), **M:1**, **M:N** (Go goroutines).

**IPC mechanisms:** pipe/named pipe (FIFO stream), message queue, **shared memory** (fastest, needs synchronization), socket (also across machines), signal (small asynchronous notification), memory-mapped file.

**`fork()` worked example**

```cpp
#include <unistd.h>
#include <sys/wait.h>
#include <cstdio>
int main() {
    int x = 10;
    pid_t pid = fork();                 // returns 0 in the child, child PID in the parent, -1 on failure
    if (pid == 0) { x += 5; printf("child x=%d\n", x); }      // child has its OWN copy of x
    else { wait(nullptr); printf("parent x=%d\n", x); }       // parent still prints 10
}
```

Output: `child x=15` then `parent x=10`. Memory is **copy-on-write**: the pages are shared until one side writes.

**Counting question:** `fork()` called `n` times in a row (no conditions) creates `2^n - 1` new processes (`2^n` in total). Three forks give 8 processes.

**Zombie vs orphan:** a **zombie** has finished but its parent has not called `wait()`, so the PCB entry remains. An **orphan** is a running child whose parent died; it is adopted by `init` (PID 1). `exec()` replaces the process image with a new program; `fork + exec` is how shells launch programs.

**Interview answer:** "A process is an isolated program instance with its own address space. Threads share the process's memory but have separate stacks and registers, so they are cheaper to create and switch, but need synchronization and give no isolation."

**Loopholes**

- Threads share the **heap and globals** but not the **stack**; local variables are private.
- A segmentation fault in one thread kills every thread in the process.
- `fork()` in a multithreaded program only duplicates the calling thread: a classic source of deadlocks.
- Zombies cannot be killed (they are already dead); the parent must reap them.
- Python's GIL means CPU-bound Python threads do not run in parallel; use multiprocessing for CPU work.

---

## A3. Context switching

**Definition:** saving the state of the running process/thread and restoring the state of the next one, so the CPU can switch between them.

```
Process P1 running -> interrupt/syscall -> kernel saves P1 registers + PC into PCB1
                   -> scheduler picks P2 -> kernel loads P2 registers + PC from PCB2
                   -> P2 running
```

**What is saved:** program counter, CPU registers, stack pointer, process state, memory-management info (page-table base). **Cost:** the switch itself (microseconds) is *pure overhead*; the larger hidden cost is **cache and TLB pollution** because the new process's data is not in the CPU cache, and a process switch flushes (or tags) the TLB. Thread switches inside one process avoid the page-table switch and are cheaper.

**Triggers:** time slice expiry (timer interrupt), blocking I/O, higher-priority process becomes ready, voluntary yield, process exit.

**Mode switch vs context switch:** a system call switches user to kernel mode in the *same* process (cheap, no process change). A context switch changes *which* process runs.

**Loopholes:** more frequent switches (very small quantum) mean more overhead; context switching is overhead, not useful work; user-level thread switches need no kernel call.

---

## A4. CPU scheduling and algorithms

**Terms (memorize these formulas)**

- **Arrival time (AT)**, **Burst time (BT)**, **Completion time (CT)**.
- **Turnaround time (TAT) = CT - AT**
- **Waiting time (WT) = TAT - BT**
- **Response time = first time on CPU - AT**
- **Throughput** = jobs finished per unit time. **CPU utilization** = fraction of time CPU is busy.
- **Preemptive:** the OS can take the CPU away. **Non-preemptive:** a process keeps the CPU until it finishes or blocks.

| Algorithm | Preemptive? | Idea | Pros | Cons |
| --- | --- | --- | --- | --- |
| FCFS | No | First come first served (FIFO queue) | simple, fair by arrival | **convoy effect** (short jobs wait behind a long one) |
| SJF | No | Shortest burst first | minimum average waiting time (provably optimal) | needs burst length in advance; **starvation** of long jobs |
| SRTF | Yes | Preemptive SJF: run the job with the least *remaining* time | best average waiting | starvation, overhead |
| Priority | Both | Highest priority first | models importance | **starvation**, fix with **aging** (raise priority over time) |
| Round Robin (RR) | Yes | Each process gets a time quantum `q`, then goes to the back | fair, good response time | high overhead if `q` too small; becomes FCFS if `q` too large |
| MLQ | Yes | Separate queues per class (system, interactive, batch) with fixed priority between queues | policy per class | rigid, starvation of lower queues |
| MLFQ | Yes | Processes move between queues based on behaviour (CPU hogs sink, I/O-bound rise) | adaptive, no burst knowledge needed | complex to tune |

**SJF burst prediction:** exponential averaging, `tau_next = alpha * t_actual + (1 - alpha) * tau_prev`.

**Worked example 1: FCFS vs non-preemptive SJF** (P1: AT 0, BT 5; P2: AT 1, BT 3; P3: AT 2, BT 8; P4: AT 3, BT 6)

```
FCFS Gantt:  | P1 0-5 | P2 5-8 | P3 8-16 | P4 16-22 |
CT:  P1=5  P2=8  P3=16  P4=22
TAT = CT - AT:  5, 7, 14, 19
WT  = TAT - BT: 0, 4, 6, 13      average WT = 23 / 4 = 5.75

SJF (non-preemptive): at t=0 only P1 is available, it runs to 5.
At t=5 ready: P2(3), P3(8), P4(6) -> pick P2, then P4, then P3.
| P1 0-5 | P2 5-8 | P4 8-14 | P3 14-22 |
TAT: P1=5, P2=7, P4=14-3=11, P3=22-2=20
WT : P1=0, P2=4, P4=5, P3=12                average WT = 21 / 4 = 5.25
```

**Worked example 2: Round Robin, q = 2** (P1 BT 5, P2 BT 3, P3 BT 1, all AT 0)

```
Queue evolution:
| P1 0-2 | P2 2-4 | P3 4-5 | P1 5-7 | P2 7-8 | P1 8-9 |
CT: P3=5, P2=8, P1=9   (AT = 0, so TAT = CT)
WT = TAT - BT: P1 = 9-5 = 4, P2 = 8-3 = 5, P3 = 5-1 = 4      average WT = 13/3 = 4.33
Response time: P1=0, P2=2, P3=4
```

**Round Robin simulation in C++ (all arrivals at 0)**

```cpp
#include <bits/stdc++.h>
using namespace std;
int main() {
    vector<int> bt = {5, 3, 1}; int q = 2, n = bt.size();
    vector<int> rem = bt, ct(n, 0); int t = 0;
    queue<int> rq; for (int i = 0; i < n; i++) rq.push(i);
    while (!rq.empty()) {
        int i = rq.front(); rq.pop();
        int run = min(q, rem[i]);
        t += run; rem[i] -= run;
        if (rem[i] > 0) rq.push(i); else ct[i] = t;
    }
    for (int i = 0; i < n; i++) printf("P%d CT=%d TAT=%d WT=%d\n", i + 1, ct[i], ct[i], ct[i] - bt[i]);
}
```

```python
from collections import deque
def round_robin(bt, q):                # all arrive at time 0
    rem, t, ct, dq = bt[:], 0, [0] * len(bt), deque(range(len(bt)))
    while dq:
        i = dq.popleft(); run = min(q, rem[i]); t += run; rem[i] -= run
        if rem[i]: dq.append(i)
        else: ct[i] = t
    return [(c, c - b) for c, b in zip(ct, bt)]    # (TAT, WT) per process
```

**Convoy effect:** one CPU-bound long job at the head of FCFS holds back many short I/O-bound jobs, so I/O devices sit idle.

**Interview answer:** "For minimum average waiting time use SJF/SRTF, but they need burst times and can starve long jobs. Round Robin gives fair, responsive time-sharing; choose the quantum large enough that most bursts finish in one quantum yet small enough for good response. Aging solves starvation in priority scheduling."

**Loopholes**

- When arrival times differ, **only consider processes that have already arrived**; picking the global minimum burst is a classic mistake.
- In RR, a newly arrived process is queued **before** the preempted process that just used its quantum (convention; state your assumption).
- Idle CPU gaps: if no process is ready, time jumps to the next arrival.
- SJF is optimal only for average waiting time in the non-preemptive setting with known bursts; it does not minimize response time.
- Priority inversion (see A5) is a scheduling problem that can occur even without starvation.

---

## A5. Concurrency and synchronization

**Race condition:** the result depends on the unlucky interleaving of threads that access shared data without coordination. The textbook case is `counter++`, which is **three** machine steps (load, add, store), not one:

```
Thread A: load counter (5)
Thread B: load counter (5)
Thread A: add 1 -> 6, store 6
Thread B: add 1 -> 6, store 6      <- one increment lost (expected 7)
```

**Critical section problem:** code that touches shared data must satisfy three requirements.

1. **Mutual exclusion:** at most one thread in the critical section.
2. **Progress:** if the section is free and someone wants in, only the waiting threads decide who goes next, with no indefinite postponement.
3. **Bounded waiting:** a limit on how many times others can enter before a waiting thread gets in (no starvation).

**Peterson's solution (two threads, software only)**

```cpp
bool flag[2] = {false, false}; int turn;
void enter(int i) {                 // i is 0 or 1, j is the other thread
    int j = 1 - i;
    flag[i] = true;                 // I want to enter
    turn = j;                       // but you go first if you also want
    while (flag[j] && turn == j) {} // busy wait
}
void leave(int i) { flag[i] = false; }
```

It satisfies all three requirements, but on modern CPUs with instruction reordering it needs memory barriers, so real code uses hardware atomics.

**Hardware support:** `test-and-set`, `compare-and-swap (CAS)`, atomic instructions; a **spinlock** is built from them. Disabling interrupts works only on a single core.

### Mutex vs semaphore

|  | Mutex | Semaphore |
| --- | --- | --- |
| Purpose | **locking**: mutual exclusion | **signalling** and counting resources |
| Value | locked/unlocked | integer counter |
| Ownership | the thread that locks **must** unlock | any thread may `signal` |
| Types | recursive, timed, spin | binary, counting |
| Typical use | protect a shared variable | limit N concurrent users, producer-consumer, ordering of events |

`wait(S)` (P/down): `S--`, block if `S < 0`. `signal(S)` (V/up): `S++`, wake one waiter. A **binary semaphore** looks like a mutex but has no ownership, so it can be released by a different thread; this is useful for signalling and dangerous for locking.

**Spinlock vs blocking lock:** a spinlock busy-waits (good when the critical section is shorter than a context switch, multicore only); a blocking lock puts the thread to sleep.

### Monitors and condition variables

A **monitor** bundles shared data, the operations on it, and an implicit lock so that only one thread is inside at a time; **condition variables** (`wait`, `notify_one`, `notify_all`) let a thread sleep until some condition on the data holds. Rule: **always wait in a `while` loop** (spurious wakeups, and another thread may consume the condition first), and wait with the mutex held (the wait releases it atomically).

### Code: the lost-update fix in C++ and Python

```cpp
#include <bits/stdc++.h>
using namespace std;
int plain = 0; atomic<int> at{0}; int locked = 0; mutex m;
void work() {
    for (int i = 0; i < 100000; i++) {
        plain++;                                // data race (undefined behaviour)
        at.fetch_add(1);                        // atomic: correct, lock-free
        { lock_guard<mutex> g(m); locked++; }   // mutex with RAII: unlocks even on exceptions
    }
}
int main() {
    thread t1(work), t2(work); t1.join(); t2.join();
    cout << plain << ' ' << at << ' ' << locked << '\n';   // plain is usually < 200000
}
```

```python
import threading
count, lock = 0, threading.Lock()
def work():
    global count
    for _ in range(100000):
        with lock:                 # 'count += 1' is not atomic in Python either
            count += 1
ts = [threading.Thread(target=work) for _ in range(2)]
[t.start() for t in ts]; [t.join() for t in ts]
print(count)                       # 200000
```

### Classic problem 1: bounded buffer (producer-consumer)

```cpp
template <class T>
class BoundedQueue {
    queue<T> q; size_t cap; mutex m; condition_variable notFull, notEmpty;
public:
    explicit BoundedQueue(size_t c) : cap(c) {}
    void push(T v) {
        unique_lock<mutex> lk(m);
        notFull.wait(lk, [&] { return q.size() < cap; });     // predicate form = while loop
        q.push(std::move(v));
        notEmpty.notify_one();
    }
    T pop() {
        unique_lock<mutex> lk(m);
        notEmpty.wait(lk, [&] { return !q.empty(); });
        T v = std::move(q.front()); q.pop();
        notFull.notify_one();
        return v;
    }
};
```

Semaphore version: `empty = N`, `full = 0`, `mutex = 1`. Producer: `wait(empty); wait(mutex); add; signal(mutex); signal(full)`. Consumer: `wait(full); wait(mutex); remove; signal(mutex); signal(empty)`. **Swapping the first two waits in the producer deadlocks** (holds the mutex while waiting for space).

### Classic problem 2: readers-writers

Many readers may read together, a writer needs exclusive access. *Readers-preference* can starve writers; *writers-preference* can starve readers. C++17 offers `shared_mutex` (`shared_lock` for readers, `unique_lock` for writers).

### Classic problem 3: dining philosophers

Five philosophers, five forks; each needs two. If everyone grabs the left fork, **deadlock**. Fixes: allow at most 4 to sit; pick up both forks atomically; **resource ordering** (always take the lower-numbered fork first, breaking circular wait); asymmetric (odd philosophers go left-first, even right-first).

### Priority inversion

A low-priority task holds a lock needed by a high-priority task, while a medium-priority task preempts the low one, so the high one is delayed indefinitely (Mars Pathfinder, 1997). Fix: **priority inheritance** (the lock holder temporarily inherits the waiter's priority).

**Interview answer:** "A mutex enforces mutual exclusion and has an owner; a semaphore is a counter used for signalling and for limiting access to N resources. Use condition variables with a predicate loop for waiting on state."

**Loopholes**

- "A mutex is just a binary semaphore" is the common wrong answer: ownership and intended use differ.
- Forgetting to unlock on an early `return`/exception: use `lock_guard`/`with`.
- Locking in different orders in different places creates deadlock; lock multiple mutexes with `scoped_lock(m1, m2)` (C++17) or a global order.
- Notifying without changing the condition under the lock can lose the wakeup.
- Double-checked locking without atomics is broken.
- `volatile` does not make code thread-safe in C++ (it is not a synchronization primitive).
- Spinning on a single core wastes the whole time slice.

---

## A6. Deadlocks: conditions, prevention, avoidance, detection

**Definition:** a set of processes where each holds a resource and waits for one held by another in the set, so none can proceed.

**Four necessary (Coffman) conditions, all must hold:**

1. **Mutual exclusion:** a resource is held by one process at a time.
2. **Hold and wait:** a process holds one resource while waiting for another.
3. **No preemption:** resources cannot be forcibly taken away.
4. **Circular wait:** a cycle of processes each waiting for the next.

```
Resource allocation graph (single-instance resources): a cycle means deadlock
   P1 --requests--> R2 --held by--> P2 --requests--> R1 --held by--> P1
```

| Strategy | Idea | How |
| --- | --- | --- |
| **Prevention** | break at least one Coffman condition | request all resources at once (no hold-and-wait); allow preemption; impose a **global resource ordering** (no circular wait) |
| **Avoidance** | grant a request only if the system stays in a **safe state** | **Banker's algorithm** (needs max demands in advance) |
| **Detection and recovery** | allow deadlock, then detect it | wait-for graph cycle detection; recover by killing processes or preempting resources / rollback |
| **Ignore (ostrich)** | assume it is rare | Unix/Linux/Windows desktops |

**Safe state:** there exists a sequence in which every process can finish. Unsafe does **not** mean deadlocked; it means deadlock is possible.

**Banker's algorithm: worked example (3 resource types A, B, C)**

```
Available = (3, 3, 2)
Process  Allocation   Max        Need = Max - Allocation
P0       0 1 0        7 5 3      7 4 3
P1       2 0 0        3 2 2      1 2 2
P2       3 0 2        9 0 2      6 0 0
P3       2 1 1        2 2 2      0 1 1
P4       0 0 2        4 3 3      4 3 1

Work = (3,3,2)
P1: need (1,2,2) <= work  -> finishes, work += alloc(2,0,0) = (5,3,2)
P3: need (0,1,1) <= work  -> finishes, work += (2,1,1)       = (7,4,3)
P4: need (4,3,1) <= work  -> finishes, work += (0,0,2)       = (7,4,5)
P0: need (7,4,3) <= work  -> finishes, work += (0,1,0)       = (7,5,5)
P2: need (6,0,0) <= work  -> finishes, work += (3,0,2)       = (10,5,7)
Safe sequence: <P1, P3, P4, P0, P2>   -> SAFE
```

**Resource-request algorithm:** when Pi requests `Req`, check `Req <= Need[i]` and `Req <= Available`; pretend to grant it (`Available -= Req`, `Alloc += Req`, `Need -= Req`), run the safety check; if unsafe, roll back and make the process wait.

```cpp
// Banker's safety algorithm (returns safe sequence or empty vector)
vector<int> safeSequence(vector<int> avail, vector<vector<int>> alloc, vector<vector<int>> need) {
    int n = alloc.size(), m = avail.size();
    vector<char> done(n, 0); vector<int> seq;
    while ((int)seq.size() < n) {
        bool progressed = false;
        for (int i = 0; i < n; i++) {
            if (done[i]) continue;
            bool ok = true;
            for (int j = 0; j < m; j++) if (need[i][j] > avail[j]) { ok = false; break; }
            if (!ok) continue;
            for (int j = 0; j < m; j++) avail[j] += alloc[i][j];   // process finishes, releases everything
            done[i] = 1; seq.push_back(i); progressed = true;
        }
        if (!progressed) return {};                               // unsafe
    }
    return seq;
}
```

```python
def safe_sequence(avail, alloc, need):
    n, work, done, seq = len(alloc), avail[:], [False] * len(alloc), []
    while len(seq) < n:
        for i in range(n):
            if not done[i] and all(need[i][j] <= work[j] for j in range(len(work))):
                work = [w + a for w, a in zip(work, alloc[i])]
                done[i] = True; seq.append(i); break
        else:
            return []          # no process could proceed: unsafe
    return seq
```

**Detection:** for single-instance resources, build a **wait-for graph** (edge Pi to Pj if Pi waits for Pj) and look for a cycle (DFS, same as directed cycle detection in Part 1). For multiple instances, use a Banker-style detection algorithm.

**Deadlock vs starvation vs livelock:** deadlock = nobody moves, blocked forever; starvation = one process is repeatedly passed over; livelock = processes keep changing state in response to each other but make no progress (two people stepping aside in a corridor).

**Interview answer:** "Deadlock needs mutual exclusion, hold-and-wait, no preemption and circular wait. Break any one to prevent it; the practical technique is a global lock ordering. Banker's algorithm avoids it by granting requests only if a safe sequence remains."

**Loopholes**

- All **four** conditions must hold simultaneously; removing one is enough.
- A cycle in the resource-allocation graph means deadlock **only** when each resource has a single instance; with multiple instances a cycle is necessary but not sufficient.
- Banker's needs advance knowledge of maximum demands, which is unrealistic in general-purpose OSes.
- Deadlock can occur in application code (locks) and in databases (see B9), not only in OS resources.
- Livelock is **not** deadlock: the processes are running.

---

## A7. Memory management, virtual memory and paging

**Goals:** give each process its own isolated address space, use RAM efficiently, let programs be larger than RAM, and protect processes from each other.

**Contiguous allocation and fragmentation**

- **Fixed vs variable partitions.** Placement policies for variable partitions: **first fit** (first hole big enough; fast), **best fit** (smallest hole that fits; leaves tiny useless holes), **worst fit** (largest hole; usually worst).
- **Internal fragmentation:** wasted space *inside* an allocated block (fixed partitions, paging's last page). **External fragmentation:** enough total free memory but in non-contiguous holes (variable partitions, segmentation). Fix: **compaction** (expensive) or **non-contiguous allocation (paging)**.

**Paging**

Physical memory is divided into fixed-size **frames**; the logical address space is divided into same-size **pages**. A **page table** maps page number to frame number. No external fragmentation; small internal fragmentation (up to one page per process).

```
Logical address = | page number p | offset d |         Physical address = | frame f | offset d |

      CPU --(p, d)--> page table[p] = f --(f, d)--> physical memory

Worked example: 4 KB pages (offset = 12 bits), logical address 0x12345
  page number = 0x12345 >> 12 = 0x12,  offset = 0x345
  page table[0x12] = frame 0x7   ->   physical address = 0x7345

If the logical address is 32 bits and the page is 4 KB: pages = 2^32 / 2^12 = 2^20 = 1,048,576 entries.
```

**Problem 1: page-table size.** 2^20 entries x 4 bytes = 4 MB per process. Solutions: **multi-level (hierarchical) page tables** (only allocate the parts in use), **hashed** page tables, **inverted** page tables (one entry per physical frame, slower lookup).

**Problem 2: every access needs two memory reads** (page table, then data). Solution: the **TLB** (Translation Lookaside Buffer), a small, fast hardware cache of recent page-to-frame translations.

**Effective access time (EAT) with a TLB (single-level page table)**

```
TLB lookup 20 ns, memory access 100 ns, hit ratio 80%
EAT = 0.8 * (20 + 100)  +  0.2 * (20 + 100 + 100)
    = 96 + 44 = 140 ns
```

**Segmentation:** divides the program by logical units (code, stack, heap) with variable size; address = (segment, offset), with base and limit checks. Suffers external fragmentation. Real systems combine them (paged segmentation).

**Virtual memory and demand paging**

Only the pages actually needed are in RAM; the rest stay on disk. Each page table entry has a **valid/present bit**, a **dirty (modified) bit**, a **reference bit** and **protection bits** (read/write/execute).

```
Page fault handling:
1. CPU references a page whose present bit = 0 -> trap to the OS
2. OS checks the access is legal (else segmentation fault)
3. OS finds a free frame (or picks a victim with a replacement algorithm; writes it back if dirty)
4. OS reads the page from disk into the frame            <- millisecond-scale I/O
5. Update page table (present = 1), restart the faulting instruction
```

**Page-fault cost:** `EAT = (1 - p) * memory_access + p * fault_service_time`. With memory access 200 ns, fault service 8 ms and fault rate p = 1/1000: `0.999 * 200 + 0.001 * 8,000,000 = 8,199.8 ns`, about **40 times slower**. Even a tiny fault rate hurts badly.

**Copy-on-write:** `fork()` shares pages read-only; the first write triggers a fault, and the OS copies just that page.

**Memory-mapped files, swap space, stack vs heap:** `mmap` maps a file into the address space; the stack is managed automatically (function frames), the heap manually (`malloc/new`) or by a garbage collector.

**Interview answer:** "Paging splits virtual memory into fixed-size pages mapped to physical frames through a page table, which removes external fragmentation. A TLB caches translations; virtual memory uses demand paging so only needed pages are in RAM, at the cost of page faults."

**Loopholes**

- Paging has **internal** fragmentation only; segmentation has **external**.
- Virtual address space size depends on the address width (32-bit = 4 GB), not on installed RAM.
- The TLB must be flushed (or tagged with ASIDs) on a process context switch.
- A page fault is a **trap, not an error**; only an invalid access becomes a segmentation fault.
- Big pages (2 MB huge pages) reduce TLB misses but increase internal fragmentation.

---

## A8. Page-replacement algorithms and thrashing

When memory is full and a fault occurs, the OS must choose a **victim** frame.

| Algorithm | Victim | Notes |
| --- | --- | --- |
| **FIFO** | the oldest loaded page | simple; suffers **Belady's anomaly** |
| **Optimal (OPT)** | page used farthest in the future | lowest possible faults; impossible in practice (benchmark only) |
| **LRU** | least recently used | good approximation of OPT; needs hardware support or approximation; **no** Belady's anomaly (stack algorithm) |
| **Clock (second chance)** | sweep a circular list; if reference bit is 1, clear it and move on; if 0, evict | cheap LRU approximation |
| **LFU / MFU** | least / most frequently used | counts can go stale |
| **NRU / enhanced clock** | prefer pages that are neither referenced nor dirty | avoids write-backs |

**Worked example (reference string, 3 frames)**

```
Reference: 7 0 1 2 0 3 0 4 2 3 0 3 2 1 2 0 1 7 0 1
FIFO: 15 page faults        LRU: 12 page faults        OPT: 9 page faults

LRU trace of the first part (frames show recency, most recent last):
7 -> [7]  fault
0 -> [7 0] fault
1 -> [7 0 1] fault
2 -> evict 7 (least recent) [0 1 2] fault
0 -> hit   [1 2 0]
3 -> evict 1 [2 0 3] fault
0 -> hit   [2 3 0]
4 -> evict 2 [3 0 4] fault ... and so on
```

**Belady's anomaly:** more frames can mean **more** faults in FIFO. Reference string `1 2 3 4 1 2 5 1 2 3 4 5`: FIFO has **9** faults with 3 frames and **10** faults with 4 frames. LRU and OPT never show it.

**Simulating FIFO and LRU**

```cpp
#include <bits/stdc++.h>
using namespace std;
int fifoFaults(const vector<int>& ref, int cap) {
    unordered_set<int> inMem; queue<int> order; int faults = 0;
    for (int p : ref) {
        if (inMem.count(p)) continue;
        faults++;
        if ((int)inMem.size() == cap) { inMem.erase(order.front()); order.pop(); }
        inMem.insert(p); order.push(p);
    }
    return faults;
}
int lruFaults(const vector<int>& ref, int cap) {
    list<int> recency; unordered_map<int, list<int>::iterator> pos; int faults = 0;
    for (int p : ref) {
        auto it = pos.find(p);
        if (it != pos.end()) recency.erase(it->second);           // hit: move to most-recent end
        else {
            faults++;
            if ((int)recency.size() == cap) { pos.erase(recency.front()); recency.pop_front(); }
        }
        recency.push_back(p); pos[p] = prev(recency.end());
    }
    return faults;
}
```

```python
from collections import OrderedDict, deque
def lru_faults(ref, cap):
    mem, faults = OrderedDict(), 0
    for p in ref:
        if p in mem: mem.move_to_end(p)
        else:
            faults += 1
            if len(mem) == cap: mem.popitem(last=False)
            mem[p] = True
    return faults
def fifo_faults(ref, cap):
    mem, q, faults = set(), deque(), 0
    for p in ref:
        if p not in mem:
            faults += 1
            if len(mem) == cap: mem.remove(q.popleft())
            mem.add(p); q.append(p)
    return faults
```

**Thrashing:** the system spends more time paging than executing. It happens when the combined **working sets** of active processes exceed physical memory. Symptoms: CPU utilization drops while disk activity soars; adding more processes (the OS sees low CPU use and admits more) makes it worse.

**Remedies:** reduce the degree of multiprogramming (suspend processes), use the **working-set model** (keep each process's recently referenced pages resident), **page-fault-frequency control** (give more frames when fault rate is high, take frames when low), local rather than global replacement, add RAM.

**Interview answer:** "LRU approximates the optimal policy by using the past as a predictor; FIFO is simple but can show Belady's anomaly. Thrashing occurs when processes don't have enough frames for their working sets, so the OS keeps faulting; the fix is to lower multiprogramming or give them more frames."

**Loopholes**

- Counting: the first reference to a page into an empty frame is still a fault (a **cold/compulsory miss**).
- OPT is not implementable; it is a lower bound.
- LRU needs per-access bookkeeping; exact LRU is costly, which is why the clock algorithm is used.
- Global replacement lets one process steal from another; local replacement isolates them but can underuse memory.
- Belady's anomaly applies to FIFO (and some others), not to stack algorithms.

---

## A8b. System calls

**Definition:** the controlled entry points through which a user program requests a kernel service (file I/O, process creation, memory, networking).

```
printf(...)  -> libc -> write(1, buf, n)   [wrapper]
  -> sets syscall number + args in registers -> executes `syscall` instruction (trap)
  -> CPU switches to kernel mode, jumps to the handler through the syscall table
  -> kernel does the work, puts the result in a register -> returns to user mode
```

| Category | Examples (Linux) |
| --- | --- |
| Process control | `fork`, `execve`, `wait`, `exit`, `kill` |
| File management | `open`, `read`, `write`, `close`, `lseek`, `stat` |
| Memory | `brk`, `mmap`, `munmap` |
| Device and IPC | `ioctl`, `pipe`, `socket`, `bind`, `connect`, `shmget` |
| Information | `getpid`, `time`, `uname` |

**Library function vs system call:** `malloc`, `printf`, `fopen` are libc functions that may batch work and *eventually* call syscalls (`brk`/`mmap`, `write`). Calling `write` directly is a syscall; `printf` buffers in user space first, which is why output can appear late.

```cpp
#include <unistd.h>
#include <fcntl.h>
int main() {
    const char msg[] = "hello via syscall\n";
    write(1, msg, sizeof msg - 1);                    // fd 1 = stdout
    int fd = open("/tmp/x.txt", O_CREAT | O_WRONLY, 0644);
    write(fd, msg, sizeof msg - 1);
    close(fd);
}
```

Try `strace ./a.out` on Linux to see every syscall a program makes.

**Interview answer:** "System calls are the API between user space and the kernel; they trigger a trap that switches to kernel mode, so they are more expensive than ordinary function calls."

**Loopholes:** a library call is not always a system call; `exit()` flushes stdio buffers while `_exit()` does not; file descriptors are per-process integers (0 stdin, 1 stdout, 2 stderr) and are inherited across `fork`; forgetting to check return values (`read` can return fewer bytes than requested).

---

## A9. Thread safety, compare-and-swap, atomics and TOCTOU

### Thread safety

**Definition:** code is thread-safe if it behaves correctly when called from several threads at once without extra coordination from the caller.

**Ways to achieve it (prefer the top of the list):**

1. **Avoid sharing:** thread confinement, thread-local storage, message passing.
2. **Immutability:** objects that never change are safe to share freely (`const`, `final`, frozen dataclasses).
3. **Stateless functions.**
4. **Atomics and lock-free structures.**
5. **Locks:** mutex, read-write lock, always with RAII.
6. **Concurrent containers** (`ConcurrentHashMap`, `queue.Queue`).

**Thread-safe vs reentrant:** reentrant code can be interrupted and called again before the first call finishes (uses no static/global state). A function can be thread-safe (guarded by a lock) yet not reentrant (the same thread re-locking a non-recursive mutex deadlocks).

**Thread-safe singleton (C++11): function-local static initialization is guaranteed thread-safe.**

```cpp
class Config {
    Config() = default;
public:
    static Config& instance() { static Config c; return c; }   // initialised once, thread-safe
    Config(const Config&) = delete; Config& operator=(const Config&) = delete;
};
```

### Compare-and-swap (CAS)

**Definition:** an atomic instruction `CAS(addr, expected, desired)`: if `*addr == expected`, store `desired` and return success; otherwise return failure (and report the current value). It is the basis of lock-free algorithms, atomics and spinlocks.

```cpp
atomic<int> counter{0};
void increment() {
    int old = counter.load();
    while (!counter.compare_exchange_weak(old, old + 1)) {}   // on failure, 'old' is refreshed, retry
}

// Lock-free stack push (Treiber stack)
struct Node { int v; Node* next; };
atomic<Node*> head{nullptr};
void push(int v) {
    Node* n = new Node{v, head.load()};
    while (!head.compare_exchange_weak(n->next, n)) {}        // n->next is updated on failure
}
```

```python
# Python has no user-level CAS; model it with a lock to understand the idea
import threading
class AtomicInt:
    def __init__(self, v=0): self.v, self._l = v, threading.Lock()
    def cas(self, expected, new):
        with self._l:
            if self.v == expected: self.v = new; return True
            return False
```

**ABA problem:** thread 1 reads A, thread 2 changes A to B and back to A, thread 1's CAS succeeds even though the structure changed underneath (in a lock-free stack a popped node can be freed and reused). Fixes: version-tagged pointers (double-width CAS), hazard pointers, epoch-based reclamation.

**Memory ordering (C++):** default `seq_cst` is the safest; `acquire`/`release` pairs establish happens-before for handing data between threads; `relaxed` gives atomicity only (fine for counters).

**Lock-free vs wait-free:** lock-free guarantees *some* thread always makes progress; wait-free guarantees *every* thread finishes in a bounded number of steps.

### TOCTOU (time-of-check to time-of-use)

**Definition:** a race where a condition is checked, then acted on later, and the state changes in between.

```cpp
// VULNERABLE: another process can replace /tmp/f with a symlink to /etc/passwd after access()
if (access("/tmp/f", W_OK) == 0) {
    int fd = open("/tmp/f", O_WRONLY);        // use
}
// FIX: do the operation atomically and handle failure; open first, then check the opened descriptor
int fd = open("/tmp/f", O_WRONLY | O_NOFOLLOW);
if (fd >= 0) { struct stat st; fstat(fd, &st); /* validate on the fd, not on the path */ }
```

**Everyday backend versions of the same bug**

```sql
-- BUG: check then act (two clients both see stock = 1)
SELECT stock FROM items WHERE id = 7;           -- 1
UPDATE items SET stock = stock - 1 WHERE id = 7; -- both succeed, stock goes to -1

-- FIX 1: one atomic conditional update
UPDATE items SET stock = stock - 1 WHERE id = 7 AND stock > 0;   -- check rows affected
-- FIX 2: row lock
SELECT stock FROM items WHERE id = 7 FOR UPDATE;
-- FIX 3: enforce with a constraint (UNIQUE, CHECK (stock >= 0)) instead of an app-level check
```

Same pattern: "check if username exists, then insert" (use a UNIQUE constraint and handle the error), "check balance then withdraw", "if cache is empty, load and store" (cache stampede), and file-exists-then-create (use `O_EXCL`).

**Interview answer:** "TOCTOU happens when a check and the action it guards are not atomic. The cure is to make them one atomic operation, or to hold a lock or database constraint that enforces the invariant."

**Loopholes**

- `volatile` is not atomic in C/C++; use `std::atomic`.
- A lock-free algorithm is not automatically faster; contention causes CAS retry storms.
- `compare_exchange_weak` may fail spuriously (use it in a loop); `_strong` does not.
- Checking `if (!map.count(k)) map[k] = v;` on a concurrent map is still a TOCTOU; use `insert`/`putIfAbsent`.
- Atomics protect one variable; invariants over several variables still need a lock.

---

# PART B: DATABASE MANAGEMENT SYSTEMS (DBMS)

## B1. DBMS basics

**Definition:** a DBMS is software that stores, retrieves, protects and manages data, providing concurrent access, recovery from failure, and a query language.

**DBMS vs file system:** files give no query language, weak concurrency control, no atomic multi-file updates, redundancy and inconsistency, and security only at file level. A DBMS offers declarative queries, transactions, integrity constraints, indexes, access control and crash recovery.

**Three-schema architecture and data independence**

```
External level  (views: what each user sees)
      | logical data independence: change the conceptual schema without changing views/apps
Conceptual level (whole logical design: tables, relations, constraints)
      | physical data independence: change storage/indexes without changing the logical schema
Internal level   (files, pages, indexes, storage structures)
```

**SQL sub-languages:** DDL (`CREATE`, `ALTER`, `DROP`, `TRUNCATE`), DML (`SELECT`, `INSERT`, `UPDATE`, `DELETE`), DCL (`GRANT`, `REVOKE`), TCL (`COMMIT`, `ROLLBACK`, `SAVEPOINT`).

**Relational model vocabulary:** relation = table, tuple = row, attribute = column, domain = allowed values, degree = number of attributes, cardinality = number of tuples, schema = structure, instance = current data.

**Relational algebra and its SQL form**

| Operator | Meaning | SQL |
| --- | --- | --- |
| selection sigma | filter rows | `WHERE` |
| projection pi | choose columns (removes duplicates in pure algebra) | `SELECT col` |
| union, intersection, difference | set operations (compatible schemas) | `UNION`, `INTERSECT`, `EXCEPT` |
| Cartesian product | all pairs | `CROSS JOIN` |
| join | product + condition | `JOIN ... ON` |
| rename rho | rename | `AS` |
| division | rows related to *all* of another set | `NOT EXISTS` double negation |

**Interview answer:** "A DBMS gives declarative data access, ACID transactions, constraints and recovery on top of raw storage, and hides physical storage details through data independence."

**Loopholes:** a database is the data; the DBMS is the software; SQL tables are **bags**, not sets (duplicates allowed unless a key prevents them), unlike pure relational algebra; `NULL` is not zero or empty string.

---

## B2. ACID properties and transactions

**Transaction:** a logical unit of work made of reads and writes that must be treated as indivisible.

**The classic example: transfer 100 from A to B**

```
BEGIN;
  UPDATE account SET bal = bal - 100 WHERE id = 'A';
  UPDATE account SET bal = bal + 100 WHERE id = 'B';
COMMIT;
```

| Property | Meaning | Violated if... | Mechanism |
| --- | --- | --- | --- |
| **Atomicity** | all or nothing | crash after the debit but before the credit leaves money lost | undo log / rollback, write-ahead log |
| **Consistency** | moves the database from one valid state to another (constraints, invariants hold) | total money changes, or a foreign key points nowhere | constraints, triggers, application logic |
| **Isolation** | concurrent transactions behave as if run one at a time (to a configured degree) | two transfers read the same balance and overwrite each other | locks, MVCC, isolation levels |
| **Durability** | once committed, survives crashes and power loss | commit acknowledged, machine dies, data gone | redo log flushed to disk (`fsync`), replication |

**Transaction states**

```
Active --> Partially committed --> Committed
   |              |
   +--> Failed <--+--> Aborted (rolled back; restart or kill)
```

**Write-ahead logging (WAL):** log the change **before** modifying the data page, and flush the log at commit. After a crash: **redo** committed changes not yet on disk, **undo** uncommitted changes that reached disk (ARIES recovery: analysis, redo, undo). Checkpoints bound recovery time.

```python
# Application pattern: always rollback on error
conn = connect()
try:
    cur = conn.cursor()
    cur.execute('UPDATE account SET bal = bal - %s WHERE id = %s', (100, 'A'))
    cur.execute('UPDATE account SET bal = bal + %s WHERE id = %s', (100, 'B'))
    conn.commit()
except Exception:
    conn.rollback()
    raise
```

**Interview answer:** "Atomicity is all-or-nothing, consistency preserves invariants, isolation hides concurrent interference, durability makes committed data survive crashes. Atomicity and durability come from the write-ahead log; isolation from locking or MVCC."

**Loopholes**

- **ACID consistency is not CAP consistency.** ACID-C is about integrity constraints; CAP-C is about all replicas seeing the same value.
- Many databases run in **autocommit** mode: each statement is its own transaction unless you `BEGIN`.
- In MySQL, DDL statements (`CREATE`, `ALTER`) cause an **implicit commit** and cannot be rolled back; PostgreSQL has transactional DDL.
- Durability can be weakened by async replication or `synchronous_commit = off` (a crash may lose the last commits).
- Long transactions hold locks and bloat logs/MVCC versions.

---

## B3. Keys and constraints

| Key | Definition |
| --- | --- |
| **Super key** | any set of attributes that uniquely identifies a row (may contain extra attributes) |
| **Candidate key** | a *minimal* super key (remove any attribute and it stops being unique) |
| **Primary key** | the chosen candidate key: unique and `NOT NULL`; one per table |
| **Alternate key** | candidate keys not chosen as primary |
| **Foreign key** | attribute(s) referencing a primary/unique key of another table |
| **Composite key** | key made of more than one attribute |
| **Surrogate key** | system-generated id (auto-increment, UUID) with no business meaning |
| **Natural key** | real-world attribute (email, passport number) |

**Counting:** if a relation has candidate keys, the number of super keys of R(A, B, C, D) with single candidate key A is `2^(n-1) = 8` (every subset containing A).

**Constraints:** `NOT NULL`, `UNIQUE`, `PRIMARY KEY`, `FOREIGN KEY`, `CHECK`, `DEFAULT`.

```sql
CREATE TABLE department (
  dept_id   INT PRIMARY KEY,
  name      VARCHAR(50) NOT NULL UNIQUE
);
CREATE TABLE employee (
  emp_id    INT PRIMARY KEY,
  email     VARCHAR(100) NOT NULL UNIQUE,
  salary    DECIMAL(10,2) CHECK (salary >= 0),
  dept_id   INT,
  FOREIGN KEY (dept_id) REFERENCES department(dept_id)
      ON DELETE SET NULL ON UPDATE CASCADE
);
```

**Referential actions:** `CASCADE` (propagate delete/update), `SET NULL`, `SET DEFAULT`, `RESTRICT`/`NO ACTION` (block the change).

**Natural vs surrogate:** natural keys can change (email) and may be long; surrogate keys are stable and small but add a meaningless column (keep a `UNIQUE` constraint on the natural key too).

**Loopholes**

- A **foreign key may be `NULL`** (no relationship), and may reference a `UNIQUE` column, not only the primary key.
- `UNIQUE` allows multiple `NULL`s in most databases (SQL Server allows only one).
- Primary key implies `NOT NULL` + `UNIQUE`, and normally creates a clustered/primary index.
- UUID primary keys cause random inserts into the B-tree (page splits, poor locality) versus sequential ids.
- Foreign key columns should be indexed, or deletes/joins on the parent scan the child table.

---

## B4. Normalization and functional dependencies

**Why normalize:** remove redundancy and the anomalies it causes.

```
StudentCourse(student_id, student_name, course_id, course_title)   <- one wide table
Insert anomaly: cannot add a course with no students.
Update anomaly: changing a student's name requires editing many rows (miss one = inconsistency).
Delete anomaly: deleting the last student of a course loses the course info.
```

**Functional dependency (FD):** `X -> Y` means whenever two rows agree on X, they agree on Y. **Armstrong's axioms:** reflexivity (if Y is a subset of X then X -> Y), augmentation (X -> Y implies XZ -> YZ), transitivity (X -> Y and Y -> Z imply X -> Z). Derived: union, decomposition, pseudo-transitivity.

**Attribute closure X+:** the set of all attributes determined by X. Algorithm: start with X, repeatedly add the right-hand side of any FD whose left side is already contained, until nothing changes. X is a super key iff X+ is all attributes; a candidate key is a minimal such X.

```python
def closure(attrs, fds):                  # fds: list of (lhs_set, rhs_set)
    res, changed = set(attrs), True
    while changed:
        changed = False
        for lhs, rhs in fds:
            if lhs <= res and not rhs <= res:
                res |= rhs; changed = True
    return res

fds = [({'A'}, {'B'}), ({'B'}, {'C'}), ({'C', 'D'}, {'E'})]
print(closure({'A'}, fds))                # {'A','B','C'}
print(closure({'A', 'D'}, fds))           # {'A','B','C','D','E'}  -> AD is a key of R(A,B,C,D,E)
```

**Finding candidate keys:** attributes that never appear on any right-hand side must be in every key; start from them and add attributes until the closure is everything.

**Normal forms**

| Form | Rule | Fixes |
| --- | --- | --- |
| **1NF** | every attribute is atomic (no repeating groups, no lists in a cell), rows are unique | multivalued cells |
| **2NF** | 1NF + no **partial dependency**: no non-key attribute depends on only *part* of a composite key | partial dependency |
| **3NF** | 2NF + no **transitive dependency**: non-key attributes depend only on the key (for every FD X -> A, X is a super key **or A is a prime attribute**) | transitive dependency |
| **BCNF** | for every non-trivial FD X -> Y, **X is a super key** | the remaining anomalies when candidate keys overlap |
| **4NF** | no non-trivial multivalued dependency unless the determinant is a super key | independent multivalued facts |
| **5NF** | no join dependency that is not implied by candidate keys | rare cases |

**Worked example: 2NF**

```
Enrollment(student_id, course_id, student_name, grade)     key: (student_id, course_id)
student_id -> student_name   is a PARTIAL dependency  =>  not 2NF
Decompose: Student(student_id, student_name)
           Enrollment(student_id, course_id, grade)
```

**Worked example: 3NF**

```
Employee(emp_id, name, dept_id, dept_name)       key: emp_id
emp_id -> dept_id -> dept_name  is TRANSITIVE  =>  not 3NF
Decompose: Employee(emp_id, name, dept_id)   Department(dept_id, dept_name)
```

**Worked example: 3NF but not BCNF**

```
R(student, course, teacher)    FDs: (student, course) -> teacher,  teacher -> course
Candidate keys: (student, course) and (student, teacher). Every attribute is prime, so R is in 3NF.
But teacher -> course violates BCNF (teacher is not a super key).
BCNF decomposition: R1(teacher, course), R2(student, teacher)
Cost: the FD (student, course) -> teacher is no longer enforced inside one table (dependency NOT preserved).
```

**Decomposition goals:** **lossless join** (joining the pieces gives back exactly the original; a decomposition into R1, R2 is lossless if `R1 intersect R2` is a key of R1 or R2) and **dependency preservation**. 3NF can always achieve both; BCNF always gives lossless join but may lose dependency preservation.

**Denormalization:** deliberately duplicating data for read speed (reporting, caches, NoSQL, precomputed counters). Trade-off: faster reads vs update anomalies and more write work.

**Interview answer:** "1NF atomic values; 2NF no partial dependency on a composite key; 3NF no transitive dependency; BCNF every determinant is a super key. I normalize to 3NF/BCNF for OLTP and denormalize selectively for read-heavy paths."

**Loopholes:** 2NF only matters with composite keys (a table with a single-column key is automatically in 2NF if in 1NF); 3NF allows a determinant that is not a super key if the dependent attribute is prime; over-normalization costs extra joins; NULL handling and multivalued attributes in 1NF.

---

## B5. ER diagrams

**Building blocks:** **entity** (rectangle), **weak entity** (double rectangle, depends on an owner and has only a partial key), **attribute** (oval; key attribute underlined; multivalued = double oval; derived = dashed; composite = has sub-attributes), **relationship** (diamond), **cardinality** (1:1, 1:N, M:N) and **participation** (total = double line, partial = single line).

```
[Student]--(1)----<Enrolls>----(N)--[Course]        M:N  (a student takes many courses, a course has many students)
  |                                    |
 sid, name                         cid, title      Enrolls carries its own attribute: grade

[Department]--(1)---<Has>---(N)--[Employee]          1:N  (employee side holds the foreign key)
```

**ER to relational mapping rules**

1. Strong entity → table; its key attribute → primary key; composite attribute → its components; **multivalued attribute → separate table** (entity key + value).
2. **1:N** → put the foreign key on the **N side**.
3. **1:1** → put the foreign key on either side (prefer the side with total participation), with `UNIQUE`.
4. **M:N** → a **junction (associative) table** with both foreign keys as a composite primary key, plus relationship attributes.
5. **Weak entity** → table whose primary key is (owner's key, partial key); foreign key to the owner with `ON DELETE CASCADE`.
6. **Generalization/specialization** (is-a): single table with a type column, table per subclass, or table per concrete class.

```sql
CREATE TABLE enrollment (
  student_id INT REFERENCES student(student_id),
  course_id  INT REFERENCES course(course_id),
  grade      CHAR(2),
  PRIMARY KEY (student_id, course_id)       -- composite key prevents duplicate enrollment
);
```

**Interview answer:** "I identify entities and their keys, relationships with cardinality and participation, then map: foreign key on the many side for 1:N, junction table for M:N, owner key inside the weak entity's key."

**Loopholes:** M:N relationships can never be a single foreign key; relationship attributes (grade, quantity, date) belong to the junction table; many ER diagrams hide **ternary** relationships (supplier, part, project), which need their own table.

---

## B6. Indexing and B-tree indexes

**Definition:** an index is a separate sorted data structure that lets the database find rows without scanning the whole table (like a book's index).

**Without index:** full table scan, O(n). **With a B+ tree index:** O(log\_f n) page reads where f (fan-out) is in the hundreds.

**B-tree vs B+ tree**

```
                 [ 50 | 100 ]                  <- root: keys + child pointers
               /      |       \
      [10|30]      [60|80]     [120|150]       <- internal nodes: keys only (in B+ tree)
       /  |  \      /  |  \      /  |   \
  leaves: [data] <-> [data] <-> [data] <-> ... <- B+ tree leaves are linked: fast range scans
```

|  | B-tree | B+ tree (used by InnoDB, PostgreSQL, SQL Server) |
| --- | --- | --- |
| Data location | in internal nodes and leaves | **only in leaves** |
| Internal nodes | hold keys and values | hold keys only, so more keys per page, higher fan-out, shorter tree |
| Range queries | need tree traversal | follow **linked leaf nodes** |
| Search time | can finish early | always reaches a leaf (predictable) |

**Height example:** 8 KB page, 8-byte key + 8-byte pointer gives fan-out about 500. Three levels hold about 500^3 = 125 million keys, so any row is found in about **3-4 disk reads**. The tree stays balanced via node splits and merges, so insert/delete/search are O(log n).

**Index types**

- **Clustered index:** the table rows are physically stored in index order (InnoDB primary key; one per table). **Non-clustered (secondary) index:** a separate structure whose leaves hold the key plus a pointer (the primary key in InnoDB, or a row id), so a lookup needs an extra step (a **bookmark lookup**) to fetch the row.
- **Composite index** on `(a, b, c)`: sorted by a, then b, then c. Usable for filters on `a`, `a,b`, `a,b,c` (**leftmost-prefix rule**), not for `b` alone.
- **Covering index:** the index contains every column the query needs, so the table is never touched (index-only scan).
- **Unique, partial/filtered, functional (expression), full-text, spatial, hash** (equality only, no ranges).

```sql
CREATE INDEX idx_emp_dept_sal ON employee (dept_id, salary);

EXPLAIN SELECT * FROM employee WHERE dept_id = 5 AND salary > 50000;   -- uses the index
EXPLAIN SELECT * FROM employee WHERE salary > 50000;                   -- cannot use it efficiently
```

**Choosing columns:** high **selectivity** (many distinct values), used in `WHERE`, `JOIN`, `ORDER BY`, `GROUP BY`. Put equality columns first and the range column last. Put the most selective, most-used column first only after checking the leftmost-prefix pattern of your queries.

**Costs:** every insert/update/delete must also update every index (slower writes), more storage, and a stale or excessive set of indexes slows the optimizer.

**Why the index may be ignored (the loopholes)**

- **Function on the column:** `WHERE YEAR(created) = 2025` or `LOWER(email) = 'x'` cannot use a plain index; rewrite as a range (`created >= '2025-01-01' AND created < '2026-01-01'`) or build a functional index.
- **Leading wildcard:** `LIKE '%abc'` cannot use the index; `LIKE 'abc%'` can.
- **Implicit type conversion:** comparing a string column with a number.
- **Low selectivity** (a `gender` column, boolean flags): the optimizer prefers a scan.
- **`OR` across different columns, `!=`, `NOT IN`**, and `SELECT *` that defeats covering indexes.
- The optimizer may skip the index when it expects to read a large share of the table; statistics can be stale (`ANALYZE`).
- Wrong column order in a composite index.

**Interview answer:** "Indexes are B+ trees that turn O(n) scans into O(log n) lookups with linked leaves for range scans. Clustered index defines physical row order; secondary indexes point back to the row. They speed reads but slow writes, so I index based on query patterns and verify with EXPLAIN."

**Loopholes:** a composite index `(a, b)` does not help `WHERE b = ?`; indexing every column is an anti-pattern; a non-clustered lookup in a huge result set (millions of rows) is slower than a table scan because of random I/O; indexes on tiny tables are useless; rebuilding or `ANALYZE` after bulk loads keeps statistics fresh.

---

## B7. Isolation levels

**Isolation** controls how much one transaction can see of another's in-flight work. Stronger isolation means fewer anomalies and less concurrency.

**Anomalies (know each with a timeline)**

```
Dirty read:           T1 UPDATE x=5 (not committed) -> T2 reads x=5 -> T1 ROLLBACK. T2 used data that never existed.
Non-repeatable read:  T1 reads x=1 -> T2 UPDATE x=2, COMMIT -> T1 reads x again = 2 (same row, different value).
Phantom read:         T1 SELECT WHERE age>30 gets 3 rows -> T2 INSERT age=40, COMMIT -> T1 repeats query, gets 4 rows.
Lost update:          T1 reads bal=100, T2 reads bal=100, T1 writes 150, T2 writes 120 -> T1's update is lost.
Write skew:           two transactions read an overlapping set, then update DIFFERENT rows, jointly breaking an invariant.
                      (Rule: at least one doctor on call. Alice and Bob each check 'two on call', each goes off call. Zero on call.)
```

**SQL-standard isolation levels**

| Level | Dirty read | Non-repeatable read | Phantom |
| --- | --- | --- | --- |
| Read Uncommitted | possible | possible | possible |
| Read Committed | **prevented** | possible | possible |
| Repeatable Read | prevented | **prevented** | possible (in the standard) |
| Serializable | prevented | prevented | **prevented** |

**Real engines differ from the table:**

- **PostgreSQL:** default Read Committed; its Repeatable Read is **snapshot isolation** (no phantoms, but **write skew is possible**); Serializable uses SSI (detects dangerous patterns and aborts one transaction, so retry on serialization failure). It has no true Read Uncommitted.
- **MySQL InnoDB:** default Repeatable Read, using MVCC snapshots plus **gap/next-key locks**, which also prevent most phantoms for locking reads.
- **Oracle / SQL Server:** default Read Committed (SQL Server can use snapshot isolation).

**How isolation is implemented**

- **Locking:** shared (S) locks for reads, exclusive (X) locks for writes; compatibility: S-S ok, anything with X conflicts. **Two-phase locking (2PL)**: a growing phase (acquire) then a shrinking phase (release), never acquiring after the first release. **Strict 2PL** holds write locks until commit, which avoids cascading aborts. Range/predicate/gap locks prevent phantoms.
- **MVCC (multi-version concurrency control):** each write creates a new row version stamped with a transaction id; a reader sees the snapshot that was current when its transaction (or statement) began. **Readers do not block writers and writers do not block readers.** Old versions are cleaned up later (VACUUM in PostgreSQL, purge in InnoDB).
- **Optimistic concurrency control:** do not lock; check at commit (version column) and retry on conflict.

```sql
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
BEGIN;
  SELECT stock FROM items WHERE id = 7 FOR UPDATE;       -- take an X lock on the row
  UPDATE items SET stock = stock - 1 WHERE id = 7;
COMMIT;

-- Optimistic locking with a version column (prevents lost updates without long locks)
UPDATE profile SET name = 'Asha', version = version + 1
WHERE id = 42 AND version = 5;           -- if 0 rows affected, someone else changed it: reload and retry
```

**Interview answer:** "Read Committed stops dirty reads, Repeatable Read also stops non-repeatable reads, Serializable also stops phantoms. Most databases implement this with MVCC so readers don't block writers; stricter levels cost throughput and can require retries."

**Loopholes**

- Snapshot isolation is **not** serializable: it allows write skew.
- The standard's "Repeatable Read allows phantoms" is not true for PostgreSQL (snapshot) or MySQL for many cases; answer by naming the engine.
- Lost updates happen at Read Committed with read-modify-write in application code; fix with atomic `UPDATE ... SET x = x + 1`, `FOR UPDATE`, or optimistic version checks.
- `SELECT ... FOR UPDATE` locks only the rows it finds; it does not stop another transaction inserting a new matching row unless gap/predicate locks exist.
- Higher isolation does not mean correctness for multi-statement business logic spanning transactions.

---

## B8. Serializability and recoverability

**Schedule:** an interleaving of the operations of several transactions. **Serial schedule:** transactions run one after another (always correct, no concurrency). **Serializable schedule:** equivalent in effect to some serial schedule.

**Conflicting operations:** two operations from **different** transactions on the **same item**, at least one a write: **RW**, **WR**, **WW**. Non-conflicting operations (two reads, or different items) can be swapped freely.

**Conflict serializability test: precedence graph.** One node per transaction; draw an edge Ti -> Tj if an operation of Ti conflicts with a *later* operation of Tj. **The schedule is conflict serializable iff the graph has no cycle**; a topological order gives an equivalent serial order.

```
Schedule S1:  R1(A) W1(A) R2(A) W2(A) R1(B) W1(B) R2(B) W2(B)
  edges: T1 -> T2 (on A), T1 -> T2 (on B)  -> acyclic  -> serializable, equivalent to T1 then T2

Schedule S2 (lost update):  R1(A) R2(A) W1(A) W2(A)
  R1(A) before W2(A): T1 -> T2
  R2(A) before W1(A): T2 -> T1        -> CYCLE  -> not conflict serializable
```

```python
def conflict_serializable(schedule):          # schedule: list of (txn, 'R'|'W', item)
    from collections import defaultdict
    g, txns = defaultdict(set), {t for t, _, _ in schedule}
    for i, (t1, op1, x1) in enumerate(schedule):
        for t2, op2, x2 in schedule[i + 1:]:
            if t1 != t2 and x1 == x2 and 'W' in (op1, op2): g[t1].add(t2)
    color = {t: 0 for t in txns}
    def cyclic(u):
        color[u] = 1
        for v in g[u]:
            if color[v] == 1 or (color[v] == 0 and cyclic(v)): return True
        color[u] = 2; return False
    return not any(color[t] == 0 and cyclic(t) for t in txns)
```

**View serializability** is weaker (more schedules qualify, e.g. blind writes), but testing it is NP-complete, so databases enforce conflict serializability. Every conflict-serializable schedule is view-serializable; the reverse is false.

**Recoverability**

- **Recoverable:** a transaction commits only after every transaction it read from has committed (otherwise a commit could depend on rolled-back data).
- **Cascadeless (ACA):** a transaction reads only committed data, so one abort never forces others to abort.
- **Strict:** no read *or write* of an item written by an uncommitted transaction (what strict 2PL gives), which makes undo simple.
- Hierarchy: strict is a subset of cascadeless, which is a subset of recoverable.

**Protocols that guarantee serializability:** 2PL (conflict serializable, but may deadlock), strict/rigorous 2PL, timestamp ordering (older timestamp wins; may restart), validation-based optimistic control, MVCC with SSI.

**Interview answer:** "A schedule is conflict serializable if its precedence graph is acyclic. 2PL guarantees this; strict 2PL also gives recoverability. Databases enforce conflict rather than view serializability because it is efficiently checkable."

**Loopholes:** serializable does not mean serial; a cyclic graph means not conflict serializable (it might still be view serializable); 2PL does **not** prevent deadlock; basic 2PL can still have cascading aborts.

---

## B9. Transaction deadlocks

**Definition:** two or more transactions each hold a lock the other needs, so none can continue (the OS deadlock of A6 applied to database locks).

```
Time  T1                                    T2
 t1   BEGIN; UPDATE acct SET .. WHERE id=1
 t2                                         BEGIN; UPDATE acct SET .. WHERE id=2
 t3   UPDATE acct SET .. WHERE id=2  (waits for T2)
 t4                                         UPDATE acct SET .. WHERE id=1  (waits for T1)  -> DEADLOCK
```

**Detection:** the DBMS maintains a **wait-for graph** and looks for cycles periodically (or on each lock wait). It chooses a **victim** (usually the cheapest to roll back), aborts it, and the other proceeds. Error codes: MySQL 1213, PostgreSQL `40P01`, SQL Server 1205. Some systems use lock **timeouts** instead.

**Prevention schemes (timestamp based):** **wait-die** (an older transaction may wait for a younger one; a younger requester dies/aborts) and **wound-wait** (an older requester wounds/aborts the younger holder; a younger requester waits). Both avoid cycles.

**How to avoid deadlocks in practice**

1. **Access rows and tables in a consistent order** everywhere (transfer: always lock the lower account id first).
2. Keep transactions **short**; do not wait for user input inside one.
3. Lock only what you need; use appropriate indexes so a statement locks few rows instead of scanning (and locking) many.
4. Use a lower isolation level where acceptable.
5. Take locks up front (`SELECT ... FOR UPDATE` on all needed rows in order).
6. **Retry** the victim transaction.

```python
import time, random
def run_with_retry(fn, tries=5):
    for attempt in range(tries):
        try:
            return fn()                                  # fn runs one whole transaction
        except DeadlockOrSerializationError:             # driver-specific exception
            time.sleep((2 ** attempt) * 0.05 + random.random() * 0.05)   # backoff with jitter
    raise RuntimeError('too many retries')
```

```cpp
// Application-level equivalent in C++: lock two mutexes in a global order
void transfer(Account& a, Account& b) {
    scoped_lock lk(a.m, b.m);     // C++17 locks both using a deadlock-avoidance algorithm
    /* move money */
}
```

**Loopholes:** a deadlock is not a slow query (nothing is progressing); the database rolls back **one** transaction, so the application **must** catch the error and retry; deadlocks can also arise from gap locks and foreign key checks; row-lock waits that eventually time out are different from deadlocks.

---

## B10. Query processing and query optimization

**Pipeline**

```
SQL text -> Parser (syntax tree) -> Binder/analyzer (check tables, columns, types)
         -> Logical plan (relational algebra) -> Optimizer (rewrite + cost-based plan choice)
         -> Physical plan (chosen scans, join algorithms) -> Executor -> result rows
```

**Logical order of evaluation (explains many "why can't I use my alias" errors):** `FROM/JOIN` → `WHERE` → `GROUP BY` → `HAVING` → `SELECT` → `DISTINCT` → `ORDER BY` → `LIMIT`. `WHERE` filters rows before grouping; `HAVING` filters groups after aggregation; aliases defined in `SELECT` are not visible in `WHERE` (but are in `ORDER BY`).

**Rule-based rewrites:** **predicate pushdown** (filter early, before joins), **projection pushdown** (read only needed columns), constant folding, subquery unnesting, join reordering.

**Cost-based optimization:** the optimizer enumerates candidate plans and estimates their cost using **statistics** (row counts, distinct values, histograms) and picks the cheapest. Bad statistics lead to bad plans; refresh with `ANALYZE`.

**Access paths:** sequential scan, index scan, index-only scan (covering index), bitmap scan.

**Join algorithms**

| Algorithm | How | Cost | Best when |
| --- | --- | --- | --- |
| Nested loop | for each outer row scan the inner | O(n \* m) | tiny inputs, or an **index on the inner** join column (index nested loop: O(n log m)) |
| Hash join | build a hash table on the smaller input, probe with the larger | O(n + m) | large unsorted inputs, **equality joins only** |
| Sort-merge join | sort both on the join key and merge | O(n log n + m log m) | inputs already sorted, range/inequality joins, large joins |

```sql
EXPLAIN ANALYZE
SELECT o.id, c.name
FROM orders o JOIN customer c ON c.id = o.customer_id
WHERE o.created_at >= '2026-01-01' AND c.country = 'IN';
-- Read: scan type (Seq Scan vs Index Scan), join type (Hash Join / Nested Loop),
-- estimated rows vs actual rows (big gaps mean stale stats), loops, total time.
```

**Optimization checklist (high-yield)**

1. Select only needed columns; avoid `SELECT *` (blocks covering indexes, wastes I/O and network).
2. Make predicates **sargable** (search-argument able): compare the bare column with a constant, no function on the column.
3. Index columns used in `WHERE`, `JOIN`, `ORDER BY`; use composite and covering indexes.
4. Avoid the **N+1 query problem** (one query for a list, then one per row): use a join or an `IN (...)` batch.
5. Prefer `EXISTS` to `IN` with large subqueries where the optimizer does not already unnest; prefer `UNION ALL` over `UNION` when duplicates are impossible or acceptable (`UNION` sorts to deduplicate).
6. Paginate with **keyset (seek) pagination** (`WHERE id > :last ORDER BY id LIMIT 20`) instead of large `OFFSET`.
7. Batch inserts, use prepared statements, keep transactions short.
8. Cache hot reads (Redis) and precompute aggregates where appropriate.
9. Partition huge tables; archive old data.

```sql
-- Slow: function on the column forces a scan
SELECT * FROM orders WHERE DATE(created_at) = '2026-03-01';
-- Fast: range on the bare column, uses the index on created_at
SELECT id, total FROM orders
WHERE created_at >= '2026-03-01' AND created_at < '2026-03-02';
```

**Interview answer:** "The optimizer parses the query, rewrites it logically (push filters down), then picks the cheapest physical plan using statistics: access path per table and a join algorithm. I read EXPLAIN to check scans, join types and row estimates, and fix with indexes, sargable predicates and better statistics."

**Loopholes:** `EXPLAIN` shows the estimated plan, `EXPLAIN ANALYZE` actually runs the query (and runs `UPDATE/DELETE` for real); the plan can change with data size and parameters; hash join cannot handle non-equality joins; the optimizer is not magic: an unindexed foreign key or stale stats can still produce disastrous plans; `COUNT(*)` on big InnoDB tables is a scan.

---

## B11. SQL vs NoSQL trade-offs

|  | SQL (relational) | NoSQL |
| --- | --- | --- |
| Data model | tables, rows, fixed schema, relationships via keys | key-value, document, wide-column, graph; flexible schema |
| Query | SQL, joins, aggregations | per-store API; limited joins/aggregation |
| Consistency | ACID transactions | often tunable/eventual (BASE), though many now offer transactions |
| Scaling | vertical first; read replicas; sharding is harder | designed for horizontal scale-out and partitioning |
| Best for | complex relations, reporting, financial/transactional data | huge write volume, flexible/evolving schema, simple access patterns, low latency |

**NoSQL families**

- **Key-value** (Redis, DynamoDB): O(1) get/put by key; sessions, caching, counters.
- **Document** (MongoDB, Couchbase): JSON-like documents; catalogs, user profiles, content.
- **Wide-column** (Cassandra, HBase): partitioned by key, extremely high write throughput; time-series, messaging, activity logs.
- **Graph** (Neo4j): relationships as first-class; social networks, recommendations, fraud rings.
- **Search** (Elasticsearch) and **time-series** (InfluxDB, TimescaleDB).

**BASE vs ACID:** Basically Available, Soft state, Eventually consistent (see CAP and PACELC in the system design part).

**How to choose:** start from the **access patterns**, consistency needs and scale. Use SQL when data is relational, requirements are transactional, or you need ad-hoc queries; use NoSQL when you need massive horizontal scale, flexible schema, or a specialized model (graph, cache). Many systems use **polyglot persistence**: PostgreSQL as the system of record, Redis for cache, Elasticsearch for search, a queue for events.

**Interview answer:** "I choose by access pattern and consistency need: relational with ACID for transactional, join-heavy data; a key-value or document or wide-column store when scale, latency and schema flexibility dominate, accepting eventual consistency and weaker querying."

**Loopholes:** "SQL doesn't scale" and "NoSQL has no schema" are both oversimplifications (relational databases scale with replicas, partitioning and sharding; NoSQL applications still have an implicit schema); MongoDB and DynamoDB support transactions now; "NoSQL = no joins" means joins are done in the application or avoided by denormalizing; choosing NoSQL because it is fashionable is a red flag in an interview.

---

# PART C: COMPUTER NETWORKS

## C1. OSI and TCP/IP models

```
OSI (7 layers)             TCP/IP (4 layers)     PDU          Examples / devices
7 Application   \                                 data         HTTP, DNS, SMTP, FTP, SSH
6 Presentation   }----->  Application            data         TLS (encryption), encoding, compression
5 Session       /                                 data         session setup/teardown, sockets API
4 Transport     ------->  Transport               segment/dgram  TCP, UDP (ports)
3 Network       ------->  Internet                packet       IP, ICMP, routing; ROUTER
2 Data link     \                                 frame        Ethernet, Wi-Fi, ARP, MAC; SWITCH
1 Physical      /-------> Link/Network access     bits         cables, radio, hubs
```

**Mnemonic (top to bottom):** All People Seem To Need Data Processing. **Encapsulation:** each layer wraps the data of the layer above with its own header (and the link layer adds a trailer): `HTTP data -> TCP header + data = segment -> IP header + segment = packet -> Ethernet header + packet + trailer = frame`. The receiver strips them in reverse order (**decapsulation**).

**Addressing at each level:** application = URL/hostname, transport = **port**, network = **IP address** (logical, end to end), link = **MAC address** (physical, changes at every hop).

**Devices:** hub (layer 1, repeats to all ports), **switch** (layer 2, learns MAC addresses, forwards to the right port), **router** (layer 3, forwards between networks by IP), gateway/firewall/load balancer (layers 3 to 7).

**Loopholes:** TLS is placed in different layers by different books (presentation in OSI, between transport and application in practice); the TCP/IP model is what the internet actually uses, OSI is a reference model; the IP address stays the same end to end (apart from NAT), the MAC address changes each hop.

---

## C2. TCP vs UDP, handshake, flow and congestion control

|  | TCP | UDP |
| --- | --- | --- |
| Connection | connection-oriented (handshake) | connectionless |
| Reliability | acknowledgements, retransmission, ordering, checksums | none (checksum only); no ordering |
| Flow/congestion control | yes | no |
| Header | 20+ bytes | 8 bytes |
| Speed / overhead | slower, heavier | faster, lighter |
| Delivery unit | byte **stream** (no message boundaries) | **datagrams** (message boundaries kept) |
| Used by | HTTP/1.1, HTTP/2, HTTPS, SSH, email, FTP, databases | DNS, DHCP, VoIP, video streaming, online games, QUIC/HTTP/3 |

**Three-way handshake (connection setup)**

```
Client                       Server
  | ---- SYN, seq=x ---------> |      client chooses initial sequence number x
  | <--- SYN+ACK, seq=y, ack=x+1 |    server chooses y and acknowledges x
  | ---- ACK, ack=y+1 ------> |      connection ESTABLISHED, data may flow
```

**Why three steps, not two:** both sides must confirm that the other can **send and receive**, and agree on initial sequence numbers; a delayed duplicate SYN from an old connection could otherwise create a false half-open connection.

**Four-way termination:** `FIN` (A done sending) → `ACK`; `FIN` (B done) → `ACK`. The side that closes first sits in **TIME\_WAIT** for 2 x MSL (about 1 to 4 minutes) so late packets die out and the final ACK can be retransmitted. Many connections in `TIME_WAIT` can exhaust ports on a busy client.

**Reliability:** sequence numbers number the bytes; the receiver sends cumulative ACKs; the sender retransmits on timeout or after **3 duplicate ACKs** (fast retransmit); checksum detects corruption.

**Flow control (protects the receiver):** the receiver advertises a **receive window (rwnd)**, the free buffer space; the sender never has more than rwnd bytes unacknowledged. rwnd = 0 stops the sender until a window update. **Sliding window.**

**Congestion control (protects the network):** the sender keeps a **congestion window (cwnd)**; it may have `min(cwnd, rwnd)` bytes in flight.

```
cwnd
  |            /\  <- loss detected (timeout): ssthresh = cwnd/2, cwnd = 1 MSS, back to slow start
  |          /   \
  |  slow  /      \ ... after 3 dup ACKs (fast retransmit/recovery): cwnd = ssthresh = cwnd/2, then linear growth
  | start /  congestion avoidance (+1 MSS per RTT, linear)
  |  (exponential: doubles every RTT until ssthresh)
  +---------------------------------> time
```

**Slow start** doubles cwnd each RTT up to `ssthresh`; **congestion avoidance** adds about one MSS per RTT (AIMD: additive increase, multiplicative decrease); **fast retransmit/fast recovery** react to duplicate ACKs without dropping all the way to 1 MSS (TCP Reno). Modern variants: CUBIC (Linux default), BBR (model-based).

**Sockets: minimal TCP echo in Python and C++**

```python
import socket
def server():
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    s.bind(('0.0.0.0', 9000)); s.listen(128)
    while True:
        conn, addr = s.accept()               # blocks until a client connects (after the handshake)
        with conn:
            while (data := conn.recv(4096)):  # recv may return fewer bytes than sent: TCP is a stream
                conn.sendall(data)            # sendall loops until everything is written

def client():
    c = socket.create_connection(('127.0.0.1', 9000))
    c.sendall(b'hello'); print(c.recv(4096)); c.close()
```

```cpp
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <unistd.h>
int main() {
    int fd = socket(AF_INET, SOCK_STREAM, 0);
    sockaddr_in a{}; a.sin_family = AF_INET; a.sin_port = htons(9000);   // htons: host to network byte order
    inet_pton(AF_INET, "127.0.0.1", &a.sin_addr);
    if (connect(fd, (sockaddr*)&a, sizeof a) == 0) {
        const char msg[] = "hello"; send(fd, msg, sizeof msg - 1, 0);
        char buf[4096]; ssize_t n = recv(fd, buf, sizeof buf, 0);        // n < 0 on error, 0 if peer closed
        (void)n;
    }
    close(fd);
}
```

**Interview answer:** "TCP gives a reliable, ordered byte stream using handshakes, sequence numbers, ACKs and retransmission, plus flow control (receiver window) and congestion control (slow start, AIMD). UDP is a thin, unreliable datagram service with lower latency, so applications that tolerate loss, or build their own reliability like QUIC, use it."

**Loopholes**

- TCP preserves **bytes, not messages**: one `send` may arrive as several `recv`s and vice versa; you must frame messages yourself (length prefix or delimiter).
- "UDP is unreliable" means the protocol does not recover loss; it does not mean packets are usually lost.
- A **SYN flood** fills the half-open queue; mitigations: SYN cookies, rate limits.
- TCP **head-of-line blocking:** one lost segment stalls all later data in that stream.
- Flow control (receiver) and congestion control (network) are different mechanisms.
- `TIME_WAIT` is not a bug; enable port reuse carefully rather than disabling it.

---

## C3. IP addressing, subnetting, routing

**IPv4:** 32 bits written as four decimal octets. **CIDR notation** `a.b.c.d/p`: the first `p` bits are the **network** part, the rest the **host** part. Usable hosts per subnet = `2^(32 - p) - 2` (minus the network and the broadcast address).

**Private ranges (RFC 1918, not routable on the internet):** `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`. Also: `127.0.0.0/8` loopback, `169.254.0.0/16` link-local (DHCP failed), `224.0.0.0/4` multicast.

**Worked example: 192.168.10.77/26**

```
/26 -> mask 255.255.255.192, block size 256 - 192 = 64, subnets start at multiples of 64 in the last octet
77 lies in 64..127
Network address  : 192.168.10.64
First host       : 192.168.10.65
Last host        : 192.168.10.126
Broadcast address: 192.168.10.127
Usable hosts     : 2^6 - 2 = 62

Split 192.168.1.0/24 into 4 equal subnets: borrow 2 bits -> /26 -> 192.168.1.0, .64, .128, .192, each 62 hosts.
```

```python
import ipaddress
n = ipaddress.ip_network('192.168.10.77/26', strict=False)
print(n.network_address, n.broadcast_address, n.num_addresses - 2)   # 192.168.10.64 192.168.10.127 62
print(list(ipaddress.ip_network('192.168.1.0/24').subnets(new_prefix=26)))
print(ipaddress.ip_address('192.168.10.80') in n)                    # True
```

**Classful ranges (historical):** A 1-126 (/8), B 128-191 (/16), C 192-223 (/24). Modern networks use classless CIDR and **VLSM** (different prefix lengths in one network) with **route summarization** (aggregating prefixes).

**NAT / PAT:** a router rewrites private source addresses (and ports, for PAT) to its public address and keeps a translation table, so many devices share one public IP. Side effect: inbound connections need port forwarding.

**IPv6:** 128-bit addresses (`2001:db8::1`), no broadcast and no NAT needed, built-in autoconfiguration (SLAAC), simplified header. Extremely large address space.

**Routing**

- A router forwards by **longest prefix match** in its routing table; a `0.0.0.0/0` **default route** catches everything else. Each packet's **TTL** decreases at every hop; at 0 the router drops it and sends ICMP "time exceeded" (this is how `traceroute` works).
- **Intra-domain protocols:** **RIP** (distance vector, hop count, Bellman-Ford idea, slow convergence, count-to-infinity), **OSPF** (link-state, floods topology, each router runs **Dijkstra**). **Inter-domain:** **BGP** (path vector between autonomous systems; the protocol that glues the internet together; policy-driven).
- **ICMP:** error and diagnostic messages (`ping` = echo request/reply, destination unreachable).
- **MTU and fragmentation:** Ethernet MTU 1500 bytes; larger IPv4 packets are fragmented (or dropped if `DF` is set; Path MTU Discovery).

**Loopholes:** the subnet's first address is the network id and the last is broadcast, so they are not assignable hosts (a /31 point-to-point link is a special case); `/24` has 254 hosts, not 256; private addresses cannot be reached from the internet without NAT; routers separate broadcast domains, switches do not (VLANs do).

---

## C4. ARP, DHCP and switching

**ARP (Address Resolution Protocol):** finds the **MAC address** for a known **IP address** on the same LAN. Host broadcasts "who has 192.168.1.1?"; the owner replies unicast with its MAC; the result is cached (ARP table). To reach an address **outside** the subnet, the host ARPs for the **default gateway's** MAC, not the destination's.

**ARP spoofing/poisoning:** an attacker replies with false mappings to intercept traffic (man in the middle); mitigated by dynamic ARP inspection and static entries. ARP has no authentication.

**DHCP (Dynamic Host Configuration Protocol)** automatically gives a host an **IP address, subnet mask, default gateway, DNS servers and a lease time**. It uses UDP ports 67 (server) and 68 (client). The exchange is **DORA**:

```
Client -> broadcast: DHCP DISCOVER   (I need an address; I don't have one yet: source 0.0.0.0)
Server -> OFFER                       (here is 192.168.1.50 for 24 h)
Client -> broadcast: REQUEST          (I accept that offer; tells other servers)
Server -> ACK                         (confirmed; lease starts, renew at 50% of lease time)
```

**Switch behaviour:** a switch **learns** source MAC addresses per port into a CAM/MAC table; it forwards unicast frames only to the right port, **floods** unknown or broadcast frames. Each switch port is its own **collision domain**; all ports in a VLAN share a **broadcast domain**.

**Loopholes:** ARP works only within one broadcast domain; DHCP uses broadcast because the client has no IP yet, so a relay agent is needed across routers; a rogue DHCP server can hijack clients; `169.254.x.x` addresses mean DHCP failed.

---

## C5. DNS

**Definition:** the distributed, hierarchical database that maps names to IP addresses (and other records). Port **53** (UDP for normal queries, TCP for large responses and zone transfers).

**Hierarchy:** root (`.`) → TLD (`.com`, `.in`) → second-level domain (`example.com`) → subdomains (`www.example.com`).

**Resolution of www.example.com (cache empty)**

```
Browser cache -> OS cache / hosts file -> configured recursive resolver (ISP, 8.8.8.8, 1.1.1.1)
   resolver -> ROOT server:        "ask the .com TLD servers"
   resolver -> .com TLD server:    "ask example.com's authoritative name servers"
   resolver -> AUTHORITATIVE server: "www.example.com A 93.184.216.34  TTL 300"
   resolver caches the answer for TTL seconds and returns it to the client
```

**Recursive** query: the resolver does all the work for the client. **Iterative** queries: each server just refers the resolver to the next one.

**Record types:** `A` (IPv4), `AAAA` (IPv6), `CNAME` (alias to another name), `NS` (name servers of a zone), `MX` (mail servers with priority), `TXT` (SPF, DKIM, verification), `PTR` (reverse lookup), `SOA` (zone authority/serial), `SRV` (service location).

**Caching and TTL:** answers are cached at the browser, OS and resolver for the record's TTL; lower the TTL **before** a planned migration. "DNS propagation" is just caches expiring.

```bash
dig www.example.com A +short
dig +trace www.example.com         # follow root -> TLD -> authoritative
nslookup example.com
```

**DNS-based load balancing / GeoDNS:** return different IPs by round robin or by client location; Anycast routes the same IP to the nearest server (used by root servers and CDNs).

**Loopholes**

- A **CNAME cannot coexist with other records at the same name** and is not allowed at the zone apex (`example.com`); use ALIAS/ANAME features of providers.
- DNS is unauthenticated by default: **cache poisoning** and spoofing; mitigations DNSSEC, DNS over HTTPS/TLS, randomized ports and IDs.
- A DNS answer does not prove the server is reachable or healthy.
- Negative answers (NXDOMAIN) are cached too.
- HTTPS encrypts the HTTP content but the DNS query (without DoH) and the SNI field can still reveal which site you visit.

---

## C6. HTTP and HTTPS

**HTTP:** an application-layer, **stateless**, request-response protocol over TCP (or QUIC for HTTP/3). Default ports 80 (HTTP) and 443 (HTTPS).

```
GET /products?id=7 HTTP/1.1                    <- request line: method, path+query, version
Host: shop.example.com                         <- headers (Host is mandatory in HTTP/1.1)
Accept: application/json
Authorization: Bearer eyJhbGciOi...
Cookie: sid=abc123
                                               <- blank line, then optional body

HTTP/1.1 200 OK                                <- status line
Content-Type: application/json
Cache-Control: max-age=60
ETag: v7

{ "id": 7, "name": "Pen" }                      <- response body
```

**Methods**

| Method | Purpose | Safe (no side effects) | Idempotent (repeating = same effect) | Body |
| --- | --- | --- | --- | --- |
| GET | read | yes | yes | no |
| HEAD | headers only | yes | yes | no |
| POST | create / process | no | **no** | yes |
| PUT | replace the whole resource (or create at a known URL) | no | yes | yes |
| PATCH | partial update | no | not guaranteed | yes |
| DELETE | remove | no | yes | optional |
| OPTIONS | allowed methods, CORS preflight | yes | yes | no |

**Important headers:** `Host`, `Content-Type`, `Content-Length`, `Accept`, `Authorization`, `Cookie` / `Set-Cookie`, `User-Agent`, `Cache-Control`, `ETag`/`If-None-Match`, `Last-Modified`/`If-Modified-Since`, `Location` (redirects), `Origin` and `Access-Control-Allow-Origin` (CORS).

**Caching:** `Cache-Control: max-age=N` lets a client reuse a response without asking. **Conditional requests:** the client sends `If-None-Match: <ETag>`; if unchanged the server returns **304 Not Modified** with no body. `no-cache` means "revalidate before use", `no-store` means "never store".

**CORS:** browsers block cross-origin JavaScript reads unless the server allows them (`Access-Control-Allow-Origin`). For non-simple requests, the browser first sends an `OPTIONS` **preflight**. CORS is a **browser** protection, not server authentication.

**Persistent connections:** HTTP/1.1 keeps the TCP connection open (`Connection: keep-alive`) to avoid repeating the handshake.

**HTTPS = HTTP over TLS:** confidentiality (encryption), integrity (tampering detected), authentication (the server proves its identity with a certificate).

```python
import requests
r = requests.get('https://api.example.com/items', params={'page': 1}, timeout=5)
r.raise_for_status(); print(r.status_code, r.json())
```

**Loopholes:** HTTP is stateless (state lives in cookies, tokens or server-side sessions); `GET` must not change state (crawlers and prefetchers call it); idempotent is not the same as safe (`DELETE` is idempotent but not safe); `POST` retries can duplicate orders unless you add an **idempotency key**; passwords and tokens must never be placed in URLs (logged everywhere).

---

## C7. TLS handshake and certificates

**Goals:** encrypt data, detect tampering, authenticate the server (optionally the client, mutual TLS).

**Idea (hybrid cryptography):** **asymmetric** cryptography (RSA/ECDSA signatures, ECDHE key exchange) is slow and is used only to authenticate and agree on a secret; **symmetric** cryptography (AES-GCM, ChaCha20-Poly1305) is fast and encrypts the actual traffic.

```
TLS 1.3 handshake (1 round trip):
Client -> ClientHello (supported ciphers, key share, SNI = hostname)
Server -> ServerHello (chosen cipher, key share) + Certificate + signature (CertificateVerify) + Finished
          -- both sides now derive the same session keys via ECDHE; everything after ServerHello is encrypted
Client -> Finished   -> application data flows, encrypted with symmetric keys
TLS 1.2 needs 2 round trips; TLS 1.3 also supports 0-RTT resumption (replay risk).
```

**Certificates and the chain of trust:** a certificate binds a public key to a domain name and is signed by a **Certificate Authority (CA)**. The browser verifies: the signature chain up to a root CA it already trusts (leaf → intermediate → root), the domain matches (CN/SAN), validity dates, and revocation status (CRL/OCSP).

**Forward secrecy:** ephemeral key exchange (ECDHE) means that stealing the server's long-term private key later does not decrypt recorded old traffic.

**HSTS:** a header telling browsers to always use HTTPS for a domain (defends against downgrade/SSL-strip).

**Loopholes**

- HTTPS protects the content, not **metadata**: the IP address, SNI hostname (unless ECH), and traffic size and timing are visible.
- The padlock proves the connection to that domain is encrypted, **not that the site is trustworthy**.
- A self-signed or expired certificate breaks authentication; disabling certificate verification in code (`verify=False`) defeats TLS.
- TLS 1.0/1.1 and SSL are obsolete.
- Corporate proxies can perform MITM with their own trusted root.

---

## C8. HTTP status codes

| Class | Meaning | Key codes |
| --- | --- | --- |
| 1xx | informational | 100 Continue, 101 Switching Protocols (WebSocket upgrade) |
| 2xx | success | **200** OK, **201** Created (+ `Location`), **202** Accepted (async), **204** No Content |
| 3xx | redirection | **301** Moved Permanently, **302** Found (temporary), **304** Not Modified, **307** Temporary Redirect and **308** Permanent Redirect (both keep the method) |
| 4xx | client error | **400** Bad Request, **401** Unauthorized (really "unauthenticated"), **403** Forbidden, **404** Not Found, **405** Method Not Allowed, **409** Conflict, **415** Unsupported Media Type, **422** Unprocessable Content (validation), **429** Too Many Requests |
| 5xx | server error | **500** Internal Server Error, **502** Bad Gateway, **503** Service Unavailable, **504** Gateway Timeout |

**Frequently asked comparisons**

- **401 vs 403:** 401 = who are you? (missing/invalid credentials); 403 = I know who you are, and you may not do this.
- **301 vs 302:** permanent (browsers/search engines cache and update) vs temporary. 301/302 may change POST to GET; **307/308 preserve the method**.
- **502 vs 504:** the gateway/proxy received an invalid response from upstream vs timed out waiting for it. **503** = overloaded or in maintenance, often with `Retry-After`.
- **400 vs 422:** malformed request vs syntactically valid but semantically invalid.
- **404 vs 410:** not found vs permanently gone. Some APIs return 404 instead of 403 to avoid revealing that a resource exists.

**Loopholes:** returning 200 with an error message in the body breaks clients, monitoring and caches; 5xx means *your* fault, 4xx the *client's* fault, so retry only 5xx/429 (with backoff); `201` should accompany a `Location` header.

---

## C9. HTTP/1.1 vs HTTP/2 vs HTTP/3

|  | HTTP/1.1 | HTTP/2 | HTTP/3 |
| --- | --- | --- | --- |
| Transport | TCP | TCP (+TLS in practice) | **QUIC over UDP** (TLS 1.3 built in) |
| Format | text | **binary frames** | binary frames |
| Concurrency | one request at a time per connection (6 connections per host in browsers) | **multiplexing** of many streams on one connection | multiplexing with independent streams |
| Header compression | none | HPACK | QPACK |
| Head-of-line blocking | at the HTTP level | solved at HTTP level, **still exists at the TCP level** (one lost packet stalls all streams) | **solved**: loss affects only its own stream |
| Handshake | TCP (+TLS) | TCP + TLS | combined, 1 RTT, 0-RTT resumption |
| Connection migration | no | no | **yes** (connection ID survives IP change, e.g. Wi-Fi to mobile) |

**Other notes:** HTTP/2 server push was largely abandoned; **WebSockets** give a full-duplex, persistent channel after an HTTP upgrade (chat, live updates); **Server-Sent Events** give one-way server-to-client streaming over HTTP; **long polling** is the fallback.

**Loopholes:** HTTP/2 does not reduce latency on a lossy network as much as expected (TCP HOL blocking), which is why HTTP/3 exists; "HTTP/2 requires TLS" is true in browsers, not in the spec; domain sharding is an HTTP/1.1 trick that hurts HTTP/2.

---

## C10. REST API design

**REST constraints:** client-server, **stateless** (each request carries all it needs), **cacheable**, **uniform interface** (resources identified by URIs, manipulated through representations, self-descriptive messages, HATEOAS), layered system, optional code on demand.

**Resource naming:** nouns, plural, hierarchical; the HTTP method is the verb.

```
GET    /users                 list (with pagination, filtering, sorting)
POST   /users                 create -> 201 Created + Location: /users/42
GET    /users/42              read
PUT    /users/42              replace (idempotent)
PATCH  /users/42              partial update
DELETE /users/42              delete -> 204 No Content
GET    /users/42/orders       sub-resource
Avoid: /getUser, /createUser, /users/42/delete
```

**Design checklist:** correct status codes; consistent JSON error format (`{ code, message, details }`); **pagination** (offset/limit, or cursor-based for large or changing data); filtering and sorting through query parameters; **versioning** (`/v1/...` or a header); **idempotency keys** for `POST` payments; authentication (OAuth2/JWT/API keys) and authorization; rate limiting (429 + `Retry-After`); input validation; `ETag` for caching and optimistic concurrency (`If-Match`); consistent naming and date formats (ISO 8601, UTC); documentation (OpenAPI/Swagger).

```python
from flask import Flask, jsonify, request, abort
app = Flask(__name__); users = {}; next_id = 1
@app.post('/users')
def create():
    global next_id
    data = request.get_json(silent=True) or {}
    if 'name' not in data: abort(400, 'name required')
    users[next_id] = {'id': next_id, 'name': data['name']}; next_id += 1
    return jsonify(users[next_id - 1]), 201, {'Location': f'/users/{next_id - 1}'}
@app.get('/users/<int:uid>')
def get(uid):
    return jsonify(users[uid]) if uid in users else abort(404)
```

**REST vs alternatives:** **GraphQL** (client specifies exact fields; one endpoint; avoids over/under-fetching; caching and N+1 are harder), **gRPC** (HTTP/2, Protocol Buffers, strongly typed, streaming; good for service-to-service), **WebSockets** for realtime.

**PUT vs PATCH vs POST:** PUT replaces the full resource and is idempotent; PATCH sends only changes; POST creates or triggers an action and is not idempotent.

**Loopholes:** "REST means JSON over HTTP" is not exact; stateless means no server-side *session* dependency, not that the server stores nothing; do not use `GET` for state changes; returning `200` for everything; exposing internal ids and stack traces; unbounded list endpoints without pagination.

---

## C11. Cookies vs sessions vs JWT

|  | Cookie | Session | JWT |
| --- | --- | --- | --- |
| What | small key-value stored by the browser and sent on every matching request | server-side record keyed by a **session id** (the id is typically carried in a cookie) | self-contained signed token carrying claims |
| State location | client | **server** (memory, Redis, DB) | **client** (server stays stateless) |
| Scaling | trivial | needs a shared store or sticky sessions | easy horizontal scale |
| Revocation | delete cookie | **easy** (delete the session) | **hard** until expiry (needs a denylist or short expiry) |
| Size | up to about 4 KB | tiny id | larger (sent on each request) |

**Cookie attributes:** `HttpOnly` (JavaScript cannot read it, mitigates XSS theft), `Secure` (HTTPS only), `SameSite=Lax/Strict/None` (mitigates CSRF), `Domain`, `Path`, `Max-Age`/`Expires`.

**JWT structure:** `base64url(header).base64url(payload).signature`. Header = algorithm and type; payload = claims (`sub`, `exp`, `iat`, roles); signature = HMAC (HS256) or RSA/ECDSA (RS256/ES256) over header.payload. The server verifies the signature and `exp`. **Signed, not encrypted: anyone can read the payload.**

**Typical secure flow:** short-lived **access token** (5 to 15 minutes) plus a long-lived **refresh token** (stored HttpOnly, rotated, revocable server-side).

**Attacks:** **XSS** (injected script steals tokens; defend with output escaping, CSP, HttpOnly), **CSRF** (the browser automatically sends cookies on a forged cross-site request; defend with SameSite cookies and CSRF tokens; header-based tokens are not sent automatically), session fixation (regenerate session id on login), `alg: none` and weak-secret JWT attacks.

**Loopholes:** never put secrets or PII in a JWT payload; storing a JWT in `localStorage` exposes it to XSS; "JWT is more secure than sessions" is false, they trade server state for revocation difficulty; sessions also need an expiry and secure cookie flags; verify the algorithm explicitly when validating.

---

## C12. What happens when you type a URL and press Enter

```
1. Browser parses the URL (scheme, host, port, path). Checks HSTS list and local caches.
2. DNS: browser cache -> OS cache/hosts -> recursive resolver -> root -> TLD -> authoritative -> IP (cached for TTL).
3. Routing: host is off-subnet, so ARP for the DEFAULT GATEWAY's MAC; frames go to the router, packets are routed hop by hop (NAT at home router).
4. TCP three-way handshake with the server IP on port 443 (or QUIC for HTTP/3).
5. TLS handshake: certificate verified, session keys agreed.
6. HTTP request sent (GET / with Host, cookies, Accept headers).
7. Server side: CDN/edge -> load balancer -> reverse proxy (nginx) -> application server -> cache (Redis) / database -> response built.
8. HTTP response (status, headers, HTML) returns over the same connection.
9. Browser: parse HTML -> build DOM; fetch CSS/JS/images (more requests, reusing the connection with HTTP/2); build CSSOM; combine into the render tree; layout; paint; run JavaScript.
10. Connection kept alive for later requests; resources cached according to Cache-Control.
```

**How to answer well:** give the 10 steps in order, then pick **one layer to go deep** (DNS, TCP/TLS, or load balancing) based on what the interviewer reacts to; mention caching at every level (browser, DNS, CDN, reverse proxy, application cache, database buffer pool).

**Loopholes:** a LAN host ARPs for the **gateway**, not the remote server; HTTPS adds the TLS handshake **after** TCP; DNS may be skipped if cached; modern browsers also preconnect and prefetch; mention HTTP/3 (QUIC) merges transport and TLS handshakes.

---

# PART D: PROGRAMMING AND SYSTEMS BASICS

(OOP, SOLID and design patterns are in Part 4.)

## D1. C++ pointers, references and memory

**Stack vs heap**

|  | Stack | Heap |
| --- | --- | --- |
| Managed by | compiler (automatic) | programmer (`new`/`delete`) or smart pointers |
| Lifetime | until the scope ends | until freed |
| Speed | very fast (move the stack pointer) | slower (allocator, fragmentation) |
| Size | small (about 1 to 8 MB per thread) | large (limited by RAM/virtual memory) |
| Typical bug | stack overflow (deep recursion, huge local arrays) | leak, double free, use-after-free |

**Pointer vs reference**

|  | Pointer `T*` | Reference `T&` |
| --- | --- | --- |
| Can be null | yes (`nullptr`) | no |
| Can be reseated | yes | no (bound for life) |
| Must be initialised | no (but should be) | yes |
| Syntax | `*p`, `p->x`, `&x` | used like the original variable |
| Arithmetic | yes | no |

```cpp
int x = 10; int* p = &x; int& r = x;
*p = 20;  r = 30;                          // both change x
const int* a;   // pointer to const int (cannot change *a, can change a)
int* const b = &x;  // const pointer (cannot change b, can change *b)
const int* const c = &x;                   // neither

void byValue(int v) {}        // copy
void byRef(int& v) { v++; }   // modifies the caller's variable
void byConstRef(const std::string& s) {}   // no copy, read-only: the default for big objects
```

**Arrays decay:** an array name converts to a pointer to its first element when passed to a function (`sizeof` then gives pointer size, so also pass the length, or use `std::array`/`vector`). `a[i]` is `*(a + i)`; pointer arithmetic scales by the element size.

**Classic memory bugs**

- **Memory leak:** `new` without `delete` (use RAII).
- **Dangling pointer / use-after-free:** pointer to freed or out-of-scope memory (returning `&local`).
- **Double free**, **`delete` vs `delete[]` mismatch**, **buffer overflow**, **uninitialised (wild) pointer**, null dereference.
- **Iterator invalidation:** `vector::push_back` may reallocate and invalidate all references and iterators.

```cpp
int* bad() { int local = 5; return &local; }          // dangling: local dies at the return
int* arr = new int[10]; /* ... */ delete[] arr;       // delete[] for new[]; arr = nullptr afterwards
```

**Rule of 0 / 3 / 5:** if a class manages a resource and you define a destructor, copy constructor or copy assignment, you almost always need all three (**rule of 3**); with move semantics add move constructor and move assignment (**rule of 5**). Best: hold resources in members that manage themselves (`vector`, `string`, smart pointers) and define none (**rule of 0**).

**Move semantics:** `std::move` casts to an rvalue reference so a move constructor can **steal** the resource instead of copying (O(1) transfer of a vector's buffer). After a move the source is valid but unspecified.

**Loopholes:** `sizeof(arr)` inside a function is the pointer size; `const` after the `*` vs before it; comparing a `char*` with `==` compares addresses, not text; undefined behaviour (signed overflow, out-of-bounds, use-after-free) may appear to "work" and then fail under optimization; returning a reference to a local is UB.

---

## D2. RAII and smart pointers

**RAII (Resource Acquisition Is Initialization):** tie a resource's lifetime to an object's lifetime: acquire in the constructor, release in the destructor. Destructors run when the scope exits **even on exceptions or early returns**, so leaks cannot happen. Used for memory, files, locks, sockets, DB connections.

```cpp
class File {
    FILE* f;
public:
    explicit File(const char* path) : f(fopen(path, "r")) { if (!f) throw std::runtime_error("open failed"); }
    ~File() { if (f) fclose(f); }                       // always runs
    File(const File&) = delete; File& operator=(const File&) = delete;   // owning type: no copies
};
{ std::lock_guard<std::mutex> g(m); /* critical section */ }               // lock released at scope exit
```

**Smart pointers (`<memory>`)**

| Type | Ownership | Notes |
| --- | --- | --- |
| `unique_ptr<T>` | **exclusive**, movable, not copyable | zero overhead; the default choice |
| `shared_ptr<T>` | **shared** via reference counting (atomic count) | freed when the last owner dies; control block overhead |
| `weak_ptr<T>` | **non-owning** observer of a `shared_ptr` | breaks **reference cycles**; use `lock()` to get a `shared_ptr` or null |

```cpp
auto u = std::make_unique<int>(5);                  // prefer make_unique / make_shared
auto v = std::move(u);                              // ownership moves; u is now null
auto s1 = std::make_shared<std::string>("hi");
auto s2 = s1;                                       // use_count() == 2

struct Node { std::shared_ptr<Node> next; std::weak_ptr<Node> prev; };   // weak_ptr avoids the cycle leak
```

**Loopholes:** two `shared_ptr`s pointing at each other **leak** (use `weak_ptr` for back-references); never create two `shared_ptr`s from the same raw pointer (double delete); `shared_ptr`'s count is atomic, but the pointed-to object is **not** thread-safe; passing `shared_ptr` by value everywhere adds atomic overhead (pass `const&` or raw reference when not sharing ownership); `unique_ptr<T[]>` for arrays; `auto_ptr` is removed.

---

## D3. STL containers, complexities and iterator rules

| Container | Access | Insert/erase | Notes |
| --- | --- | --- | --- |
| `vector` | O(1) index | O(1) amortized at end, O(n) middle | contiguous, cache friendly; the default |
| `deque` | O(1) | O(1) both ends | chunked; no contiguous guarantee |
| `list` | O(n) | O(1) with an iterator | poor cache behaviour, rarely best |
| `array<T,N>` | O(1) | n/a | fixed size, on the stack |
| `set`/`map` | O(log n) | O(log n) | red-black tree, sorted order |
| `unordered_set`/`unordered_map` | O(1) avg | O(1) avg | hash table, O(n) worst case |
| `priority_queue` | top O(1) | push/pop O(log n) | binary heap |
| `stack`/`queue` | adapters on `deque` | O(1) |  |

**Iterator invalidation (interview favourite):** `vector`: insert/erase/`push_back` that reallocates invalidates **all**; erase invalidates from the erased position onward. `map`/`set`/`list`: erase invalidates only the erased element. `unordered_map`: **rehash** invalidates iterators (not references). Erase while iterating: `it = c.erase(it)`.

**Useful algorithms:** `sort`, `stable_sort`, `nth_element`, `lower_bound`, `accumulate`, `unique` + `erase`, `reverse`, `rotate`, `next_permutation`, `min_element`, `count_if`, `any_of`, `transform`, `iota`.

---

## D4. Linux essentials for developers

**Filesystem hierarchy:** `/` root, `/home` user files, `/etc` configuration, `/var` logs and variable data, `/tmp`, `/usr` programs, `/bin` essential binaries, `/dev` devices, `/proc` kernel and process information, `/opt` optional software. **Everything is a file** (devices, sockets, pipes).

**Permissions:** `r=4, w=2, x=1` for user/group/other. `chmod 755 f` = `rwxr-xr-x`; `644` = `rw-r--r--`; `chmod +x script.sh`; `chown user:group f`. Directories need `x` to be entered. `setuid` makes a program run with its owner's privileges (security-sensitive).

**Commands you must know**

| Task | Command |
| --- | --- |
| list / navigate | `ls -la`, `cd`, `pwd`, `tree` |
| view files | `cat`, `less`, `head -n 20`, `tail -f app.log` (follow live) |
| search text | `grep -rn 'error' .`, `grep -i -E 'a\|b'`, `rg` |
| find files | `find . -name '*.log' -mtime -1` |
| text processing | `awk '{print $1}'`, `sed 's/old/new/g'`, `sort \| uniq -c \| sort -nr`, `cut -d, -f2`, `wc -l` |
| processes | `ps aux`, `top`/`htop`, `kill -9 PID`, `pkill name`, `jobs`, `bg`, `fg`, `nohup cmd &` |
| disk and memory | `df -h`, `du -sh *`, `free -m`, `vmstat` |
| network | `curl -v`, `ping`, `traceroute`, `ss -tulpn` (listening ports), `netstat`, `lsof -i :8080`, `dig`, `ip addr` |
| archives | `tar -czvf a.tgz dir/`, `tar -xzvf a.tgz`, `zip`, `unzip` |
| package/services | `apt install`, `systemctl status/restart nginx`, `journalctl -u nginx -f` |
| remote | `ssh user@host`, `scp`, `rsync -av` |

**Pipes and redirection:** `cmd1 | cmd2` (stdout to stdin), `> file` (overwrite), `>> file` (append), `2>&1` (stderr to stdout), `< file`, `/dev/null` (discard). Exit status `$?` (0 = success), `&&` / `||` chain on success/failure.

**Signals:** `SIGINT` (2, Ctrl+C), `SIGTERM` (15, polite stop, can be handled for graceful shutdown), `SIGKILL` (9, cannot be caught), `SIGSTOP`, `SIGHUP`, `SIGSEGV` (invalid memory access).

**Hard link vs soft (symbolic) link:** a hard link is another directory entry to the same **inode** (survives deletion of the original name, same filesystem only); a symlink is a small file holding a path (can dangle, can cross filesystems).

**Small shell script**

```bash
#!/usr/bin/env bash
set -euo pipefail                      # stop on errors, undefined variables and pipe failures
logfile=${1:?usage: $0 logfile}
grep ERROR "$logfile" | awk '{print $4}' | sort | uniq -c | sort -nr | head -5   # top 5 error sources
```

**Debugging a slow server (standard story):** `uptime`/`top` (load, CPU), `free -m` (memory, swap), `vmstat 1` (swap activity), `iostat`/`df` (disk), `ss -s` (connections), app logs (`journalctl`), then profile.

**Loopholes:** `rm -rf` has no undo (double-check variables like `rm -rf $DIR/`); quote variables in shell scripts (spaces); `kill -9` skips cleanup so try `SIGTERM` first; `cat file | grep` is an unnecessary use of `cat`; `>` silently overwrites; a file's permission bits do not apply to root.

---

## D5. Compilation, linking and debugging

```
source.cpp -> [Preprocessor] -> expanded source (#include, #define, #ifdef)
           -> [Compiler]     -> assembly
           -> [Assembler]    -> object file (.o)   (machine code with unresolved symbols)
           -> [Linker]       -> executable (resolves symbols, joins libraries)
           -> [Loader/OS]    -> process in memory
```

- **Static linking:** library code copied into the executable (bigger, self-contained). **Dynamic linking:** shared libraries (`.so`, `.dll`) loaded at run time (smaller binaries, shared memory, updatable, but dependency and version issues).
- **Compiler vs interpreter vs JIT:** ahead-of-time translation to machine code (C++, Rust, Go); line-by-line execution (classic Python bytecode VM); **JIT** compiles hot code at run time (Java HotSpot, JavaScript V8).
- **Header guards / `#pragma once`** prevent double inclusion; the **One Definition Rule** forbids multiple definitions of the same function across translation units (put definitions in `.cpp`, or mark `inline`).
- **Typical errors:** *compile error* (syntax, type), *undefined reference* (**linker** error: declared but not defined or missing library), *segmentation fault* (runtime invalid memory access), *stack overflow*.
- **Tools:** `g++ -O2 -Wall -Wextra -g`, **gdb** (breakpoints, backtrace `bt`), **Valgrind** and **sanitizers** (`-fsanitize=address,undefined`) to catch leaks and memory errors, **profilers** (`perf`, `gprof`).
- **Build systems:** Make, CMake, Maven/Gradle, npm.

**Loopholes:** optimization levels can change the behaviour of programs that contain undefined behaviour; debug builds are much slower than release builds, so never benchmark `-O0`; a link error is not a compile error; templates are instantiated at compile time (code bloat).

---

## D6. Computer architecture essentials

**Instruction cycle:** fetch → decode → execute → memory access → write back. **Pipelining** overlaps stages for throughput; **hazards**: structural, data (a value not ready), control (branches); solved by forwarding, stalls, and **branch prediction**.

**Memory hierarchy (approximate latency)**

```
Registers   < 1 ns      (a few hundred bytes)
L1 cache    ~1 ns       (32-64 KB per core)
L2 cache    ~4 ns       (256 KB - 1 MB per core)
L3 cache    ~15 ns      (shared, MBs)
RAM         ~100 ns     (GBs)
SSD         ~100 us
HDD         ~10 ms
Network round trip (same datacenter) ~0.5 ms; cross-continent ~100+ ms
```

**Locality:** *temporal* (reuse soon) and *spatial* (neighbours soon); caches load a whole **cache line** (64 bytes). This is why **arrays beat linked lists** in practice and why traversal order matters:

```cpp
// Row-major C++ arrays: the inner loop over the LAST index touches consecutive memory.
long long sumRows(const std::vector<std::vector<int>>& m) {
    long long s = 0;
    for (size_t i = 0; i < m.size(); i++) for (size_t j = 0; j < m[i].size(); j++) s += m[i][j];   // fast
    return s;
}
// Swapping the loops (column-wise) strides across memory, causing cache misses: often several times slower.
```

**Cache terms:** hit/miss, miss types (compulsory, capacity, conflict), write-through vs write-back, associativity, **false sharing** (two threads write different variables on the same cache line and keep invalidating each other; fix with padding/`alignas(64)`).

**Other facts**

- **Endianness:** little-endian (x86/ARM typically) stores the least significant byte first; network byte order is big-endian (`htons`, `htonl`).
- **Two's complement:** negative numbers are stored as `~x + 1`; an n-bit signed int ranges from `-2^(n-1)` to `2^(n-1) - 1`; `INT_MIN` has no positive counterpart.
- **Floating point (IEEE 754):** binary fractions cannot represent 0.1 exactly: `0.1 + 0.2 == 0.30000000000000004` in Python; compare with a tolerance; never use floats for money (use integers of cents or `Decimal`).
- **RISC vs CISC:** simple fixed-length instructions (ARM, RISC-V) vs complex variable-length (x86).
- **Multicore, SIMD/vectorization, GPUs** for data-parallel work; **branch prediction** is why a sorted array can be processed faster than a shuffled one in a loop with an `if`.
- **Amdahl's law:** `Speedup = 1 / ((1 - p) + p / s)` where `p` is the parallelizable fraction and `s` the speedup of that part. With p = 0.9 and s = 10: `1 / (0.1 + 0.09) = 5.26`; the serial 10% caps speedup at 10x no matter how many cores.

```cpp
bool isLittleEndian() { uint32_t x = 1; return *reinterpret_cast<unsigned char*>(&x) == 1; }
```

**Loopholes:** "more cores = proportionally faster" ignores Amdahl and synchronization; big-O ignores cache effects (a contiguous O(n) scan can beat an O(log n) pointer-chasing structure for small n); `float` has about 7 decimal digits, `double` about 15.

---

# PART E: RAPID-FIRE REVISION SHEET

Read this the night before an interview. Each line is a complete answer.

**Operating systems**

- Process = program in execution with its own address space; thread = unit of execution sharing the process's heap/code/globals with its own stack and registers.
- Context switch = save/restore registers, PC and page-table base; overhead; thread switches are cheaper.
- Scheduling formulas: TAT = CT - AT; WT = TAT - BT; response = first run - AT. SJF minimizes average waiting but starves long jobs; aging fixes starvation; RR needs a good quantum.
- Race condition = unsynchronized shared access; fix with mutex/atomics. Mutex has an owner (locking); semaphore is a counter (signalling).
- Deadlock = mutual exclusion + hold-and-wait + no preemption + circular wait; break any one (lock ordering). Banker's algorithm = avoidance via safe state.
- Paging removes external fragmentation; TLB caches translations; page fault = trap to load a page from disk; EAT = (1-p)*mem + p*fault\_time.
- Replacement: FIFO (Belady's anomaly), LRU (good, stack algorithm), OPT (theoretical). Thrashing = more paging than work; cut multiprogramming or use working-set.
- System call = trap from user to kernel mode. Zombie = finished, not reaped; orphan = parent died.
- CAS = atomic compare-and-set, basis of lock-free structures; ABA problem needs version tags. TOCTOU = non-atomic check then use.

**DBMS**

- ACID: atomicity (undo/WAL), consistency (constraints), isolation (locks/MVCC), durability (redo log, fsync).
- Keys: super key (unique), candidate key (minimal super key), primary key (chosen), foreign key (reference).
- 1NF atomic; 2NF no partial dependency; 3NF no transitive dependency; BCNF every determinant is a super key.
- Index = B+ tree, O(log n), leaves linked for ranges; clustered = rows stored in index order; leftmost-prefix rule; functions on a column kill the index.
- Isolation: dirty read, non-repeatable read, phantom, write skew; Read Committed / Repeatable Read / Serializable; MVCC means readers don't block writers.
- Conflict serializable iff the precedence graph is acyclic; 2PL guarantees it but can deadlock; DB resolves deadlocks by aborting a victim, and the app must retry.
- Joins: nested loop O(nm), hash join O(n+m) equality only, sort-merge for sorted inputs; read EXPLAIN; keep predicates sargable; avoid N+1 and large OFFSET.
- SQL for transactions and relations; NoSQL for scale, flexible schema and simple access patterns; pick by access pattern.

**Networks**

- OSI: Physical, Data link, Network, Transport, Session, Presentation, Application. TCP/IP has 4 layers.
- TCP: reliable, ordered, 3-way handshake, flow control (rwnd), congestion control (slow start, AIMD). UDP: connectionless, fast, no guarantees. TCP is a byte stream, so frame your messages.
- Subnet: hosts = 2^(32-p) - 2; network = IP AND mask; broadcast = network OR inverse mask; /26 = 64 addresses, 62 hosts.
- ARP = IP to MAC in the LAN (use the gateway for remote hosts). DHCP = DORA. DNS = recursive resolver asks root, TLD, authoritative; cached by TTL.
- HTTP is stateless; GET safe and idempotent, POST neither, PUT/DELETE idempotent. 401 = unauthenticated, 403 = forbidden, 404 not found, 429 rate limited, 502/503/504 gateway/overload/timeout.
- HTTPS = HTTP over TLS: certificate chain authenticates the server, ECDHE agrees keys, AES encrypts. TLS 1.3 needs 1 RTT.
- HTTP/2 multiplexes on TCP; HTTP/3 uses QUIC over UDP to remove TCP head-of-line blocking.
- JWT is signed, not encrypted; sessions are easy to revoke; cookies need HttpOnly, Secure, SameSite.
- URL typed: parse, DNS, TCP, TLS, HTTP request, server stack (LB, app, cache, DB), response, render.

**C++ and systems**

- Use `unique_ptr`/`shared_ptr`/RAII, not raw `new`/`delete`; `weak_ptr` breaks cycles; rule of 0.
- Stack is automatic and small; heap is manual and large. Dangling pointer, leak, double free, iterator invalidation are the standard bugs.
- Compile = preprocess, compile, assemble, link; "undefined reference" is a linker error.
- Arrays beat linked lists because of cache lines and spatial locality. False sharing hurts multithreaded counters.
- Floats are inexact: `0.1 + 0.2 != 0.3`; use integers for money.

---

*End of Part 2. Part 3 covers SQL query practice and Git, and Part 4 covers OOP, SOLID, design patterns and low-level design.*
