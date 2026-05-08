# Chapter 9 — Consistency and Consensus

## When to Use This Chapter

Consult this chapter when the task involves any of the following:

- Explaining or comparing **consistency models**: linearizability, causal consistency, eventual consistency
- Discussing the **CAP theorem** or PACELC — and their practical implications
- Implementing or reasoning about **distributed locking**, leader election, or unique constraints
- Understanding **consensus algorithms**: Paxos, Raft, Zab, Viewstamped Replication
- Explaining **Two-Phase Commit (2PC)** and its failure modes (in-doubt transactions, coordinator crashes)
- Understanding **total order broadcast** and its equivalence to consensus
- Discussing **ZooKeeper or etcd** — what they provide and when to use them
- Ordering events with **Lamport timestamps** or **vector clocks**
- Questions like: *"What is linearizability and do I need it?"*, *"How does Raft work?"*, *"Why is 2PC considered blocking?"*, *"What does ZooKeeper actually give me?"*, *"Is my system linearizable?"*, *"What's the difference between linearizability and serializability?"*

---

How to make distributed systems behave correctly despite unreliable networks and partial failures.

---

## Consistency Guarantees

**Eventual consistency**: If you stop writing, all replicas will *eventually* converge to the same value. Weak — doesn't say when, and allows many intermediate states.

This chapter explores stronger consistency models.

---

## Linearizability

The strongest consistency model. Makes a distributed system *appear* as if there is a single copy of the data, and all operations are atomic.

**Formal definition**: Once a read returns a new value (written by a concurrent write), all subsequent reads must also return that new value (or newer). The system behaves as if operations took effect at a single point in time within their duration.

Not the same as serializability (which is about transaction isolation). Linearizability is about single-object, single-operation recency guarantees.

### Uses of Linearizability

- **Leader election** (e.g., ZooKeeper/etcd): Only one node must be leader at a time — requires a linearizable lock.
- **Distributed locking**: A lock must be linearizable to be safe.
- **Unique constraints**: "Only one user with this username" — requires linearizable writes.
- **Cross-channel timing dependencies**: E.g., file upload + resize job communicated via different channels — a race condition requires linearizability.

### Implementing Linearizability

- Single-leader replication: Potentially linearizable (if reads go to leader or synchronous follower).
- Multi-leader: Not linearizable (concurrent writes on different leaders).
- Leaderless (Dynamo-style): Not linearizable (quorums can produce race conditions).

**Consensus algorithms** (Zookeeper, etcd): Linearizable. They prevent split-brain.

### CAP Theorem

**Consistency** (linearizability), **Availability**, **Partition tolerance** — you can have at most two of three.

In practice: network partitions are unavoidable. So the real choice is: **linearizability or availability** during a partition.

*C*: Linearizability (not "consistency" in the ACID sense)  
*A*: Every request gets a response (not just "available in the dictionary sense")  
*P*: The system continues to operate when the network is partitioned

CAP is a narrow theorem, often over-applied. Better framing: **PACELC** — even without partitions, there's a trade-off between latency and consistency.

### Cost of Linearizability

If two datacenters are partitioned, a linearizable system must stop accepting writes (or reads on some nodes) until the partition heals. CPU performance: multi-core machines require memory barriers to enforce linearizable behavior between cores.

---

## Ordering Guarantees

Ordering pervades distributed systems: replication logs, transaction serialization, causality.

### Causal Ordering

Events have *causal dependencies* — question before answer, read before write, cause before effect. Causally ordered systems guarantee that if event A caused event B, then all processes see A before B.

**Causality vs total order**: Linearizability implies total order (any two events can be compared). Causality is a weaker partial order — only causally related events are ordered; concurrent events have no order.

**Causal consistency**: Weaker than linearizability but stronger than eventual consistency. Sufficient for many applications.

### Sequence Numbers and Lamport Timestamps

**Lamport timestamps**: Each node maintains a counter. Every operation increments the counter. On communication, receiver takes max(own counter, received counter) + 1. This provides a *total order* consistent with causality.

Limitation: you can't tell from a Lamport timestamp alone whether two operations are concurrent or causally related.

**Sequence number generators**: In single-leader systems, the leader generates monotonically increasing sequence numbers. Can be used as a total order. Multi-leader systems lack a single authority — various tricks (odd/even, time-prefix, pre-allocated ranges) give per-node uniqueness but not global total order.

### Total Order Broadcast

Protocol that guarantees:
1. **Reliable delivery**: No messages lost; if delivered to one node, delivered to all.
2. **Total order**: All nodes deliver messages in the same order.

This is equivalent to consensus. Used by ZooKeeper and etcd.

Can implement a linearizable register on top of total order broadcast: broadcast a message to acquire a write, only commit if your message was the first.

---

## Distributed Transactions and Consensus

**Consensus**: Get several nodes to agree on a single value. Needed for: leader election, atomic broadcast, linearizable operations.

**Impossible result (FLP)**: In a fully asynchronous system with even one crash-faulty node, it is impossible to guarantee consensus. *Practical systems bypass this by using timeouts (partially synchronous assumption).*

### Two-Phase Commit (2PC)

Protocol for atomic transactions across multiple nodes (all commit or all abort). Not the same as 2PL.

**Participants**: Resource managers (DB nodes).  
**Coordinator**: Orchestrates the commit process (often embedded in the application or a dedicated transaction manager).

**Phases**:
1. **Prepare phase**: Coordinator sends "prepare" to all participants. Each either votes "yes" (promises to commit if asked) or "no" (aborts).
2. **Commit phase**: If all vote yes → coordinator sends "commit" to all. If any vote no → coordinator sends "abort" to all.

**Commit point**: Once coordinator decides and writes its decision to a WAL, the decision is irrevocable.

**Failure modes**:
- If coordinator crashes *after* sending prepare but *before* sending commit: participants in "uncertain" state, holding locks, can't proceed without coordinator. This is the "*in-doubt*" problem.
- 2PC is called a *blocking* atomic commit protocol — participants block waiting for coordinator.

**Heuristic decisions**: Some systems allow participants to abandon 2PC and decide unilaterally after a long timeout (breaks atomicity — use only as a last resort).

### Three-Phase Commit (3PC)

Non-blocking in theory. Assumes a bounded network delay (synchronous model). Not used in practice.

### Distributed Transactions in Practice

Two types:
1. **Database-internal distributed transactions**: All nodes run the same database software. Efficient — can optimize internally.
2. **Heterogeneous distributed transactions** (XA transactions): Coordinator is a separate component (e.g., Java EE JTA). Spans different systems. Difficult and brittle.

Problems with XA/2PC in practice:
- Coordinator is a single point of failure (if coordinator crashes, participants hold locks indefinitely)
- Application servers are not stateless (coordinator log must be on durable storage)
- Incompatibilities between XA implementations

### Fault-Tolerant Consensus

Properties:
- **Uniform agreement**: No two nodes decide differently.
- **Integrity**: No node decides twice.
- **Validity**: Decided value was proposed by some node.
- **Termination** (liveness): Every non-crashed node eventually decides.

Algorithms: **Paxos**, **Raft**, **Zab** (ZooKeeper), **Viewstamped Replication (VSR)**.

All are leader-based: one node is the leader and proposes values. Other nodes accept or reject.

**Leader election as consensus**: To agree on a new leader, you need consensus. Consensus requires a leader. This is handled by epoch numbers — nodes vote for a new leader per epoch. A leader can act only if it has a quorum of support for its epoch.

**Cost of consensus**: Requires majority quorum, at least 2 message round-trips per decision. Not cheap. Good for low-frequency critical decisions (leader election, lock acquisition, schema changes) not high-throughput data writes.

### ZooKeeper and etcd

ZooKeeper and etcd are fault-tolerant key-value stores that implement total order broadcast (and therefore consensus).

Uses:
- **Distributed locks / leader election**: Ephemeral nodes + watches
- **Service discovery**: Register services, watch for changes
- **Membership and coordination**: Track which nodes are alive
- **Cluster metadata**: Configuration, routing tables

ZooKeeper off-loads the hard distributed systems problems from application code. Use it as an "outsourced" coordination service.

---

## Key Terms

| Term | Definition |
|---|---|
| Linearizability | Appears as single copy of data; operations are instantaneous |
| CAP theorem | Can't have all three: Consistency (linearizability), Availability, Partition tolerance |
| Causal consistency | Events with causal relationships are seen in order by all nodes |
| Lamport timestamp | Logical timestamp providing total order consistent with causality |
| Total order broadcast | All nodes receive messages in same order (equivalent to consensus) |
| Consensus | Getting nodes to agree on a single value |
| 2PC | Two-Phase Commit — atomic commit across multiple nodes (blocking) |
| In-doubt transaction | Transaction waiting for coordinator decision; holds locks |
| Paxos | Foundational consensus algorithm |
| Raft | Consensus algorithm designed for understandability |
| Zab | Consensus algorithm used by ZooKeeper |
| Epoch number | Generation counter to handle multiple leaders; higher epoch wins |
| ZooKeeper | Coordination service implementing fault-tolerant total order broadcast |
