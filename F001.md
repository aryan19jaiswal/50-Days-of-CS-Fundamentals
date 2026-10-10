# F001: What an OS actually does

**Weekend / Slot:** W01 · Saturday (2026-10-10)
**Phase / Block:** 1 Learning · A Operating Systems
**Time budget:** 2 hours

**Topics (curriculum):** Kernel vs user mode, system calls, process lifecycle, PCB, context switch cost, `fork`/`exec`/`wait`, zombie & orphan processes

**Drill:** Trace the syscalls of a tiny program (`strace` in a Linux container, or `dtruss` on macOS) and annotate every line.

**Done when:** You can explain, memory-wise, what happens on `fork()` and why a zombie exists.

> Goal of the day: stop reciting "the OS manages resources." Be able to draw what the CPU and memory look like at the moment of `fork()`, and say why a dead process can still show up in `ps`.

---

## 0. How to use this file

- Predictions have their answers hidden in `<details>` blocks. **Write your own answer first**, then open the block. Guessing wrong is the point; note what you got wrong, because the note you write at the end needs one.
- Supporting material (not in the curriculum topic list) is marked **[Supporting]**.
- Answer interview questions out loud, claim → mechanism → example → trade-off.

### The 2 hours

| Chunk | Min | What | Level |
|-------|-----|------|-------|
| 1 | 20 | Kernel vs user mode, system calls | Understand |
| 2 | 25 | Process, lifecycle, PCB, context switch | Understand |
| 3 | 30 | `fork` / `exec` / `wait`, zombie & orphan | Understand → Reproduce |
| 4 | 30 | **The Drill**: strace + zombie + orphan | Reproduce → Diagnose |
| 5 | 15 | Interview drill, checkpoint, written note | Defend |

Chunks 1-3 are deliberately short. The Drill is the centre of the day; it should make the theory necessary, not decorate it.

---

## 1. Setup (do this first, 5 min, outside the clock if needed)

You are on macOS. `strace` is Linux-only; `dtruss` on macOS needs `sudo` and runs into System Integrity Protection. Use a Linux container so everything below behaves as written.

```bash
docker run --rm -it --cap-add=SYS_PTRACE ubuntu:24.04 bash
# inside the container:
apt-get update && apt-get install -y gcc strace procps
```

- `--cap-add=SYS_PTRACE` is required, otherwise `strace` fails with `ptrace: Operation not permitted`.
- `procps` gives you `ps`, `top`.
- macOS equivalent if you insist: `sudo dtruss -f ./a.out` (expect SIP friction; the container is the reliable path).

---

## 2. Chunk 1: Kernel vs user mode, system calls (20 min)

### The problem
Many programs share one machine. If any program could touch the disk controller, overwrite another program's memory, or halt the CPU, one bug (or one malicious program) takes everything down. Something has to be the **only** code allowed to do dangerous things, and everyone else has to ask it politely.

### Mental model

```text
 ┌─────────────────────────── User mode (ring 3) ───────────────────────────┐
 │  Your JVM / Spring Boot app / a.out                                      │
 │  Can: compute, read/write its OWN memory                                 │
 │  Cannot: touch devices, other processes' memory, page tables, halt CPU   │
 └──────────────────────────────┬───────────────────────────────────────────┘
            system call  (special instruction: "syscall" / "svc")
                                │   CPU flips to privileged mode,
                                │   jumps to a fixed kernel entry point
 ┌──────────────────────────────▼───────────────────────────────────────────┐
 │  Kernel mode (ring 0)                                                    │
 │  Scheduler · memory manager · file systems · network stack · drivers     │
 │  Full access to hardware and all memory                                  │
 └──────────────────────────────────────────────────────────────────────────┘
```

The mode is a **bit in the CPU**, not a separate machine. The hardware refuses privileged instructions when the bit says "user." That is what makes the boundary real and not a convention.

### How a system call actually flows

1. Your code calls a libc wrapper, e.g. `write(1, "hi\n", 3)`.
2. The wrapper puts the **syscall number** (write = 1 on x86-64 Linux) and arguments in specific registers.
3. It executes the `syscall` instruction. The CPU switches to kernel mode and jumps to the kernel's entry point; the user can't choose *where*, only *which number*.
4. The kernel looks up the number in the **syscall table**, validates the arguments (is that pointer actually yours?), does the work (maybe blocking you), and puts the result in a register.
5. It returns to user mode (`sysret`); the wrapper turns a negative result into `-1` + `errno`.

Key facts:
- A syscall is **cheap compared to a disk read, expensive compared to a function call** (mode switch, argument validation, cache/TLB effects). That is why buffering exists: `printf` collects bytes in user space and calls `write` once. Don't quote a nanosecond figure; measure it if asked.
- **[Supporting]** Some "syscalls" avoid the kernel entirely: `gettimeofday`/`clock_gettime` are served by the **vDSO**, a small kernel-provided page mapped into your process. You will not see them in `strace`.
- **Syscall vs interrupt vs exception** (people mix these up):
  - *Syscall / trap*: the program asked, deliberately, synchronously.
  - *Exception / fault*: the program did something the CPU can't complete (divide by zero, page fault), synchronous.
  - *Interrupt*: hardware says "attention" (timer tick, disk done, packet arrived), asynchronous, unrelated to the current instruction.
  - The **timer interrupt** is how the kernel gets the CPU back from a program that never gives it up. Remember this for scheduling (F003).

### Java connection
Your Spring Boot app never touches the network card. `socket.read()` in Java → JVM native code → libc `read`/`recv` → kernel. Every blocking call you write is a syscall that may put your thread to sleep.

### Prediction 1 (write before opening)
A C program calls `printf("hello\n")` ten times in a loop, with stdout redirected to a file. Roughly how many `write` syscalls will `strace` show for the output, and why?

<details>
<summary>Answer</summary>

Usually **one** (or very few). When stdout is a file or pipe, libc uses full buffering (block buffered, ~4 KB), so all ten lines sit in a user-space buffer and are flushed with a single `write` at exit. When stdout is a terminal it is line buffered, so you'd see ten. This is also the root of the duplicate-output trap in Section 5.

</details>

---

## 3. Chunk 2: Process, lifecycle, PCB, context switch (25 min)

### The problem
A program is a file on disk. Running it needs state: where it's up to (registers, program counter), its memory, its open files. And one CPU has to look like many. The OS needs a *thing* it can start, pause, resume and kill. That thing is a **process**.

### Mental model
A process = **an address space + one or more threads of execution + kernel-held resources (open files, signals, credentials)**, tracked by the kernel in a **PCB**.

### The PCB (Process Control Block)
The kernel's per-process record (in Linux, `task_struct`). Think of it as everything the OS must save to freeze you and restore you later:

| Group | Contents |
|-------|----------|
| Identity | PID, PPID (parent), UID/GID |
| State | new / ready / running / waiting / terminated |
| CPU context | Program counter, stack pointer, general registers (saved when you're not running) |
| Memory | Pointer to page tables / memory map (address space) |
| Files | Open file descriptor table |
| Scheduling | Priority, accounting, which queue it's on |
| Other | Signal handlers, exit status, pointers to children |

### Lifecycle

```text
          admit               scheduler dispatch
  New ───────────► Ready ◄───────────────────────► Running
                     ▲        timer interrupt /        │   │
                     │        preempted                │   │ exit()
          I/O done / │                                 │   ▼
          event      │        I/O request /            │  Terminated
                     └──── Waiting (blocked) ◄─────────┘  (→ Zombie until
                              wait on lock, disk, net        parent wait()s)
```

- **Ready**: could run, waiting for a CPU. **Waiting/blocked**: cannot run until something happens (disk read finishes, packet arrives).
- A blocked process costs no CPU. This is why a server with 10,000 idle connections isn't burning 10,000 cores.
- **[Supporting]** What `ps`/`top` show on Linux in the `STAT` column: `R` running/runnable, `S` interruptible sleep, `D` uninterruptible sleep (usually waiting on disk; can't be killed), `T` stopped, `Z` zombie.

### Context switch: what it is and what it costs
Switching from process A to B on one CPU:

1. A is interrupted (timer interrupt) or blocks (syscall that waits).
2. Kernel **saves A's CPU context** into A's PCB.
3. Scheduler picks B.
4. Kernel **loads B's context** from B's PCB and **switches the address space** (new page-table base register).
5. Return to user mode; B continues as if nothing happened.

Cost has two parts:
- **Direct**: save/restore registers, scheduler decision, kernel entry/exit. Small, fixed.
- **Indirect (usually bigger)**: B's data isn't in the CPU caches; the **TLB** held A's translations (flushed or partly invalid after an address-space switch), so B starts cold and takes cache misses and TLB misses.
- A **thread** switch within the same process skips the address-space switch, so it's cheaper (preview of F002).
- It's pure overhead: no useful work happens during the switch. Too many runnable threads → the machine spends its time switching.

### Prediction 2
A process is in the Running state and calls `read()` on a socket with no data yet. Which state does it go to, which state does it **not** go to, and who moves it back?

<details>
<summary>Answer</summary>

Running → **Waiting (blocked)**, not Ready: it can't make progress even if given the CPU. It leaves the CPU, and the scheduler runs something else. When the network card raises an **interrupt** and the kernel's network stack delivers data to that socket, the kernel moves the process Waiting → **Ready**. It then runs only when the scheduler dispatches it (Ready → Running). Note it goes to Ready, not straight to Running.

</details>

---

## 4. Chunk 3: `fork` / `exec` / `wait`, zombie & orphan (30 min)

This is the centre of the "Done when." Slow down here.

### The problem
How does a new process come to exist? Unix's answer is two small, separate operations instead of one big "spawn":

- `fork()` : **clone the calling process**.
- `exec*()` : **replace the current program** with a new one.

Splitting them lets the child adjust things (redirect stdout, close fds, drop privileges) *between* the two, which is exactly how a shell implements `ls > out.txt`.

### `fork()`: what happens, memory-wise

```text
BEFORE fork()                     AFTER fork() (before anyone writes)

Parent                            Parent                 Child
 page table ──► [frame A]         page table ─┐          page table ─┐
                [frame B]                      ├─► [frame A]  ◄──────┤   (both read-only,
                [frame C]                      ├─► [frame B]  ◄──────┤    marked COW)
                                               └─► [frame C]  ◄──────┘

AFTER child writes to B:
 Parent ──► [frame B]          Child ──► [frame B']   ← kernel copies just that page
```

Step by step:
1. Kernel allocates a new PCB with a **new PID**; PPID = caller's PID.
2. It **copies the parent's page tables** (not the data pages) into the child. Both processes now map the **same physical frames**.
3. All writable pages are marked **read-only + copy-on-write (COW)** in both.
4. The **file descriptor table is copied**, but the entries point to the **same open file descriptions**. So parent and child **share the file offset** for an open file. (This is why two processes writing to the same inherited stdout interleave rather than overwrite.)
5. The CPU registers are copied. Both resume right after `fork()`.
6. The return value differs: **parent gets the child's PID, child gets 0** (and `-1` on failure). That's the only way the code knows who it is.
7. On the first write by either side to a shared page → **page fault** → kernel copies *that one page*, gives the writer a private copy, and lets the write proceed.

Why COW matters: fork is fast even for a 4 GB process because no data is copied up front. And fork+exec (the dominant pattern) would waste work if it copied everything only to throw it away at exec.

Things that are **not** shared after fork: PID, memory contents after the first write (private from then on), pending signals, locks held by other threads (only the calling thread exists in the child, a known multithreading hazard).

### `exec*()`: what happens
- Same process (same PID, same PCB, same open fds unless marked close-on-exec), but the **address space is thrown away** and rebuilt from the new executable: new code, fresh stack and heap.
- It does **not** create a process. `exec` does not return on success; the old program is gone.
- That's why `execve` appears in the very first line of every `strace`.

### `wait()` / `waitpid()`
- Parent blocks until a child **changes state** (exits), then receives the child's **exit status**.
- Kernel also sends the parent `SIGCHLD` when a child dies.
- **Crucially, `wait()` is what lets the kernel finally delete the child's leftover record.** That sets up the zombie.

### Zombie process: why it exists
When a child calls `exit()`:
- The kernel frees its **memory, open files, and most resources immediately**.
- But it must keep a **minimal PCB stub**: PID, exit status, resource-usage stats. *Reason*: the parent may call `wait()` later and has a right to learn how the child ended. The kernel can't throw that information away just because the parent was busy.
- Until the parent calls `wait()`, the child is a **zombie**: dead, holding no memory, but still occupying a **process-table slot and a PID**.

Consequences:
- `ps` shows state `Z` / `<defunct>`.
- **You can't `kill` a zombie**: it's already dead. There is nothing to signal.
- The fix is on the **parent**: it must call `wait()`, or the parent must die (then the zombie is reparented and reaped).
- A handful of zombies are harmless; a parent that spawns children in a loop and never waits will eventually **exhaust PIDs / the process table** (`fork: Resource temporarily unavailable`).

### Orphan process
Parent dies **before** the child. The child keeps running; the kernel **reparents** it to `init` (PID 1) (or a designated "subreaper"), which calls `wait()` on adopted children as they exit. So orphans are fine; they get cleaned up.

| | Zombie | Orphan |
|---|--------|--------|
| Who's dead? | The **child** (finished) | The **parent** (finished) |
| Who's alive? | Parent, not calling `wait()` | The child, still running |
| Memory held? | None (only a table entry) | Yes, it's running normally |
| Cleanup | Parent `wait()`s, or parent exits | `init` adopts, reaps on exit |

### Prediction 3 (the Done-when question; answer in writing)
Process P (using 2 GB of RAM) calls `fork()`. Immediately afterwards:
(a) roughly how much additional physical memory is used?
(b) the child then writes to one 4 KB page. What happens, step by step?
(c) the child then immediately calls `execve("/bin/ls")`. What happens to the pages it had been sharing?

<details>
<summary>Answer</summary>

(a) Almost none: a new PCB and a copy of the **page tables** (a small fraction of the data size), because the data frames are shared and marked COW.

(b) The write hits a read-only COW page → hardware raises a **page fault** → kernel sees "this is a COW page" → allocates a new 4 KB frame, copies the contents, maps it writable into the child's page table, drops the shared mapping for that page (if the parent is now the sole user, it can write freely too) → instruction is restarted and succeeds. Only that one page was copied.

(c) `execve` discards the child's whole address space. The child's mappings to the shared frames are removed (reference counts drop; frames remain in use by the parent), and the child gets a fresh address space for `/bin/ls`. The 2 GB was never copied, so the fork cost nothing meaningful. That's the payoff of COW.

</details>

---

## 5. Chunk 4: THE DRILL (30 min)

**Drill:** Trace the syscalls of a tiny program and annotate every line.

You'll use one program that exercises everything: it forks, the child execs `echo`, the parent waits.

### Step 1: write the program (`/tmp/t.c`, inside the container)

```c
#include <stdio.h>
#include <unistd.h>
#include <sys/wait.h>

int main(void) {
    printf("parent pid=%d\n", getpid());
    pid_t pid = fork();
    if (pid == 0) {                       // child
        execlp("echo", "echo", "hello from child", (char *)NULL);
        perror("exec failed");            // only reached if exec fails
        _exit(127);
    }
    int status;
    waitpid(pid, &status, 0);             // parent reaps child
    printf("child %d exited with %d\n", pid, WEXITSTATUS(status));
    return 0;
}
```

```bash
gcc -o t /tmp/t.c
./t                                  # sanity check
strace -f -o /tmp/trace.txt ./t      # -f = follow children, -o = write to file
cat /tmp/trace.txt
```

`-f` matters: without it you'd only see the parent and miss the child's `execve`.

### Step 2: annotate every line

Copy the trace into a table like this and fill the right-hand column **in your own words**. Do not skip lines you don't recognise: look them up with `man 2 <name>`. This table *is* the drill.

| Line (syscall) | What it asks the kernel to do | Phase of the program |
|----------------|-------------------------------|----------------------|
| `execve("./t", ...)` | | |
| `brk(NULL)` | | |
| `mmap(...)` | | |
| `openat(... "libc.so.6" ...)` | | |
| `read` / `fstat` / `close` | | |
| `write(1, "parent pid=...")` | | |
| `clone(...)` / `clone3(...)` | | |
| `[pid N] execve("/usr/bin/echo", ...)` | | |
| `[pid N] write(1, "hello from child\n")` | | |
| `[pid N] exit_group(0)` | | |
| `wait4(N, ...)` | | |
| `--- SIGCHLD ---` | | |
| `write(1, "child N exited ...")` | | |
| `exit_group(0)` | | |

Typical shape of what you'll see (details vary by distro/glibc):
- **Process start-up noise**: `execve`, then `brk`/`mmap`/`openat`/`read`/`fstat`/`close` calls that load the dynamic linker and libc, and set up the heap. All of this happens *before* your `main()` runs.
- `fork()` is implemented as **`clone` or `clone3`** on modern Linux. `fork` is a wrapper. Find the return value: in the parent it's the child's PID.
- The child then does its **own** loader noise again after its `execve` (it's a new program: `echo`).
- The parent sits in `wait4` while the child runs, and `SIGCHLD` arrives when the child exits.

Questions to answer from **your own trace**:
1. Which line is the `fork`? What does it return in the parent, and what PID appears on the child's lines?
2. Where does the parent block, and for how long relative to the child's activity?
3. The child's first syscall after `clone` returns: is it `execve` immediately? Why does that make COW worthwhile?
4. Why is there a loader sequence *again* after the child's `execve`?
5. How many `write` calls did the parent's two `printf`s produce? Why? (Prediction 1.)

### Step 3: failure #1, create a zombie on purpose

Predict first: if the child exits while the parent sleeps without ever calling `wait`, what does `ps` show for the child, and does it use memory?

```c
// /tmp/zombie.c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int main(void) {
    pid_t pid = fork();
    if (pid == 0) { exit(0); }                       // child dies immediately
    printf("parent=%d child=%d (no wait)\n", getpid(), pid);
    sleep(60);                                       // parent never reaps
    return 0;
}
```

```bash
gcc -o zombie /tmp/zombie.c && ./zombie &
ps -o pid,ppid,stat,cmd --ppid $!       # look for STAT = Z and "<defunct>"
kill -9 <zombie-pid>                    # try it. Does anything change?
```

Observe and write down:
- STAT column shows `Z`.
- `kill -9` on the zombie does nothing. Why?
- Now kill the **parent** (`kill $!`) and re-run `ps`. The zombie disappears. *Who* reaped it? (Hint: Section 4, orphan/reparenting.)
- Fix the program by adding `wait(NULL)` in the parent. Confirm no zombie appears.

### Step 4: failure #2, create an orphan on purpose

```c
// /tmp/orphan.c
#include <stdio.h>
#include <unistd.h>

int main(void) {
    pid_t pid = fork();
    if (pid == 0) {
        printf("child: ppid before = %d\n", getppid());
        sleep(5);                                    // parent will be gone by now
        printf("child: ppid after  = %d\n", getppid());
        return 0;
    }
    sleep(1);
    printf("parent exiting\n");
    return 0;                                        // parent dies first
}
```

Predict `ppid after` before you run it. Then run it and compare. In this container, PID 1 is your `bash`, not a real init. **Question:** what happens if PID 1 inside a container never calls `wait()` on adopted children? (This is a real production issue: see Section 6.)

### Step 5: failure #3, the duplicated-output trap (bonus, 5 min)

```c
// /tmp/dup.c
#include <stdio.h>
#include <unistd.h>
int main(void) {
    printf("before fork\n");      // newline flushes only if stdout is a terminal
    fork();
    printf("after\n");
    return 0;
}
```

Run `./dup` (terminal) and then `./dup | cat` (pipe). Predict the output of each, then explain from what fork copies. Answer in the details below *after* you've tried.

<details>
<summary>Answer</summary>

On a terminal, stdout is **line buffered**: the `\n` flushes `before fork` before the fork, so you see `before fork` once, then `after` twice.

Through a pipe, stdout is **block buffered**: `before fork\n` is still sitting in the user-space buffer when `fork()` runs. fork **copies the whole address space, including that buffer**, so parent and child each flush their own copy: `before fork` appears **twice**, then `after` twice.

Lesson: fork duplicates user-space state, including unflushed stdio buffers. Fix: `fflush(stdout)` before `fork()`, or use `_exit` in the child.

</details>

---

## 6. Production reality

| Concept | How it looks in production |
|---------|----------------------------|
| Syscall cost | A service doing many tiny unbuffered `write`s is slow. Batching/buffering reduces syscalls. `strace -c -p <pid>` gives a syscall count/time summary (don't use it on a hot prod process: it slows the target a lot). |
| Context switches | `vmstat 1` (`cs` column), `pidstat -w`. Thousands of runnable threads per core → high `cs`, low useful throughput. |
| Zombies | `ps aux \| awk '$8=="Z"'` or `top` summary "N zombie". Usually a buggy parent (a supervisor or app that spawns subprocesses without `wait`/`SIGCHLD` handling). Fix the parent, not the zombie. |
| PID exhaustion | `fork: retry: Resource temporarily unavailable`, `java.lang.OutOfMemoryError: unable to create native thread` can be a pid/thread limit, not heap. Check `ulimit -u`, `/proc/sys/kernel/pid_max`. |
| Containers | The first process in a container is PID 1 and becomes the reaper for orphans. If it's your Java app (or a shell script) that doesn't reap, zombies accumulate. Use `docker run --init` (or `tini`) so a tiny proper init reaps. |
| `fork` of a big JVM | The JVM spawning subprocesses (`ProcessBuilder`) historically used fork-ish approaches; with COW it's cheap, but memory overcommit settings still matter. Modern JDKs use `posix_spawn`/`vfork` to avoid the cost. **[Supporting]** |
| Preview of later days | Threads (F002) change what is shared; scheduling (F003) decides who gets the CPU after the timer interrupt; synchronisation (F004) is needed because threads share memory. |

---

## 7. Interview drill (Chunk 5, part 1)

Answer each out loud in 60-90 seconds as **claim → mechanism → example → trade-off**. Then open the hints.

1. **Definition:** What is a system call, and why can't user code just do what it wants?
2. **Why:** Why does Unix separate `fork` and `exec` instead of having one `spawn`?
3. **How:** Walk through what `fork()` does to memory. What is copy-on-write, and when does the actual copy happen?
4. **Failure:** What is a zombie process? Why does it exist? Why can't you `kill -9` it, and how do you get rid of it?
5. **Scenario:** A production host shows 20,000 `<defunct>` processes and new forks start failing. What do you check and what is the root cause?
6. **Trade-off:** What does a context switch cost, and why is the indirect cost usually bigger than the direct one?
7. **Container twist:** Why might a Java app running as PID 1 in a Docker container leak zombies?

<details>
<summary>Hints (open only after attempting)</summary>

1. Privileged instructions only work in kernel mode (a CPU bit). The syscall instruction is the controlled gate: fixed entry point, number → table → validated args. Protection and isolation are the reason.
2. Between fork and exec the child can rearrange fds/env/privileges (shell redirection, pipes). Cost of fork is small due to COW. Trade-off: fork of a multithreaded process only copies the calling thread, which is a hazard.
3. Page tables copied, frames shared, pages marked read-only COW, copy on first write via page fault. Mention return values (child PID / 0).
4. Terminated child, parent hasn't called `wait()`. Kernel keeps PID + exit status. Nothing to kill. Fix the parent or kill the parent so init reaps.
5. `ps -eo pid,ppid,stat,cmd \| grep ' Z'`: group by PPID; one parent is not reaping. Fix its `wait`/`SIGCHLD` handling, or restart it. Root cause: PID table exhaustion from unreaped children.
6. Saved context + scheduler (direct); cold caches and TLB after address-space switch (indirect). Thread switches within a process skip the address-space part.
7. PID 1 in a container is the orphan reaper; a JVM isn't written to reap arbitrary adopted children. Use `--init`/`tini`.

</details>

---

## 8. Checkpoint (Chunk 5, part 2)

Do these without scrolling up.

**Explain:** On a whiteboard, draw parent and child page tables before and after `fork()`, then after a child write, then after the child's `exec`. Explain each arrow.

**Apply:** `ps` shows PID 4100 with state `Z` and PPID 4000. Name the exact command(s) you'd use to find out what process 4000 is, and the two ways the zombie can disappear.

**Diagnose:** A Java service that shells out to `convert` (an image tool) hundreds of times per minute slowly leaks defunct processes until fork fails. Trace to the root cause from first principles and say what the fix is.

**Interview (timed, 2 min):** "What happens when you run `ls` in a shell?" Cover: shell forks, child execs `ls`, parent waits, `ls` makes syscalls (`openat`/`getdents`/`write`), exits, parent reaps and prints the prompt.

### Done-when check (the curriculum's own words)
> *You can explain, memory-wise, what happens on `fork()` and why a zombie exists.*

Rate yourself honestly:

- [ ] **Green**: I can explain, apply, and defend it without notes. Done-when is met.
- [ ] **Yellow**: I get the idea but need more hands-on practice (re-run the Drill).
- [ ] **Red**: The mental model is still unclear. Revisit Chunk 3, add it to the gap list.

**Gap list (carry forward):**
-
-

---

## 9. Your note (required: Phase 1 is only done when it exists)

Write this **in your own words, from memory**, in 5-10 lines. Don't copy from this file. A good note is short, mechanism-first, and includes **one thing you got wrong or found surprising** (e.g. a prediction you missed, a line in `strace` you didn't expect).

```text
F001 note:

Fork, in memory terms:


Why a zombie exists:


Zombie vs orphan:


What surprised me / what I got wrong:


```

When you've written a draft, bring it back and ask for a tightening pass: it'll get checked for vagueness and errors while keeping your voice.

---

## 10. Quick reference (for revision, not for learning)

| Term | One line |
|------|----------|
| Kernel mode | CPU privilege level where all instructions and memory are allowed |
| Syscall | Controlled entry from user into kernel: number + args → table → work → return |
| PCB | Kernel's per-process record: identity, state, saved registers, memory map, fd table |
| Context switch | Save A's context, load B's, switch address space; cost = direct + cold caches/TLB |
| `fork` | Clone process: new PID, copied page tables, shared COW frames, shared open file descriptions |
| COW | Pages shared read-only; first write faults and copies just that page |
| `exec` | Replace current process image; same PID; doesn't return on success |
| `wait` | Block until child exits; collect status; lets kernel free the child's table entry |
| Zombie | Exited child whose parent hasn't `wait()`ed: no memory, still a PID + status |
| Orphan | Running child whose parent exited; adopted by PID 1 (or subreaper) |

### Commands used today
`strace -f -o file ./prog` · `strace -c` · `ps -o pid,ppid,stat,cmd` · `top` · `kill` · `man 2 fork|execve|wait4|clone` · `ulimit -u` · `docker run --init`

### What's next
**F002: Processes vs threads.** Today you saw what fork copies; tomorrow you find out what a thread deliberately *doesn't* copy, and why that makes it cheaper and more dangerous.
