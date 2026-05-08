# Chapter 8 — The Trouble with Distributed Systems

## When to Use This Chapter

Consult this chapter when the task involves any of the following:

- Reasoning about **network failures**, timeouts, and partial failures in distributed systems
- Designing **failure detection** — heartbeats, timeouts, phi accrual detectors
- Discussing **clock synchronization**, NTP, time-of-day vs monotonic clocks, or clock skew bugs
- Handling **process pauses** — GC pauses, VM suspension, leader lease expiry
- Preventing stale leaders from causing damage — **fencing tokens**, lease-based locks
- Explaining **Byzantine fault tolerance** or system models (synchronous, partially synchronous, asynchronous)
- Differentiating **safety vs liveness** properties in distributed algorithm design
- Questions like: *"How do I detect node failure reliably?"*, *"Why can't I use wall-clock time for ordering events in a distributed system?"*, *"What is a fencing token?"*, *"What does 'partially synchronous' mean?"*, *"How do GC pauses cause distributed system bugs?"*

---

A single computer either works or doesn't. A distributed system can partially fail in complex ways. This chapter explores what can go wrong.

---

## Faults and Partial Failures

A computer is either fully functional or crashed — deterministic. A distributed system has *partial failures*: some parts work, others don't, in unpredictable ways. Partial failures are **non-deterministic** — sometimes work, sometimes fail, depending on timing.

**Failure modes to design for**: tolerate partial failure rather than assume perfect operation.

---

## Unreliable Networks

The internet and most datacenter networks are *asynchronous packet networks*. Packets may be:
- Lost (queue overflow, misconfigured router)
- Delayed arbitrarily (network congestion, full queue)
- Delivered out of order
- Duplicated

When you send a request and don't get a reply, you **cannot tell** whether:
- The request was lost
- The remote node is down
- The reply was lost
- The reply was delayed

The only safe response: **timeout**. After some timeout, declare the node unreachable.

**Timeouts**: Too short → false positives (trigger failover on slow node, adding load). Too long → slow fault detection.

**Network congestion**: Packets queued in switches, OS, and application. TCP flow control. Background processes, garbage collection, virtual machine pauses — all cause variable delays.

**Unbounded delays**: Unlike telephone circuits (synchronous networks with guaranteed bandwidth), packet networks have no upper bound on delay. Makes timeouts inherently imprecise.

Mitigation: **Deadline / phi accrual failure detectors** that adapt to observed network conditions.

---

## Unreliable Clocks

Each machine has its own clock (quartz oscillator). Clocks drift and must be synchronized via NTP (typically against GPS clocks or atomic clocks).

### Two Types of Clocks

**Time-of-day clock**: Returns the current date and time. Based on epoch (Unix time). Synchronized via NTP — can *jump backwards* or be adjusted. Do not use for measuring elapsed time.

**Monotonic clock**: Measures elapsed time between two points. Guaranteed to always move forward. Use for measuring timeouts and durations. Absolute value is meaningless across machines.

### Clock Synchronization Problems

- NTP sync can be imprecise (10–100ms over internet, sub-ms on local network)
- Network latency affects NTP accuracy
- Leap seconds can cause clock jumps or repeats
- VMs: clock jumps when the hypervisor pauses/resumes the VM
- Skewed clocks can cause incorrect ordering of events

**Confidence interval**: A clock's value isn't a precise moment but a range (e.g., ±95ms). Google's TrueTime API in Spanner exposes this explicitly.

### Ordering Events with Clocks

Dangerous to use timestamps to order events across nodes. "Last-write-wins" with timestamps: if clocks aren't perfectly synchronized, recent writes can be silently dropped.

**Logical clocks**: Lamport clocks, vector clocks — measure relative ordering of events, not wall-clock time. Safe for determining which event happened first.

### Process Pauses

A process can be *paused* for arbitrary durations without knowing it:
- Stop-the-world garbage collection
- Virtual machine suspend/resume
- OS context switch, sleep() call
- Disk I/O wait

During a pause, a process that holds a lease (distributed lock) may continue acting as if it does after the lease has expired. Another node may have acquired the lease. This is called a **GC pause problem**.

**Fencing tokens**: Monotonically increasing token issued with each lock grant. Storage server rejects writes with a token older than the last seen. Prevents expired lock holders from doing damage.

---

## Knowledge, Truth and Lies

### Distributed Decision-Making

A node cannot trust its own knowledge in isolation. **Quorums** — majority agreement — are the standard tool.

**"The node is dead" declaration**: Often made by the majority. Single node cannot declare itself leader if the majority doesn't agree.

### Byzantine Faults

Nodes that send *deliberately incorrect* or *misleading* messages. "Byzantine fault-tolerant" systems tolerate malicious/faulty nodes.

In most datacenter environments (all nodes under one organization), Byzantine faults are not assumed. Relevant for: blockchain, aerospace, multi-party financial systems.

**Byzantine generals problem**: *n* generals must agree on an action, but up to *f* may be traitors. Requires *n > 3f* nodes to tolerate *f* Byzantine faults.

### Weak Forms of Lying

Not full Byzantine faults, but worth handling:
- Network packets can be corrupted (use checksums)
- NTP can report incorrect time (use multiple servers, sanity-check jumps)
- Input validation (never trust user-provided data to be well-formed)

### System Models for Distributed Systems

**Timing assumptions**:
- *Synchronous model*: Bounded network delay, process pause, clock drift. Unrealistic.
- *Partially synchronous*: Behaves synchronously *most* of the time, but occasionally has unbounded delays. Realistic.
- *Asynchronous model*: No timing assumptions at all. Very restrictive.

**Crash assumptions**:
- *Crash-stop*: Node halts and never recovers.
- *Crash-recovery*: Node may halt but comes back, with durable state intact.
- *Byzantine*: Node may behave arbitrarily.

Most practical algorithms assume: **partially synchronous** + **crash-recovery**.

### Safety vs Liveness

**Safety**: Nothing bad happens. If violated, we can point to the exact moment it happened. Examples: uniqueness of locks, atomicity.

**Liveness**: Something good eventually happens. May not hold at every instant. Examples: eventual delivery, eventual consistency.

Correctness proofs usually define algorithms by: they preserve *safety* in all circumstances; they provide *liveness* eventually (possibly subject to assumptions about bounded delays).

---

## Key Terms

| Term | Definition |
|---|---|
| Partial failure | Some parts of the system fail while others work |
| Asynchronous network | No upper bound on message delay |
| Timeout | Mechanism for detecting node failure; hard to calibrate |
| Time-of-day clock | Wall-clock time; can jump backwards |
| Monotonic clock | Elapsed time measurement; always moves forward |
| NTP | Network Time Protocol — clock synchronization |
| Logical clock | Measures event ordering rather than wall-clock time |
| Lamport clock | Logical clock tracking causality |
| Vector clock | Per-node version counters tracking causal history |
| Process pause | Arbitrary suspension of a process (GC, VM, OS) |
| Fencing token | Monotonic token to prevent expired lock holders from causing damage |
| Byzantine fault | Node sends incorrect or malicious messages |
| Safety property | Nothing bad ever happens |
| Liveness property | Something good eventually happens |
| Partially synchronous | System usually meets timing bounds but occasionally doesn't |
