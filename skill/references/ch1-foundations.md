# Chapter 1 — Reliable, Scalable, and Maintainable Applications

## When to Use This Chapter

Consult this chapter when the task involves any of the following:

- Designing or evaluating a system for **reliability** — fault tolerance, error handling, hardware/software/human failure modes, rollback strategies
- Discussing **scalability** — handling increased load, choosing between vertical vs horizontal scaling, elastic infrastructure, capacity planning
- Analysing **performance** — response time, throughput, latency percentiles (p50/p95/p99), tail latency, SLAs, SLOs
- Reviewing **system architecture** for maintainability — operability, reducing complexity, evolvability, designing for change
- Questions like: *"How should I design this for high availability?"*, *"What's the difference between latency and response time?"*, *"How do I think about SLAs?"*, *"What makes a system fault-tolerant?"*, *"How does Twitter handle fan-out?"*

---

Data-intensive applications typically need to: store data (databases), speed up reads (caches), search data (search indexes), send messages asynchronously (stream processing), and periodically crunch data (batch processing).

The three concerns that matter most in most software systems: **Reliability**, **Scalability**, **Maintainability**.

---

## Reliability

The system should work *correctly* even in the face of adversity (hardware/software faults, human error).

**Fault vs Failure**: A *fault* is a component deviating from spec. A *failure* is when the whole system stops serving users. Fault-tolerant systems anticipate faults and cope with them. **Prefer tolerating faults over preventing them.**

### Types of faults

- **Hardware faults**: Hard disk crashes, RAM failure, power outage. Historically handled with redundancy (RAID, dual PSU). Modern trend: tolerate loss of entire machines via software fault-tolerance — enables rolling upgrades with no downtime.
- **Software errors**: Systematic faults that affect many nodes simultaneously (e.g., runaway process consuming shared resources, bugs triggered by unusual input, cascading failures). Harder to handle than hardware faults — they often lie dormant until an unusual condition triggers them.
- **Human errors**: Config errors are the leading cause of outages. Mitigations:
  - Admin interfaces that make the right thing easy and the wrong thing hard
  - Non-production sandbox environments
  - Automated testing (unit, integration, manual)
  - Fast rollback mechanisms; gradual rollout
  - Detailed monitoring (performance metrics, error rates — *telemetry*)
  - Good management practices and training

---

## Scalability

The system's ability to cope with increased load.

### Describing Load

Load is described with *load parameters*: requests/sec, ratio of reads to writes, number of simultaneous active users, cache hit rate, etc. The right parameters depend on the architecture.

**Twitter example**: Two operations — post tweet (4.6k req/sec, 12k peak) and home timeline (300k req/sec).
- Approach 1: Global tweet table + fan-out on read using JOIN
- Approach 2: Per-user home timeline cache + fan-out on write

Approach 2 dominates because read rate vastly exceeds write rate. But high-follower users (30M+) make Approach 2 expensive. Twitter uses a **hybrid**: fan-out on write for most users, fan-out on read for celebrities, merged at read time.

### Describing Performance

When load increases, ask: (a) how does performance degrade with fixed resources? (b) how much must you scale up to keep performance constant?

- **Batch systems**: care about *throughput* (records per second)
- **Online systems**: care about *response time* (distribution, not just average)

> **Latency vs Response Time**: Response time is what the client observes. Latency is time the request waits to be handled.

**Use percentiles, not averages.** p50 (median), p95, p99, p999 reveal tail latency.
- Amazon targets p999 internally because slowest users are often highest-value customers
- p9999 typically too expensive to optimize

**SLOs and SLAs**: Contracts defining expected performance/availability (e.g., p50 < 200ms, p99 < 1s). Missed SLAs entitle customers to refunds.

**Head-of-line blocking**: Even a single slow backend call can delay an end-user request that fans out to many services in parallel. Measure response times from the *client side*.

### Approaches for Coping with Load

- **Vertical scaling** (scale up): bigger machine
- **Horizontal scaling** (scale out): distribute across many smaller machines
- **Elastic scaling**: auto-add resources when load spikes

Distributing stateless services is easy. Distributing stateful data (databases) introduces significant complexity. Rule of thumb: keep DB on a single node until forced to do otherwise.

Architecture is often highly specific to the application's load profile — there is no one-size-fits-all scalable architecture.

---

## Maintainability

The majority of software cost is ongoing maintenance. Three principles:

- **Operability**: Make routine tasks easy for operations teams.
- **Simplicity**: Make it easy for new engineers to understand the system. Remove *accidental complexity* — complexity not inherent in the problem being solved. Best tool: **abstraction** (hiding implementation behind clean APIs).
- **Evolvability** (aka *extensibility* or *plasticity*): Make it easy to change the system as requirements evolve. Agile practices provide a framework for this at the process level; good architecture does it at the code level.

### Operability checklist — what a good operations team needs:
- Monitoring + quick restore
- Tracking root causes of problems
- Keeping platforms up to date
- Understanding inter-system dependencies
- Capacity planning (anticipating problems)
- Defined deployment, config management, and rollback procedures
- Maintaining security
- Preserving organisational knowledge

---

## Key Terms

| Term | Definition |
|---|---|
| Fault | One component deviating from spec |
| Failure | System stops providing service |
| Fault-tolerant / resilient | Anticipates and copes with faults |
| Throughput | Records processed per second (batch) |
| Response time | Client-observed time from request to response |
| Latency | Time a request spends waiting |
| p99 / p999 | 99th / 99.9th percentile response time |
| SLO | Service Level Objective (internal target) |
| SLA | Service Level Agreement (contractual) |
| Vertical scaling | Moving to a bigger machine |
| Horizontal scaling | Distributing across many machines |
| Elastic | Auto-scales with load |
| Accidental complexity | Complexity not inherent in the problem |
| Abstraction | Hiding implementation behind a clean interface |
