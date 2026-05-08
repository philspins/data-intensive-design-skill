# Chapter 5 — Replication

## When to Use This Chapter

Consult this chapter when the task involves any of the following:

- Designing or explaining **database replication**: single-leader, multi-leader, or leaderless (Dynamo-style)
- Discussing **replication lag** and its consequences — read-your-writes, monotonic reads, consistent prefix reads
- Handling **leader failover**, replica promotion, or split-brain scenarios
- Understanding **conflict resolution** in multi-leader or leaderless systems (LWW, CRDTs, version vectors)
- Evaluating **quorum reads/writes**, sloppy quorums, and hinted handoff
- Discussing specific systems: **PostgreSQL streaming replication, MySQL binlog, Cassandra, DynamoDB, Riak, MongoDB replica sets**
- Questions like: *"How does replication lag affect my application?"*, *"What happens when a leader fails?"*, *"How does Cassandra handle concurrent writes?"*, *"What's the difference between synchronous and asynchronous replication?"*, *"How do I implement read-your-writes consistency?"*

---

Replication: keeping a copy of the same data on multiple machines.

Reasons: reduce latency (geo-proximity), increase availability (tolerate node failure), scale read throughput.

Assume dataset fits on one machine. Partitioning for larger datasets is Chapter 6.

The hard part: **handling changes** to replicated data.

---

## Leaders and Followers (Single-Leader / Master-Slave)

**Leader** (master/primary): Accepts all writes. Writes to local storage and to a *replication log* (change stream).

**Followers** (replicas/slaves/secondaries/hot standbys): Receive the replication log and apply writes in the same order. Serve read queries.

All writes go to the leader. Reads can go to any replica (with consistency caveats — see below).

### Synchronous vs Asynchronous Replication

**Synchronous**: Leader waits for follower acknowledgment before confirming write to client. Follower is guaranteed to be up to date. Problem: if follower is slow/unavailable, leader blocks.

**Semi-synchronous**: One follower is synchronous, others async. If the sync follower fails, another is promoted to sync. Guarantees at least two up-to-date copies.

**Asynchronous** (most common): Leader doesn't wait. Write confirmed quickly. Risk: if leader fails before followers catch up, **acknowledged writes can be lost** (durability trade-off).

### Setting Up New Followers

1. Take a consistent snapshot of leader (without locking if possible)
2. Copy snapshot to new follower
3. Follower connects to leader and requests all changes since snapshot (using *log sequence number* / *binlog coordinates*)
4. Follower has *caught up* when it has processed all changes

### Handling Node Outages

**Follower failure (catch-up recovery)**: Follower knows last transaction processed. On restart, request all changes since that point from leader.

**Leader failure (failover)**:
1. Detect leader has failed (timeout)
2. Elect a new leader (most up-to-date replica — by consensus or configured controller node)
3. Reconfigure clients to send writes to new leader
4. Old leader must become follower when it returns

**Failover pitfalls**:
- New leader may not have received all writes from old leader → lost writes (especially dangerous if writes were acknowledged to clients)
- Conflict if old leader comes back thinking it's still leader ("split brain")
- Detecting leader failure: timeouts can be too short (unnecessary failovers) or too long (slow recovery)
- Some teams prefer **manual failover** for safety

### Replication Logs (Implementation)

- **Statement-based**: Forward SQL statements to followers. Problems: non-deterministic functions (NOW(), RAND()), auto-increment, side effects.
- **WAL shipping**: Ship the write-ahead log. Closely tied to storage engine internals. Can't replicate across versions.
- **Logical (row-based) log**: Decoupled from storage engine. Describes changes at row level (insert: all new values; update: row identifier + changed values; delete: row identifier). Backward compatible. Useful for external consumers (CDC).
- **Trigger-based**: Custom application-level logic via DB triggers. Flexible but higher overhead and more error-prone.

---

## Problems with Replication Lag

Async replication → followers can fall behind → **replication lag**. Usually milliseconds. Can grow to seconds or minutes under load.

### Read-Your-Own-Writes (Read-After-Write Consistency)

User writes data, then reads it back but may hit a lagging replica and see stale data. Solutions:
- Read from leader for data the user may have modified (e.g., own profile)
- Track when user last updated; read from leader if less than 1 minute ago
- Client remembers timestamp of last write; route reads to replicas caught up to that point

Cross-device complication: user writes on one device, reads on another. Need to centralize "last update timestamp."

### Monotonic Reads

User reads from replica A, then replica B (which is more lagged). Sees data appear to go *backwards* in time. Fix: user always reads from same replica (e.g., based on hash of user ID).

### Consistent Prefix Reads

In partitioned databases, writes happen in different orders on different partitions. A reader might see an answer before the question. Fix: causally related writes go to same partition, or track causal dependencies.

---

## Multi-Leader Replication

Multiple nodes accept writes. Each leader is also a follower of the other leaders.

**Use cases**:
- Multi-datacenter (one leader per DC)
- Clients with offline operation (each device is a "leader")
- Collaborative editing (each client buffers changes locally)

**Biggest problem: write conflicts**. Both leaders accept a conflicting write. Must resolve.

### Conflict Resolution

**Avoid conflicts**: Route all writes for a particular record to the same leader. Simple and effective.

**Converge to consistent state**:
- Last-write-wins (LWW): Each write gets a timestamp; highest timestamp wins. Risk of data loss.
- Higher-replica-ID wins: Arbitrary; loses data.
- Merge values (e.g., concat strings; union sets)
- Record conflict, let application/user resolve (e.g., CouchDB's conflict flagging)

**CRDTs** (Conflict-free Replicated Data Types): Data structures designed to automatically resolve conflicts in sensible ways.

**Operational Transformation**: Algorithm used by collaborative editors (Google Docs).

---

## Leaderless Replication

No designated leader. Clients write to several replicas in parallel. Used by: Amazon Dynamo, Cassandra, Riak, Voldemort.

### Quorums

Given *n* replicas, write to *w* replicas, read from *r* replicas. As long as **w + r > n**, reads overlap with writes → can always find the most recent value.

Common: n=3, w=2, r=2. Or n=5, w=3, r=3.

Lower w or r → lower latency, higher availability, but more chance of stale reads.

### Sloppy Quorums and Hinted Handoff

If some of the *n* home replicas are unavailable, write to some other reachable nodes temporarily (*sloppy quorum*). When home nodes recover, those nodes forward the writes (*hinted handoff*). Increases write availability but breaks the quorum guarantee.

### Detecting Stale Reads

- **Read repair**: On a read, client detects stale values (from version comparison) and writes back the newer value.
- **Anti-entropy**: Background process continuously compares replicas and copies missing data. May have significant lag.

### Limitations of Quorum Consistency

Even with w + r > n, edge cases allow stale reads:
- Sloppy quorums bypass the strict overlap
- Concurrent writes require conflict resolution (same as multi-leader)
- Write partially fails — rolled back on some but not others
- Clock skew with LWW

Dynamo-style DBs are generally optimized for **eventual consistency** (high availability), not strong consistency.

### Concurrent Writes and Versioning

**Last-write-wins (LWW)**: Discard older writes. Loses data. Only safe for truly immutable keys (e.g., cache).

**Version vectors** (vector clocks): Each replica tracks a version number for each key. On merge, keep the union of values, and the max version for each replica. Client must send the version it knows about when writing, so DB can detect concurrent vs sequential writes.

---

## Key Terms

| Term | Definition |
|---|---|
| Leader / master | Node that accepts all writes |
| Follower / replica / slave | Node that receives and applies replication log |
| Replication lag | Delay between write on leader and application on follower |
| Failover | Promoting a follower to leader after leader failure |
| Read-your-writes consistency | Guarantee that you always read your own writes |
| Monotonic reads | Guarantee that you never read older data after having read newer data |
| Consistent prefix reads | Guarantee you never see writes out of causal order |
| Multi-leader | Multiple nodes accept writes; conflicts must be resolved |
| CRDT | Conflict-free Replicated Data Type — data structure with automatic conflict resolution |
| Leaderless / Dynamo-style | Any replica accepts writes; quorums determine consistency |
| Quorum | Minimum number of nodes that must agree on a read/write (w + r > n) |
| Sloppy quorum | Writing to available nodes outside home set for higher availability |
| Hinted handoff | Temporarily held writes forwarded to home node when it recovers |
| Read repair | Fixing stale replicas during a read operation |
| Version vector | Per-replica version counters for detecting concurrent writes |
