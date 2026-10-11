# F002: Processes vs threads

**Weekend / Slot:** W01 · Sunday (2026-10-11)
**Phase / Block:** 1 Learning · A Operating Systems
**Time budget:** 2 hours

**Topics (curriculum):** Address space sharing, user vs kernel threads, thread pools, cost of a thread, virtual threads, multiprocessing vs multithreading

**Drill:** Run the same workload as processes and as threads and measure both.

**Done when:** You can say exactly what is shared and what is copied between the two.

> Goal of the day: yesterday you saw what `fork()` copies (page tables, fd table) and shares (physical frames, open file descriptions). Today you find out what a thread deliberately *doesn't* copy, why that makes it cheaper and more dangerous, and what Java does about the cost (pools, then virtual threads).

---

## 0. How to use this file

- Same rules as Day 1: **write your prediction first**, then open the `<details>` block. Note what you got wrong; your end-of-day note needs one such thing.
- Supporting material (not in the curriculum topic list) is marked **[Supporting]**.
- Answer interview questions out loud, claim → mechanism → example → trade-off.
- Results of the benchmarks depend on your machine. **The shape of the result matters, not the exact number.** Do not trust any number in this file as a promise; measure.

### The 2 hours

| Chunk | Min | What | Level |
|-------|-----|------|-------|
| W | 5 | Warm-up: recall Day 1 | Understand |
| 1 | 20 | Address space sharing: what a thread shares and what it owns | Understand |
| 2 | 15 | User vs kernel threads, cost of a thread | Understand |
| 3 | 20 | Thread pools, virtual threads, multiprocessing vs multithreading | Understand → Defend |
| 4 | 40 | **The Drill**: measure processes vs threads | Reproduce → Diagnose |
| 5 | 20 | Interview drill, checkpoint, written note | Defend |

---

## 1. Setup (do this first, 5 min, outside the clock if needed)

Same Linux container as Day 1, but now with a mounted folder so your code and results survive between runs, and a JDK for the Java part.

```bash
mkdir -p ~/fundamentals-lab
docker run --rm -it --cap-add=SYS_PTRACE -v "$HOME/fundamentals-lab":/work -w /work ubuntu:24.04 bash
# inside the container:
apt-get update && apt-get install -y gcc strace procps openjdk-21-jdk-headless
nproc            # how many CPUs does the container see? Write it down: it decides your results
```

- `nproc` reports the CPUs available to the Docker VM, which may be fewer than your Mac has (Docker Desktop setting). Use that number wherever "cores" appears below.
- Virtual threads need **JDK 21 or newer**. Check with `java -version`.
- Put every file below in `/work` (that's `~/fundamentals-lab` on your Mac).

---

## 2. Warm-up + Chunk 1: Address space sharing (5 + 20 min)

### Warm-up (5 min, from memory, no scrolling)
1. What does `fork()` copy, and what stays shared until a write?
2. Why can a zombie exist and why can't you `kill -9` it?
3. What is a system call, in one sentence?

Anything you missed goes on the gap list at the bottom of this file.

### The problem
Processes are isolated, which is great for safety and terrible for cooperation. Two processes that want to work on the same data need explicit IPC (pipes, shared memory, sockets: F009), and creating one means cloning an address space. For a web server handling thousands of concurrent requests that all need the same cache and config, that's too heavy. We want **many independent lines of execution inside one address space**. That's a **thread**.

### Mental model
A process is a **container of resources**. A thread is a **line of execution** inside it.

```text
ONE PROCESS, THREE THREADS

 ┌──────────────────────── Process (one address space) ───────────────────────┐
 │  SHARED by all threads                                                     │
 │    code (text) · globals/static data · heap · open file descriptors        │
 │    signal handlers · PID · working directory · user/group IDs              │
 │                                                                            │
 │   Thread 1             Thread 2             Thread 3                       │
 │   ┌──────────────┐     ┌──────────────┐     ┌──────────────┐               │
 │   │ own stack    │     │ own stack    │     │ own stack    │               │
 │   │ own registers│     │ own registers│     │ own registers│               │
 │   │ own PC       │     │ own PC       │     │ own PC       │               │
 │   │ own TID, TLS │     │ own TID, TLS │     │ own TID, TLS │               │
 │   └──────────────┘     └──────────────┘     └──────────────┘               │
 └────────────────────────────────────────────────────────────────────────────┘
```

Contrast with processes after `fork()` (Day 1):

```text
 Process A: own page tables ──┐
                              ├──► same physical frames, read-only COW, until a write
 Process B: own page tables ──┘
 Separate address spaces (same virtual addresses, different memory after a write).
```

### The split, precisely

| | Threads of one process | Two processes (after `fork`) |
|---|------------------------|------------------------------|
| Address space (code, globals, heap) | **Shared** (same memory) | **Copied logically** (page tables copied; frames shared COW, private on write) |
| Stack | **Private** per thread | Private (each has its own) |
| Registers / program counter | **Private** per thread | Private |
| Open file descriptors | **Shared** (one fd table) | Fd table **copied**; entries point to the **same open file description** (shared offset) |
| Signal handlers (dispositions) | **Shared** | Copied |
| Signal mask | Private per thread | Inherited copy |
| PID | **Same** (the "thread group id") | Different |
| Thread ID (TID) | Different per thread | n/a (TID = PID for a single-threaded process) |
| Thread-local storage, `errno` | Private per thread | Private |
| Scheduling state (ready/running/blocked) | **Per thread**: the kernel schedules threads | Per process |

The single most important consequence: **threads communicate by just reading and writing the same memory.** That is the speed advantage and also the source of every race condition (F004).

### Linux detail: a thread is a task created with different `clone()` flags
In Linux the kernel doesn't have separate "process" and "thread" objects. Both are a **task** (`task_struct`). You saw `clone`/`clone3` in your Day 1 strace. The flags decide what is shared:

- `fork()` ≈ `clone` **without** `CLONE_VM` (new address space copy-on-write), with `SIGCHLD` so the parent is notified on exit.
- `pthread_create()` ≈ `clone` **with** `CLONE_VM | CLONE_FS | CLONE_FILES | CLONE_SIGHAND | CLONE_THREAD …`. Share memory, fd table, signal handlers, and join the same thread group.

You will see this in your own `strace` in the Drill. `getpid()` returns the thread-group id (shared); `gettid()` returns the task's own id.

### Prediction 1 (write before opening)
Two threads of the same process. For each item, shared or private?
(a) a global variable (b) a local variable inside a function (c) an object allocated with `new`/`malloc` (d) the program counter (e) an open socket (f) `errno` (g) the signal handler table

<details>
<summary>Answer</summary>

(a) shared (b) **private**: it lives on that thread's own stack (but see the caveat) (c) shared: it's on the heap, so any thread holding the pointer can touch it (d) private (e) shared: one fd table per process (f) private: `errno` is thread-local, else every syscall error would race (g) shared.

Caveat on (b): stacks are *private by convention, not protected by hardware*. All stacks live in the same address space. If thread A hands thread B a pointer to one of its locals, B can read it and corrupt it. In Java the equivalent is escape: a local reference to a shared object isn't private at all; only the *reference*, not the object, is on the stack.

</details>

---

## 3. Chunk 2: User vs kernel threads, cost of a thread (15 min)

### The problem
Who knows the thread exists? There are two answers with very different consequences.

| | Kernel-level thread | User-level thread |
|---|---------------------|-------------------|
| Managed by | The kernel scheduler | A library/runtime in user space |
| Kernel knows about it? | Yes | No: the kernel sees one thread (the carrier) |
| Switch cost | Needs a kernel entry (context switch) | Just swap registers in user space: very cheap |
| Blocking syscall | Blocks **only that thread** | May block **the whole process** if the kernel sees one thread |
| Multicore parallelism | **Yes**, kernel puts threads on different cores | Not by itself (one kernel thread = one core) |

### Mapping models

```text
Many-to-one   many user threads ──► 1 kernel thread     (cheap, no parallelism, one block stalls all)
One-to-one    1 user thread     ──► 1 kernel thread     (Linux pthreads, Java platform threads)
Many-to-many  M user threads    ──► N kernel threads    (Go goroutines, Java virtual threads)
```

- **Java platform threads are one-to-one with OS threads**: `new Thread(...).start()` ends in a `clone` syscall (you'll see the thread appear in `ps -L`). A Java thread blocked on a socket read is a *kernel thread* blocked in a syscall: the Day 1 "Waiting" state.
- **Virtual threads** (chunk 3) are many-to-many.

### The cost of a thread (what you actually pay)

| Cost | Where it shows up |
|------|-------------------|
| **Stack memory**: reserved virtual address space per thread (glibc default = `ulimit -s`, typically 8 MB; Java default `-Xss` = 1 MB on 64-bit Linux). Committed lazily as it's touched. | `VmSize` jumps, `VmRSS` barely does |
| **Kernel memory**: a `task_struct` plus a kernel stack (16 KB on x86-64) per thread | Not in the Java heap; invisible to `-Xmx` |
| **Creation**: a syscall, allocate stack + kernel structures. Cheaper than `fork` (no page-table or fd-table copy). | Measured in the Drill |
| **Scheduling**: more runnable threads than cores → more context switches, worse cache behaviour (Day 1 indirect cost) | `vmstat 1` `cs` column |
| **Hard limits**: `ulimit -u`, `/proc/sys/kernel/threads-max`, `pid_max`, cgroup `pids.max`, and native memory | `OutOfMemoryError: unable to create native thread` |

Threads vs context switch (link to Day 1): a switch **between threads of the same process** skips the address-space switch, so the TLB entries and much of the cache stay useful. That's why it's cheaper than a process switch. It is not free: still a kernel entry, register save/restore, and a scheduling decision.

### Prediction 2
A Java service has 50 platform threads. One of them calls `socket.getInputStream().read()` and no data has arrived. (a) What state is that thread in, and does it use CPU? (b) What happens to the other 49? (c) What would be different under a **many-to-one user-level** model?

<details>
<summary>Answer</summary>

(a) Blocked in the `read`/`recv` syscall, kernel state Waiting (Day 1 lifecycle). It uses no CPU. (b) Nothing: they're separate kernel threads and keep running on other cores. (c) With many-to-one, the kernel only sees one thread; the blocking syscall blocks that kernel thread and therefore **every user thread multiplexed on it**. This is the classic reason pure many-to-one died out, and why runtimes using M:N (Go, Java virtual threads) take care to turn blocking calls into non-blocking ones (`epoll`, F009) underneath.

</details>

---

## 4. Chunk 3: Thread pools, virtual threads, multiprocessing vs multithreading (20 min)

### Thread pools: the problem
Spawning a thread per task has two problems: creation cost per task (even if small), and, much worse, **no upper bound**. 50,000 simultaneous requests → 50,000 threads → memory exhaustion, scheduler thrash, `unable to create native thread`. A **pool** keeps a fixed set of reusable worker threads and queues the rest. It converts "unbounded concurrency" into "bounded concurrency + a queue", which is a deliberate back-pressure point.

```text
 tasks ──► [ queue ] ──► worker 1 ┐
                         worker 2 ├─ bounded set of long-lived threads
                         worker N ┘
```

### Sizing a pool (a reasoned starting point, then measure)
- **CPU-bound** tasks: about `N` threads (≈ number of cores, maybe `N+1`). More just adds context switching.
- **I/O-bound** tasks: threads spend most of their time waiting, so you can have more. Rule of thumb: `threads ≈ N × (1 + W/C)` where `W` = wait time and `C` = compute time per task.
- Cross-check with **Little's Law**: `concurrent requests in flight = arrival rate × average latency`. 150 req/s at 400 ms average → 60 threads busy on average.
- Tomcat's default max worker threads is 200 (`server.tomcat.threads.max`). That's a *pool*, and when all 200 are blocked on a slow DB, requests queue and latency explodes while CPU sits near idle. You'll see this again in F017 (connection pools).

### `ThreadPoolExecutor` pitfall (interview classic)
Parameters: `corePoolSize`, `maximumPoolSize`, a **work queue**, a rejection policy. The order of events surprises people:

1. Fewer than `core` threads → start a new thread.
2. Otherwise → **put the task on the queue**.
3. Only if the queue is **full** → start extra threads up to `max`.
4. If that's also full → rejection policy.

### Prediction 3
`new ThreadPoolExecutor(2, 10, 60, SECONDS, new LinkedBlockingQueue<>())` and you submit 100 slow tasks at once. How many threads run? And `Executors.newFixedThreadPool(2)` after 10 million queued tasks?

<details>
<summary>Answer</summary>

Two threads. `LinkedBlockingQueue<>()` with no capacity is **unbounded**, so the queue never fills, step 3 never happens, and `maximumPoolSize` is dead configuration. `newFixedThreadPool` uses the same unbounded queue, so queued tasks accumulate on the heap → eventually `OutOfMemoryError`. Fix: a **bounded** queue + an explicit rejection policy, so overload becomes a visible failure (shed load, 429/503) rather than a slow memory leak.

</details>

### Virtual threads (Java 21, JEP 444)

**Problem:** a thread-per-request design is the easiest to write (plain blocking code) but platform threads are expensive, so you can't have a million. Async/reactive code scales but is harder to write and debug.

**Idea:** make threads cheap enough that you can have one per task again, so blocking code scales.

```text
 virtual threads (millions, live on the heap)
 VT1  VT2  VT3  VT4  VT5  ...
   \   |    |   /
   ┌───▼────▼──▼──────────────┐
   │ carrier threads          │  ← platform threads (a ForkJoinPool, ≈ #cores by default)
   └──────────────────────────┘
        └──► OS schedules these as normal kernel threads
```

How it works:
1. A virtual thread is **mounted** on a carrier thread and runs.
2. When it hits a blocking operation the JDK knows about (`Thread.sleep`, socket I/O, `java.util.concurrent` locks…) it is **unmounted**: its stack is copied to the heap, the carrier is freed for another virtual thread.
3. When the operation can proceed (underneath, the JDK uses non-blocking I/O such as `epoll`), it's scheduled onto a carrier again.

Rules and limits:
- **Great for I/O-bound** work with lots of waiting. **No benefit for CPU-bound** work: the cores are the limit.
- **Don't pool virtual threads.** They're cheap; one per task. To limit concurrency against a scarce resource (DB, downstream API), use a `Semaphore`. Removing your 200-thread pool also removed an implicit throttle, so your *connection pool* becomes the new bottleneck.
- **Pinning:** a virtual thread can be stuck to its carrier while blocked, notably inside `synchronized` (before JDK 24, JEP 491 fixed most of this) or in native calls. A few pinned threads are fine; many can starve the carriers. Diagnose with the JFR event `jdk.VirtualThreadPinned`.
- Some operations (certain file I/O, native calls) still block a carrier; the JDK compensates but it's not magic.
- Thread-locals still work but get expensive at a million threads.
- Spring Boot 3.2+: `spring.threads.virtual.enabled=true`.

### Multiprocessing vs multithreading

| | Multiprocessing | Multithreading |
|---|-----------------|----------------|
| Isolation | **Strong**: a crash/leak in one doesn't corrupt others | **None**: one bad pointer / `SIGSEGV` / OOM-kill takes down all threads |
| Communication | Explicit IPC (pipes, sockets, shared memory) | Just read/write shared memory |
| Creation cost | Higher (new address space, page tables) | Lower |
| Memory | Separate (COW softens it) | Shared |
| Bugs | Fewer shared-state bugs | Races, deadlocks (F004, F005) |
| Parallelism | Yes | Yes (except runtime limits, e.g. CPython's GIL in the default build **[Supporting]**) |

Real systems (the choice is a design decision, not a default):
- **PostgreSQL**: process per connection. Strong isolation (a crashing backend doesn't take down others) at the cost of memory per connection and context switching. That is exactly why you put **PgBouncer / a pool** in front when you have thousands of clients (F017).
- **nginx**: a few worker processes, each running an event loop (F009).
- **Chrome**: process per site/tab for isolation and security.
- **Tomcat/Spring Boot**: thread pool inside one JVM process.
- **Python CPU-bound work**: `multiprocessing` to get past the GIL. **[Supporting]**

---

## 5. Chunk 4: THE DRILL (40 min)

**Drill:** Run the same workload as processes and as threads, and measure both.

You'll run four small experiments: a CPU-bound workload, a creation-cost test, a "what's shared" probe, and a deliberate race. Then a Java experiment comparing platform and virtual threads. Put all files in `/work`.

### Step 1: CPU-bound, same job as N processes vs N threads (10 min)

`bench.c`: each worker does the *same fixed job* (counting primes below 2,000,000), so doubling workers doubles total work. Perfect parallelism means wall time stays flat until you run out of cores.

```c
// gcc -O2 -pthread -o bench bench.c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <time.h>
#include <unistd.h>
#include <pthread.h>
#include <sys/wait.h>

#define LIMIT 2000000L

volatile long sink;                     // stops the compiler deleting the "unused" work

static double now(void) {
    struct timespec ts;
    clock_gettime(CLOCK_MONOTONIC, &ts);
    return ts.tv_sec + ts.tv_nsec / 1e9;
}

static long count_primes(long lo, long hi) {
    long c = 0;
    for (long n = lo; n < hi; n++) {
        if (n < 2) continue;
        int prime = 1;
        for (long d = 2; d * d <= n; d++)
            if (n % d == 0) { prime = 0; break; }
        c += prime;
    }
    return c;
}

static void *thread_main(void *arg) {
    sink = count_primes(0, LIMIT);
    return arg;
}

int main(int argc, char **argv) {
    if (argc != 3) { fprintf(stderr, "usage: %s proc|thread N\n", argv[0]); return 1; }
    int n = atoi(argv[2]);
    double t0 = now();
    if (strcmp(argv[1], "thread") == 0) {
        pthread_t *tid = malloc(n * sizeof *tid);
        for (int i = 0; i < n; i++) pthread_create(&tid[i], NULL, thread_main, NULL);
        for (int i = 0; i < n; i++) pthread_join(tid[i], NULL);
    } else {
        for (int i = 0; i < n; i++)
            if (fork() == 0) { sink = count_primes(0, LIMIT); _exit(0); }
        for (int i = 0; i < n; i++) wait(NULL);
    }
    printf("%-6s x%-3d wall = %.3f s\n", argv[1], n, now() - t0);
    return 0;
}
```

```bash
gcc -O2 -pthread -o bench bench.c
./bench thread 1                         # baseline: how long is one worker?
for n in 1 2 4 8; do ./bench proc $n; ./bench thread $n; done
time ./bench thread 4                    # compare real vs user
```
If `nproc` is 4, use `1 2 4 8 16`; the interesting points are around and above your core count. If one worker takes under ~0.3 s, raise `LIMIT`.

**Prediction 4 (write before running):** Let C = `nproc`. (a) For N ≤ C, will threads or processes finish faster? (b) What happens to wall time once N > C? (c) In `time ./bench thread 4` on a 4-core box, how do `real` and `user` compare?

<details>
<summary>Answer</summary>

(a) **About the same.** The work is pure computation inside one address space; no sharing, no syscalls. Creation overhead is microseconds against hundreds of milliseconds of work. (If you saw a big gap, the job is too short.) (b) Wall time grows roughly linearly: with N workers on C cores the total work is N units on C cores, so wall ≈ (N / C) × single-worker time, plus context switching. Extra workers don't add speed, they only take turns. (c) `user` ≈ C × `real` when parallelism works (4 cores busy for `real` seconds ≈ 4 × `real` of CPU time). If `user` ≈ `real`, your threads ran serially: check `nproc`.

</details>

Fill in your results:

| N | proc wall (s) | thread wall (s) |
|---|---------------|-----------------|
| 1 | | |
| 2 | | |
| 4 | | |
| 8 | | |
| | | |

**Takeaway to write down:** for CPU-bound work, processes vs threads is *not* a performance question. It's an isolation-vs-sharing question.

### Step 2: creation cost, where they actually differ (5 min)

`spawn.c`: create-and-wait for 10,000 do-nothing workers, once with threads, once with processes, optionally with a fat parent.

```c
// gcc -O2 -pthread -o spawn spawn.c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <time.h>
#include <unistd.h>
#include <pthread.h>
#include <sys/wait.h>

char *big;                                   // global, so the compiler can't drop the allocation

static double now(void) {
    struct timespec ts;
    clock_gettime(CLOCK_MONOTONIC, &ts);
    return ts.tv_sec + ts.tv_nsec / 1e9;
}
static void *noop(void *a) { return a; }

int main(int argc, char **argv) {
    int n = 10000;
    size_t mb = argc > 2 ? (size_t)atol(argv[2]) : 0;
    if (mb) { big = malloc(mb << 20); memset(big, 1, mb << 20); }   // make the parent "fat"
    double t0 = now();
    if (strcmp(argv[1], "thread") == 0) {
        for (int i = 0; i < n; i++) {
            pthread_t t;
            pthread_create(&t, NULL, noop, NULL);
            pthread_join(t, NULL);
        }
    } else {
        for (int i = 0; i < n; i++) {
            pid_t p = fork();
            if (p == 0) _exit(0);
            waitpid(p, NULL, 0);
        }
    }
    printf("%-6s parent=%zu MB: %.1f us per create+join\n",
           argv[1], mb, (now() - t0) / n * 1e6);
    return 0;
}
```

```bash
gcc -O2 -pthread -o spawn spawn.c
./spawn thread
./spawn proc
./spawn thread 256       # fat parent: 256 MB touched
./spawn proc 256
```

**Prediction 5:** (a) Which is cheaper per create+join, thread or process? (b) What happens to each when the parent holds 256 MB? Why (use Day 1: what does `fork` copy up front?).

<details>
<summary>Answer</summary>

(a) Threads, typically several times cheaper. A thread needs a stack and a task; a fork also needs a new address space (copy of page tables), fd table, etc. (b) **Thread cost stays about the same; fork cost goes up with parent size.** `fork` copies the page tables, and more mapped memory means more page-table entries to copy and mark copy-on-write, even though no data pages are copied. This is also why very large JVMs historically hurt on `fork`-based subprocess spawning. If the difference in your run is small, that's fine: record the ratio and the trend.

</details>

### Step 3: what's actually shared (10 min)

`shared.c` prints identity and addresses from the main thread, a new thread, and a forked child.

```c
// gcc -O2 -pthread -o shared shared.c
#define _GNU_SOURCE
#include <stdio.h>
#include <unistd.h>
#include <pthread.h>
#include <sys/wait.h>
#include <sys/syscall.h>

int counter = 0;

static void report(const char *who) {
    int local = 0;
    printf("%-7s pid=%d tid=%ld  &counter=%p  &local=%p  counter=%d\n",
           who, getpid(), (long)syscall(SYS_gettid),
           (void *)&counter, (void *)&local, counter);
    fflush(stdout);
}

static void *thread_main(void *arg) {
    counter += 100;
    report("thread");
    return arg;
}

int main(void) {
    report("main");
    pthread_t t;
    pthread_create(&t, NULL, thread_main, NULL);
    pthread_join(t, NULL);
    report("main");                  // after the thread ran

    pid_t p = fork();
    if (p == 0) { counter += 100; report("child"); _exit(0); }
    wait(NULL);
    report("main");                  // after the child ran
    return 0;
}
```

**Prediction 6 (write a table before running):** for the five lines printed (`main`, `thread`, `main`, `child`, `main`), predict whether `pid`, `tid`, `&counter`, `&local` and the `counter` value are the same as the first `main` line or different.

```bash
gcc -O2 -pthread -o shared shared.c
./shared
strace -f -e trace=clone,clone3 ./shared 2>&1 | grep -E 'clone'
```
(To see threads as separate entries in `ps -L`, run the bonus `stacks` program below and inspect it from a second shell.)

<details>
<summary>Answer</summary>

| Line | pid | tid | &counter | &local | counter |
|------|-----|-----|----------|--------|---------|
| main | P | P | X | S1 | 0 |
| thread | **P (same)** | **different** | **X (same)** | **very different** (its own stack) | 100 |
| main (after thread) | P | P | X | S1 | **100**: the main thread sees the thread's write |
| child | **different** | = its own pid | **X (same address!)** | similar to S1 | 200 (starts from the 100 it copied, adds 100) |
| main (after child) | P | P | X | S1 | **100**: the child's write is invisible |

Two lessons: (1) the thread shares the PID and the variable; (2) the child has the **same virtual address** for `counter` but **different memory**: after its write, copy-on-write gave it a private page. Same address in two processes means nothing about physical memory.

In the `strace` output, the thread's `clone3` flags include `CLONE_VM|CLONE_FS|CLONE_FILES|CLONE_SIGHAND|CLONE_THREAD`; the fork's does not include `CLONE_VM`. That *is* the difference between a thread and a process on Linux. (On older glibc you'll see `clone(...)` instead of `clone3`.)

</details>

### Step 4: failure, the race that sharing creates (5 min)

Sharing memory gives you the data race. Processes can't have it on a variable they don't share.

```c
// gcc -O0 -pthread -o race race.c     (-O0: keep the load/add/store visible, no clever optimising)
#include <stdio.h>
#include <pthread.h>

long counter = 0;

static void *work(void *arg) {
    for (int i = 0; i < 1000000; i++)
        counter++;                      // <-- the line to point at
    return arg;
}

int main(void) {
    pthread_t t[4];
    for (int i = 0; i < 4; i++) pthread_create(&t[i], NULL, work, NULL);
    for (int i = 0; i < 4; i++) pthread_join(t[i], NULL);
    printf("expected 4000000, got %ld\n", counter);
    return 0;
}
```

**Prediction 7 (before running):** (a) which line holds the race and why? (b) Will the result be exactly 4,000,000? (c) What if the four workers were `fork`ed processes instead?

```bash
gcc -O0 -pthread -o race race.c && for i in 1 2 3; do ./race; done
```

<details>
<summary>Answer</summary>

(a) `counter++` is not one step: **load** the value, **add 1**, **store** it back. Two threads can both load 17, both add 1, both store 18: one increment lost. (b) Almost certainly less than 4,000,000, and a different number each run. (Needs more than one core to show; if your container has 1 CPU, races are rarer: use `--cpus`/Docker VM settings.) (c) With processes each child increments **its own copy** of `counter` (post-fork private page), so each reaches 1,000,000, none lost, and the parent's value stays 0. No race, but also no shared result: you'd need explicit shared memory or IPC. **Sharing is the power and the bug.** F004 gives you the fix (mutex/atomics).

</details>

### Step 5: Java: platform vs virtual threads (10 min)

`Threads.java`: run 10,000 tasks, each "blocking" for one second (stands in for a DB or HTTP call).

```java
import java.util.concurrent.*;
import java.util.stream.*;

public class Threads {
    public static void main(String[] args) throws Exception {
        int tasks = Integer.parseInt(args[1]);
        long t0 = System.nanoTime();
        try (ExecutorService ex = switch (args[0]) {
            case "platform" -> Executors.newFixedThreadPool(200);          // like Tomcat's default
            case "virtual"  -> Executors.newVirtualThreadPerTaskExecutor();
            default -> throw new IllegalArgumentException(args[0]);
        }) {
            IntStream.range(0, tasks).forEach(i -> ex.submit(() -> {
                Thread.sleep(1000);                                          // blocking I/O stand-in
                return i;
            }));
        }                                                                    // close() waits for all tasks
        System.out.printf("%s: %d tasks in %d ms%n",
                args[0], tasks, (System.nanoTime() - t0) / 1_000_000);
    }
}
```

**Prediction 8 (calculate it):** (a) With a fixed pool of 200 threads, 10,000 tasks of 1 s each: roughly how long? (b) With virtual threads? (c) How many OS threads (`NLWP`) will the JVM have in each case?

```bash
java Threads.java platform 10000 &
sleep 5; ps -o pid,nlwp,cmd -C java       # NLWP = number of OS threads in the process
wait
java Threads.java virtual 10000 &
sleep 0.5; ps -o pid,nlwp,cmd -C java
wait
```
(The platform run takes about a minute. Use `fg`/wait or open a second shell.)

<details>
<summary>Answer</summary>

(a) 10,000 ÷ 200 = 50 rounds × 1 s ≈ **50 seconds** (Little's Law: throughput = concurrency ÷ latency = 200 per second). (b) About **1-2 seconds**: all 10,000 virtual threads block on `sleep` at once, each one unmounted so a handful of carriers suffice. (c) Platform: roughly 200 + the JVM's own housekeeping threads (GC, compiler, etc.). Virtual: roughly the carrier count (~`nproc`) + the JVM's housekeeping threads: *not* 10,000. Those 10,000 "threads" exist only as heap objects, and the OS never sees them.

The lesson: pool size is a **throughput cap** for blocking work. Virtual threads remove that cap for waiting, but not the cap of whatever downstream resource you're waiting on.

</details>

### Bonus (if time): what 1,000 threads cost in memory

```c
// gcc -O2 -pthread -o stacks stacks.c
#include <stdio.h>
#include <string.h>
#include <unistd.h>
#include <pthread.h>

static void *sleeper(void *a) { sleep(30); return a; }

static void show(const char *tag) {
    FILE *f = fopen("/proc/self/status", "r");
    char line[256];
    while (fgets(line, sizeof line, f))
        if (!strncmp(line, "VmSize", 6) || !strncmp(line, "VmRSS", 5) || !strncmp(line, "Threads", 7))
            printf("%s %s", tag, line);
    fclose(f);
}

int main(void) {
    show("before");
    pthread_t t;
    for (int i = 0; i < 1000; i++) pthread_create(&t, NULL, sleeper, NULL);
    sleep(1);
    show("after ");
    return 0;
}
```
Predict: how much do `VmSize` and `VmRSS` grow? (Expect `VmSize` to jump by gigabytes (1,000 × the 8 MB stack reservation) while `VmRSS` grows by only a few MB: the stacks are reserved address space, committed lazily on touch. Same demand-paging idea you'll formalise in F007.) Also try `ulimit -s 1024` before running to shrink the reservation and watch `VmSize` drop.

---

## 6. Production reality

| Concept | How it looks in production |
|---------|----------------------------|
| Seeing threads | `top -H`, `ps -L -p <pid>`, `ls /proc/<pid>/task`. In Java: `jstack <pid>` / `jcmd <pid> Thread.print`. Thread states (RUNNABLE, BLOCKED, WAITING, TIMED_WAITING) map back to the Day 1 Ready/Running/Waiting lifecycle. |
| Pool exhaustion | Latency spikes, **CPU low**, thread dump shows all pool threads blocked on the same thing (DB call, lock, downstream HTTP). Requests queue behind a full pool. Not a CPU problem. |
| Unbounded pools / queues | `newCachedThreadPool` + slow downstream = thread explosion. `newFixedThreadPool` + unbounded queue = heap growing until OOM. Use bounded queue + rejection policy. |
| `unable to create native thread` | Not a heap problem. Check `ulimit -u`, `/proc/sys/kernel/threads-max`, `pid_max`, container `pids.max`, and **native memory** (stacks + kernel structures live outside `-Xmx`). |
| Memory accounting | 1,000 platform threads × 1 MB `-Xss` reserved ≈ 1 GB of address space. Mostly uncommitted, but shows up in container memory pressure when stacks are deep. |
| Context-switch overload | `vmstat 1` high `cs`, run queue much bigger than cores → too many runnable threads for CPU-bound work. |
| Virtual thread adoption | Remove the pool; add a `Semaphore` for the scarce downstream; watch for pinning (`jdk.VirtualThreadPinned` JFR event) and for DB connection pool becoming the limit. |
| Process vs thread choice | PgBouncer in front of Postgres because each connection is a process; nginx worker processes; a crashing Chrome tab doesn't kill the browser. Isolation is why. |
| Preview of later days | Scheduling (F003) decides which ready thread runs; synchronisation (F004) fixes the race you just made; `epoll`/event loops (F009) are how virtual threads and Netty avoid a thread per connection; container CPU limits (F010) change how many cores your pools *think* they have. |

---

## 7. Interview drill (Chunk 5, part 1)

Answer each out loud in 60-90 seconds as **claim → mechanism → example → trade-off**. Then open the hints.

1. **Definition:** What is the difference between a process and a thread? What is shared and what is private?
2. **Why:** Why is creating a thread cheaper than creating a process? Why is a thread context switch cheaper?
3. **Kernel vs user:** What's the difference between kernel-level and user-level threads? Which model do Linux and Java platform threads use?
4. **Failure:** Why does a bug in one thread crash the whole application but a bug in one process usually doesn't?
5. **Pools:** Why use a thread pool? How would you size one for CPU-bound vs I/O-bound work?
6. **Virtual threads:** What problem do they solve and how? When would you *not* use them?
7. **Scenario:** A Spring Boot service's p99 latency tripled but CPU is at 10%. Thread dump shows all 200 Tomcat threads waiting. What's happening, and what are your options?
8. **Design:** Why does PostgreSQL use a process per connection while Tomcat uses threads? What does each choice cost?

<details>
<summary>Hints (open only after attempting)</summary>

1. Process = address space + resources; thread = execution context inside it. Shared: code, globals, heap, fds, signal handlers, PID. Private: stack, registers/PC, TID, TLS/errno. Mention Linux `clone` flags.
2. No new address space, no page-table/fd-table copy → cheaper creation. Switch skips the address-space switch → TLB and caches remain useful. Still a kernel entry.
3. Who schedules: kernel vs library. Block behaviour and multicore difference. Linux/Java platform: one-to-one. Go/virtual threads: many-to-many.
4. Threads share one address space and one process: `SIGSEGV`, OOM-kill, or memory corruption affects all. Processes are isolated by separate address spaces. Cost of isolation: IPC and memory.
5. Bounded concurrency + reuse + back-pressure. CPU-bound ≈ cores. I/O-bound ≈ N×(1+W/C), or Little's Law. Bounded queue + rejection policy.
6. Blocking-style code with massive concurrency; unmount on blocking, carriers shared. Not for CPU-bound, don't pool them, pinning, downstream limits.
7. Pool exhaustion from a slow dependency (Little's Law: latency up → concurrency demand up → hits 200). Not CPU-bound. Find what the threads wait on (`jstack`), add timeouts, bulkheads, bounded queues; virtual threads help the thread cap but not the slow dependency.
8. Isolation vs memory and switch overhead; shared memory convenience vs risk. Postgres → pooler. Tomcat → careful with shared state.

</details>

---

## 8. Checkpoint (Chunk 5, part 2)

Do these without scrolling up.

**Explain:** On a whiteboard, draw (a) one process with three threads and (b) two processes after `fork`. Mark exactly what is shared, what is private, and what is copied. Add the `clone` flag that makes the difference.

**Apply:** A service handles 150 requests/s. Average latency is 400 ms, almost all waiting on a database. (1) How many requests are in flight on average? (2) Is a pool of 200 threads enough? (3) A downstream slowdown pushes latency to 2 s. What now? Show the arithmetic.

**Diagnose:** A Java service starts throwing `OutOfMemoryError: unable to create native thread` while heap usage shows only 40%. Thread count in `ps -o nlwp` keeps climbing. Trace to the root cause from first principles. Which pool configuration would cause this, and what are two fixes?

**Interview (timed, 2 min):** "When would you choose multiple processes over multiple threads, and vice versa?"

### Done-when check (the curriculum's own words)
> *You can say exactly what is shared and what is copied between the two.*

Rate yourself honestly:

- [ ] **Green**: I can explain, apply, and defend it without notes. Done-when is met.
- [ ] **Yellow**: I get the idea but need more hands-on practice (re-run Steps 3 and 4).
- [ ] **Red**: The mental model is still unclear. Revisit Chunk 1 and Day 1's fork section, add it to the gap list.

**Gap list (carry forward; include any Day 1 warm-up misses):**
-
-

---

## 9. Your note (required: Phase 1 is only done when it exists)

Write this **in your own words, from memory**, in 5-10 lines. Don't copy from this file. A good note is short, mechanism-first, and includes **one thing you got wrong or found surprising** (e.g. the child printing the same `&counter` address, how few OS threads virtual threads used, an unexpected benchmark result).

```text
F002 note:

Shared vs private in a thread (and what a process copies instead):


Why threads are cheaper than processes (creation and switching):


Platform vs virtual threads in one line:


Where the race came from:


What surprised me / what I got wrong:


```

When you've written a draft, bring it back and ask for a tightening pass.

---

## 10. Quick reference (for revision, not for learning)

| Term | One line |
|------|----------|
| Process | Address space + resources + threads, tracked by a PCB |
| Thread | Execution context (stack, registers, PC, TID) inside a process; shares memory, fds, handlers |
| Shared by threads | Code, globals, heap, open fds, signal handlers, PID |
| Private to a thread | Stack, registers/PC, TID, TLS, `errno`, signal mask |
| `clone` flags | `CLONE_VM\|FILES\|SIGHAND\|THREAD…` = thread; without `CLONE_VM` = process (fork) |
| Kernel thread (1:1) | Kernel schedules it; blocking blocks only it; runs in parallel on cores |
| User-level thread | Runtime schedules it; cheap switch; blocking can stall all (M:1) |
| M:N | Many user-level tasks on few kernel threads (Go, Java virtual threads) |
| Cost of a thread | Reserved stack, kernel stack + `task_struct`, creation syscall, scheduling overhead, system limits |
| Thread pool | Bounded reusable workers + queue = bounded concurrency and back-pressure |
| Pool order | Core threads → queue → extra threads up to max (only when queue is full) |
| Sizing | CPU-bound ≈ cores; I/O-bound ≈ N×(1+W/C); check with Little's Law |
| Virtual thread | Cheap heap-allocated thread; mounts on a carrier, unmounts when it blocks |
| Pinning | Virtual thread stuck to its carrier while blocked (e.g. in `synchronized` pre-JDK 24, native frames) |
| Little's Law | In-flight requests = arrival rate × average latency |

### Commands used today
`nproc` · `time ./prog` · `ps -L -o pid,lwp,nlwp,cmd` · `top -H` · `ls /proc/<pid>/task` · `strace -f -e trace=clone,clone3` · `ulimit -s` / `ulimit -u` · `jstack` / `jcmd <pid> Thread.print` · `java File.java`

### What's next
**F003: CPU scheduling.** You now know that the kernel schedules *threads*. Tomorrow's question is how it picks which ready thread gets the CPU and for how long: FCFS, SJF, SRTF, Round Robin, priority and MLFQ. It's a computation-heavy day, so have pen and paper ready for Gantt charts and turnaround/waiting/response times.
