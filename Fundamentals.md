# Fundamentals - 180 Days of Batman

> Source of truth for the **CS Fundamentals tab**.
> Everything below is meant to be parsed/rendered by the website: phases, weekend blocks, day cards, and progress rules.
> Referenced by `Plan.md`. Sibling weekend tracks: `SystemDesign.md`, `Projects.md`. Weekday track: `Development.md`.

---

## 1. Tab Meta (for the website)

```yaml
tab: fundamentals
heading: "Fundamentals"
subheading: "OS · DBMS · Networks · OOPs"
blurb: >
  This is the weekend theory track. Saturdays and Sundays I go after the CS core
  that interviewers still actually ask about - how an OS really runs my code, how
  a database really stores and locks it, how a packet really reaches it, and how
  I model all of it as objects. 36 days to learn it properly, then 14 days of
  timed mock rounds using only questions real product companies have asked.
  50 weekend days. Every day ends with something written in my own words.
theme:
  primary: "#F5C518"    # Bat-signal yellow
  accent:  "#8A94A6"    # steel
  surface: "#0B0C10"    # Gotham black
schedule:
  days_of_week: [Sat, Sun]
  start_date: "2026-10-10"   # first Saturday after the 2026-10-05 kickoff
  end_date:   "2027-03-28"   # Sunday, last weekend inside the 180 days
  total_days: 50             # 25 weekends x 2 days
  daily_commitment: "2 hours"
  timezone: "Asia/Kolkata"   # IST
  pairing: "Shares the weekend with SystemDesign.md and Projects.md - Fundamentals first (theory while fresh), then System Design, then Projects."
counters:
  - { label: "Weekend Days", value: 50 }
  - { label: "Weekends", value: 25 }
  - { label: "Phases", value: 2 }
  - { label: "Subjects", value: 4 }
```

---

## 2. Phase Map

| # | Phase | Days | Weekends | Plan.md mapping | Badge colour |
|---|-------|------|----------|-----------------|--------------|
| 1 | Learning Mode - build the core | F001-F036 | W01-W18 | "Important concepts from different CS core subjects" (OS, DBMS, CN, OOPs) | `#F5C518` Bat-signal yellow |
| 2 | Interview Mode - company-tagged mocks | F037-F050 | W19-W25 | Same subjects, revised from an interview angle (added, see §6) | `#4EA8DE` signal blue |

### Subject blocks inside Phase 1

| Block | Subject | Days | Weekends | Badge colour |
|-------|---------|------|----------|--------------|
| A | Operating Systems | F001-F010 | W01-W05 | `#F5C518` Bat-signal yellow |
| B | Databases (DBMS) | F011-F020 | W06-W10 | `#4EA8DE` signal blue |
| C | Computer Networks | F021-F028 | W11-W14 | `#7BD389` utility green |
| D | OOPs & Object Design | F029-F036 | W15-W18 | `#C0C4CC` titanium |

Progress bar on the tab = `completedDays / 50`. Each phase renders its own sub-bar, and each Phase 1 subject block renders a mini-bar in the legend.

---

## 3. Data Contract

Each day card is one object. The website should hydrate from this shape:

```ts
type FundDay = {
  id: string;          // "F023"
  day: number;         // 23
  weekend: number;     // 12  - the Nth weekend, not the Nth week of the year
  slot: "Sat" | "Sun";
  phase: 1 | 2;
  subject: "OS" | "DBMS" | "CN" | "OOPS" | "MIXED";
  title: string;       // short label on the card
  topics: string[];    // bullet list inside the card
  drill: string;       // the hands-on task for the day
  doneWhen: string;    // the completion check - this is what unlocks the tick
  companies?: string[];// Phase 2 only - renders as tag chips on the card
  status: "locked" | "today" | "done" | "missed";
  notes?: string;      // user-written, persisted locally
};
```

Rules:
- A day's `date` is derived: the Nth weekend day counting from `2026-10-10`, skipping Mon-Fri.
  Odd `day` → Saturday, even `day` → Sunday, `weekend = ceil(day / 2)`.
- `status` is user-controlled (checkbox), **not** date-controlled - a missed day stays open.
- `today` is computed against IST (`Asia/Kolkata`), same clock source as the Home + Calendar tab.
- Weekdays show a dimmed "Weekday - see DSA / Development" strip in the timeline.
- Phase 2 cards render `companies[]` as chips; the tab exposes a company filter built from the union of all tags.
- Every card has a **notes** box. Phase 1 completion means the note is written; that is the whole point of this track.

---

## 4. The Roadmap

### PHASE 1 - Learning Mode (F001-F036)

*Goal: stop reciting definitions. Be able to draw the mechanism on a whiteboard and defend the tradeoff.*

#### Block A - Operating Systems (F001-F010 · W01-W05)

| Day | Title | Topics | Drill | Done when |
|-----|-------|--------|-------|-----------|
| F001 | What an OS actually does | Kernel vs user mode, system calls, process lifecycle, PCB, context switch cost, `fork`/`exec`/`wait`, zombie & orphan processes | Trace the syscalls of a tiny program (`strace` in a Linux container, or `dtruss` on macOS) and annotate every line | You can explain memory-wise what happens on `fork()` and why a zombie exists |
| F002 | Processes vs threads | Address space sharing, user vs kernel threads, thread pools, cost of a thread, virtual threads, multiprocessing vs multithreading | Run the same workload as processes and as threads and measure both | You can say exactly what is shared and what is copied between the two |
| F003 | CPU scheduling | FCFS, SJF, SRTF, Round Robin, priority, MLFQ, preemption, starvation, ageing, convoy effect, turnaround vs waiting vs response time | Hand-compute a Gantt chart for 5 processes across 4 algorithms, then code an RR simulator | Given a workload you pick a scheduler and defend it with numbers |
| F004 | Synchronization | Critical section, race conditions, atomicity, mutex vs semaphore vs monitor, producer-consumer, readers-writers, dining philosophers | Solve producer-consumer with raw semaphores, then again with `BlockingQueue` | You can point at the exact line where the race lives, before running it |
| F005 | Deadlocks | Four Coffman conditions, resource allocation graph, prevention vs avoidance vs detection, Banker's algorithm, livelock, priority inversion | Write code that deadlocks two threads, then fix it by lock ordering | You can break a deadlock by naming which condition you removed |
| F006 | Memory management | Contiguous allocation, fragmentation, paging, segmentation, page tables, multi-level tables, TLB, address translation, MMU | Translate virtual to physical by hand through a 2-level page table with a TLB hit and a miss | You can compute effective access time with a TLB hit ratio |
| F007 | Virtual memory | Demand paging, page faults, FIFO / LRU / Optimal / Clock, Belady's anomaly, thrashing, working set, `mmap`, copy-on-write, swap | Simulate three replacement policies over the same reference string and compare fault counts | You can explain thrashing and what a real system does about it |
| F008 | File systems & storage | Inodes, journaling file systems, directories, hard vs soft links, buffer cache, `fsync` and durability, disk scheduling, SSD vs HDD access cost | Create a symlink and a hardlink, then explain the difference from inode counts | You can trace `open()` → inode → data blocks |
| F009 | I/O, IPC & the Linux toolbox | Interrupts vs polling, DMA, blocking vs non-blocking vs async I/O, `select`/`epoll`, pipes, shared memory, message queues, sockets, signals | Inspect a running JVM: `ps`, `top`, `lsof`, file descriptors, `ulimit` | You can explain "too many open files" and fix it |
| F010 | Containers & OS consolidation | Namespaces, cgroups, what a container really is, OOM killer, CPU throttling in containers, why JVM heap flags matter in Docker | One-page mental map of Block A + 30 rapid-fire self-quiz questions answered in writing | You can teach OS to someone using only your one-pager |

#### Block B - Databases (F011-F020 · W06-W10)

| Day | Title | Topics | Drill | Done when |
|-----|-------|--------|-------|-----------|
| F011 | Relational model & SQL foundations | Relational algebra, keys (primary/candidate/composite/foreign), joins (inner/left/right/full/self/anti/cross), set operations, three-valued NULL logic, logical order of a SELECT | Rebuild 10 queries from scratch on a seeded Postgres schema | You can name the true execution order of a SELECT, not the written order |
| F012 | Advanced SQL | `GROUP BY`/`HAVING`, window functions (`ROW_NUMBER`, `RANK`, `LAG`/`LEAD`), frames, CTEs and recursive CTEs, correlated subqueries, `EXISTS` vs `IN` | Solve 10 hard query problems: top-N per group, running totals, gaps & islands | No problem needs application-side post-processing |
| F013 | Schema design & normalization | Functional dependencies, 1NF to BCNF, anomalies, denormalization tradeoffs, surrogate vs natural keys, ER modelling, soft deletes, audit columns | Normalize a deliberately ugly schema to BCNF, then denormalize one table on purpose and justify it | You can argue both directions for the same table |
| F014 | Indexing & query planning | B+ tree vs hash indexes, clustered vs non-clustered, composite indexes & the leftmost-prefix rule, covering indexes, selectivity, `EXPLAIN ANALYZE`, when an index is ignored | Take a slow query, read its plan, add the right index, measure before/after | You can read a plan and name the operator that hurt |
| F015 | Storage engines & query execution | Pages and buffer pool, B-tree vs LSM-tree, WAL, query parser/optimizer/executor, join algorithms (nested loop, hash, merge), statistics | Compare a B-tree and an LSM-based store on a write-heavy workload | You can say why one wins writes and the other wins reads |
| F016 | Transactions & ACID | Atomicity/consistency/isolation/durability, WAL, commit & rollback, isolation levels, dirty read, non-repeatable read, phantom read, lost update, MVCC vs 2PL | Open two psql sessions and reproduce the anomalies by hand | You can produce each anomaly on demand and name the level that stops it |
| F017 | Locking in practice | Row vs table locks, shared vs exclusive, optimistic vs pessimistic locking, `SELECT ... FOR UPDATE`, DB deadlocks, connection pool exhaustion, `@Transactional` pitfalls | Force a DB deadlock between two sessions, read the deadlock log, then fix it | You can size a pool and defend the number |
| F018 | Replication, partitioning & sharding | Leader/follower, sync vs async replication, lag, partitioning vs sharding, shard keys, consistent hashing, CAP and PACELC, eventual consistency | Pick shard keys for three products and explain the hotspot you avoided | You can explain what breaks the day after you shard |
| F019 | NoSQL & polyglot persistence | Key-value vs document vs wide-column vs graph, Redis vs Postgres vs Mongo decision rules, access-pattern-first modelling | Write a one-page decision doc: which store for cart, session, feed, ledger, search, and why | You can reject a NoSQL store for a good reason, not a fashionable one |
| F020 | DBMS consolidation | Hot partitions, idempotency & unique constraints, migrations without downtime, index-per-query discipline | Design and defend the full schema for a real product (food delivery), with indexes derived from the top 8 queries | Every index you added maps to a query you wrote down |

#### Block C - Computer Networks (F021-F028 · W11-W14)

| Day | Title | Topics | Drill | Done when |
|-----|-------|--------|-------|-----------|
| F021 | Layers & the life of a request | OSI vs TCP/IP, encapsulation, MAC vs IP vs port, ARP, NAT, switches vs routers, "what happens when you type a URL" end to end | Write the URL-to-pixels story in full, layer by layer, with no gaps | Nobody can ask "and then what?" and stump you |
| F022 | TCP vs UDP | 3-way handshake, 4-way teardown, TIME_WAIT, sequence/ack numbers, retransmission, flow control & sliding window, congestion control, head-of-line blocking, when UDP wins | Capture a handshake with `tcpdump`/Wireshark and label every flag | You can explain why a "slow API" was actually a TCP problem |
| F023 | IP, subnetting & routing | IPv4 addressing, CIDR & subnet masks, private ranges, routing tables, default gateway, ICMP, `ping`/`traceroute`, IPv6 basics, DHCP | Subnet a /24 into 6 usable networks by hand | You can read a routing table out loud correctly |
| F024 | DNS & HTTP | Resolver flow, record types, TTL & caching, HTTP/1.1 vs HTTP/2 vs HTTP/3 (QUIC), keep-alive, multiplexing, methods & idempotency, status codes, cookies, CORS preflight | `dig` a domain through every step; then trigger and fix a real CORS failure | You can explain a CORS error without guessing |
| F025 | TLS & security | Symmetric vs asymmetric crypto, TLS handshake, certificates & chain of trust, hashing vs encryption vs encoding, salting & bcrypt, JWT vs session cookies, OWASP top hits | Generate a cert, serve HTTPS locally, and read the handshake | You can say exactly what a JWT protects and what it does not |
| F026 | Web plumbing | Load balancers L4 vs L7, reverse proxy, CDN & edge caching, sticky sessions, health checks, WebSockets vs SSE vs long polling, timeouts, retries, backoff & jitter | Put nginx in front of two app instances and demo failover | You can explain why naive retries make an outage worse |
| F027 | Network debugging & performance | `curl -v`, `dig`, `ss`/`netstat`, `tcpdump`, latency vs bandwidth vs throughput, RTT, p50/p95/p99, tail latency, MTU, packet loss | Diagnose three broken scenarios: DNS failure, closed port, TLS mismatch | You reach for a tool, not a guess |
| F028 | Networks consolidation | Full-path thinking: browser, DNS, CDN, load balancer, app, DB, and back | Draw the annotated end-to-end trace of one request through your own Spring Boot app, marking every layer and every place latency can hide | The diagram survives your own cross-examination |

#### Block D - OOPs & Object Design (F029-F036 · W15-W18)

| Day | Title | Topics | Drill | Done when |
|-----|-------|--------|-------|-----------|
| F029 | The four pillars, properly | Encapsulation, abstraction, inheritance, polymorphism, static vs dynamic dispatch, overriding vs overloading, composition over inheritance | Refactor a deep inheritance hierarchy into composition | You can say what each pillar buys you and where inheritance hurts |
| F030 | Classes, interfaces & the object model | Abstract class vs interface, default methods, `equals`/`hashCode` contracts, immutability, initialisation order, inner classes, coupling and cohesion | Build an immutable value type with correct equality and use it as a map key | You can pick abstract class or interface and justify it |
| F031 | SOLID in practice | Single responsibility, open/closed, Liskov, interface segregation, dependency inversion, with a violation and a fix for each | Take a 200-line "god class" and refactor it against all five principles | Adding a new behaviour touches no existing class |
| F032 | Creational & structural patterns | Singleton (and its thread-safety traps), Factory, Abstract Factory, Builder, Prototype; Adapter, Decorator, Facade, Proxy, Composite | Implement five of them on a small domain and name the problem each one solves | You can tell a Decorator from a Proxy without notes |
| F033 | Behavioural patterns | Strategy, Observer, State, Command, Template Method, Chain of Responsibility, Iterator | Replace a chain of `if/else` with Strategy and State | You can explain when a pattern is overkill |
| F034 | Design process & UML | Requirements to entities to relationships, class and sequence diagrams, anti-patterns (god class, anemic model), design by contract | Draw class and sequence diagrams for an e-commerce checkout | Someone else could code from your diagrams |
| F035 | LLD I - classic designs | Parking lot, elevator system, library management: classes, relationships, code skeleton | Design and code the skeleton for a parking lot | Adding a new vehicle type touches no existing class |
| F036 | LLD II & Phase 1 capstone | Rate limiter, Splitwise, BookMyShow seat booking with concurrency, thread-safety in design rounds | Design Splitwise and an in-memory rate limiter down to code-level classes | You can defend every interface you introduced |

---

### PHASE 2 - Interview Mode (F037-F050)

*Goal: no new theory. Timed rounds, real company questions, spoken out loud, self-scored.*

**Day format (every day in this phase, 2 hours):**
1. Two 45-minute timed mock rounds - answer out loud, whiteboard or blank page only, no notes open.
2. 30 minutes of self-scoring against a rubric: correctness, structure, tradeoffs, clarity.
3. Write down every gap; gaps become the next day's warm-up.

| Day | Round | Focus | Question set | Companies | Done when |
|-----|-------|-------|--------------|-----------|-----------|
| F037 | OS rapid round I | Processes, threads, scheduling | Process vs thread, context switch cost, deadlock conditions with a real example, scheduler choice for a given workload | Amazon, Microsoft, Adobe | 15 questions answered inside 45 minutes, none waffled |
| F038 | OS rapid round II | Memory & concurrency | Paging vs segmentation, virtual memory & thrashing, mutex vs semaphore, zombie/orphan, how Java threads map to OS threads, virtual threads | Google, Oracle, VMware | You survived three "why?" follow-ups per answer |
| F039 | DBMS round I | Indexing & schema | Why is this query slow, index choice, composite index order, clustered vs non-clustered, normalization vs denormalization for a given feature | Amazon, Flipkart, Walmart | Every answer started from access patterns |
| F040 | DBMS round II | Transactions & scale | Isolation levels and which anomaly each stops, MVCC, DB deadlock handling, optimistic vs pessimistic, sharding & its cost | Microsoft, Salesforce, Atlassian | You gave a wrong-then-corrected answer zero times |
| F041 | SQL query grind | Window functions & messy data | Top-N per group, running total, month-over-month growth, deduplication, gaps & islands, cohort retention | Meta, Amazon, Uber | 10 queries, first-attempt correct, under time |
| F042 | CN round I | TCP & the request path | TCP vs UDP with a use case each, handshake & teardown, congestion control, what happens when you type a URL, latency vs throughput | Cisco, Amazon, Microsoft | The URL question took 10 minutes and stayed structured |
| F043 | CN round II | Web protocols & security | DNS resolution & caching, HTTP/2 vs HTTP/3, CDN behaviour, load balancer L4 vs L7, CORS, JWT vs sessions, CSRF vs XSS | Google, Cloudflare, Atlassian | You corrected a deliberately wrong premise in the question |
| F044 | OOPs rapid round | Concepts & SOLID | Pillars with examples, abstract class vs interface, composition vs inheritance, SOLID violations to spot, pattern-recognition questions | Amazon, Microsoft, Adobe | Every answer came with a code-level example |
| F045 | LLD mock I | Object design under time | Parking lot, elevator system, BookMyShow seat booking with concurrency | Amazon, Uber, Arcesium | Classes, interfaces and concurrency story delivered in 45 minutes |
| F046 | LLD mock II | Systems as objects | Rate limiter, notification service, cab-matching domain model, logging library | Flipkart, Swiggy, Zeta | You handled "now add this requirement" without a rewrite |
| F047 | Cross-fire I | OS + DBMS scenarios | "What breaks and why": a deadlock in production, a slow query after a deploy, a connection pool exhausted, a thrashing server | Amazon, Microsoft, Uber | Each scenario traced to a root cause from first principles |
| F048 | Cross-fire II | CN + OOPs scenarios | A slow API that is really TCP, a CORS outage, a design that breaks when one requirement changes, a pattern misused | Google, Atlassian, Flipkart | You found the cause before the interviewer hinted |
| F049 | Dress rehearsal | All four subjects | Four 25-minute rounds back to back, one per subject, drawn from your gap list | Mixed panel | Every round finished on time, scored, with a gap list |
| F050 | Final full-loop mock | Everything | One round per subject plus one LLD round, back to back, timed, self-scored | Mixed panel | Scored, gaps listed, and the list is short |

---

## 5. Progress & UI Rules (for the website)

- **Card states**: `locked` (dim, yellow hairline), `today` (Bat-signal glow pulse), `done` (yellow fill + tick), `missed` (red hairline, still clickable - nothing is ever locked away).
- **Timeline**: 25 weekend blocks, each block a Sat/Sun pair, grouped under Phase 1 / Phase 2 headers with subject colour rails down the left edge.
- **Subject legend**: OS yellow, DBMS blue, CN green, OOPs titanium; each with its own mini progress bar.
- **Company filter** (Phase 2 only): chips built from the union of all `companies[]` values; selecting one dims non-matching cards.
- **Notes**: every card has a persisted notes box - Phase 1 is only "done" when the note exists.
- **Cross-links**: each card links sideways to that weekend's System Design and Project days, so the weekend reads as one unit.
- **Counters**: 50 weekend days · 25 weekends · 2 phases · 4 subjects.
- **State**: persist `status` + `notes` per day in localStorage under a versioned key (e.g. `batman.fund.v1`).
- Same clock source as the Home + Calendar tab: IST (`Asia/Kolkata`), never a day earlier than `2026-10-05`.

---

## 6. Consistency Check against `Plan.md`

| Plan.md requirement | Status |
|---|---|
| Fundamentals done only on Saturday and Sunday | ✅ 25 weekends × 2 days = 50 days |
| 2 hours each day | ✅ `daily_commitment: 2 hours`; Phase 2 days are 2 × 45 min mocks + 30 min scoring |
| Subjects: OS, DBMS, CN, OOPs | ✅ Blocks A-D, F001-F036 |
| System Design is not part of this tab | ✅ Removed; it lives in `SystemDesign.md` |
| Info blurb below the heading | ✅ Section 1 → `blurb` |
| Referenced by the Fundamentals tab | ✅ This file is the tab's data source |
| Weekends fit inside the 180 days | ✅ Last day is Sun 2027-03-28; the weekend of 3-4 Apr falls outside |

**Decisions beyond `Plan.md` (flagged for review)**
- **Two phases (36 learning days + 14 interview days)** are my proposal, copied in spirit from the Idea version. `Plan.md` gives no split.
- **Block sizes** (OS 10, DBMS 10, CN 8, OOPs 8): DBMS got the two days freed by moving System Design out, since it is the most-asked subject. Change if you want a different balance.
- **LLD lives in OOPs** (F035-F036, F045-F046) because it is object design. If `SystemDesign.md` also covers LLD, one of the two should drop it to avoid doing it twice.
- **Company tags** in Phase 2 are indicative, carried from the Idea version. Treat them as themes, not guarantees of what a company asks.
- **Time check with `Lifestyle.md`:** the holiday routine gives Fundamentals 12:00 pm-2:00 pm, which matches the 2 hours here.
