# Chapter 7 — Transactions

## When to Use This Chapter

Consult this chapter when the task involves any of the following:

- Explaining or applying **ACID properties** — atomicity, consistency, isolation, durability
- Choosing or reasoning about **transaction isolation levels**: read committed, snapshot isolation, serializable
- Diagnosing concurrency bugs: **dirty reads, dirty writes, read skew, lost updates, write skew, phantom reads**
- Implementing safe **read-modify-write cycles** (atomic operations, SELECT FOR UPDATE, compare-and-set)
- Understanding **MVCC** (multi-version concurrency control) in PostgreSQL, MySQL, Oracle
- Comparing **Two-Phase Locking (2PL)** vs **Serializable Snapshot Isolation (SSI)**
- Questions like: *"What isolation level do I need?"*, *"How do I prevent lost updates?"*, *"What is write skew and how do I avoid it?"*, *"What's the difference between repeatable read and snapshot isolation?"*, *"Is this transaction safe under concurrent access?"*

---

A *transaction* is a way for an application to group several reads and writes into a logical unit — either the whole thing succeeds (commit) or the whole thing fails (abort/rollback).

Transactions simplify error handling: on failure, just retry.

---

## ACID

**Atomicity**: All writes in a transaction succeed or all are rolled back. Nothing is left half-done. "All-or-nothing."

**Consistency**: Application-defined invariants are always satisfied (e.g., debits == credits). This is actually the application's responsibility, not the DB's. The C in ACID is a bit of a stretch.

**Isolation**: Concurrent transactions behave as if they ran serially. Each transaction is unaware of others. Formally: *serializability* (rarely fully enforced — too expensive).

**Durability**: Once a transaction commits, its data is not lost even if the system crashes. Involves WAL, replication, etc.

BASE (Basically Available, Soft state, Eventual consistency) — a contrasting philosophy for highly available NoSQL systems. Deliberately vague.

---

## Weak Isolation Levels

Full serializability is expensive. Most databases offer weaker isolation levels.

### Read Committed (most common default)

Two guarantees:
1. **No dirty reads**: Only see data that has been committed.
2. **No dirty writes**: Only overwrite data that has been committed.

Prevents: dirty reads (reading uncommitted data), dirty writes (overwriting uncommitted data).

**Does NOT prevent**: read skew (non-repeatable reads).

Implementation: Row-level locks for writes. For reads: keep old committed value alongside the new uncommitted value; serve old value to concurrent readers until write commits.

### Snapshot Isolation (Repeatable Read)

Each transaction reads from a consistent *snapshot* of the database as it was at the start of the transaction. Even if other transactions commit while this transaction runs, they're invisible to it.

Useful for: long-running read-only queries (backups, analytics), preventing read skew.

**MVCC** (Multi-Version Concurrency Control): Database keeps multiple versions of each row. Each transaction sees only the versions that were committed before it started.

Implementation: Each row tagged with the transaction IDs that created and deleted it. A transaction only sees rows created by earlier committed transactions and not deleted by earlier committed transactions.

PostgreSQL, MySQL InnoDB, Oracle, SQL Server all support snapshot isolation (may call it "repeatable read").

### Lost Updates

Two transactions do read-modify-write. One's update may overwrite the other's. Examples: counter increment, account balance update, wiki edit.

Solutions:
- **Atomic write operations**: `UPDATE table SET value = value + 1 WHERE id = x` — takes exclusive lock, no read-modify-write needed. MongoDB, Redis provide atomic operations.
- **Explicit locking**: `SELECT ... FOR UPDATE` — application explicitly locks rows.
- **Automatic lost update detection**: DB detects and aborts. PostgreSQL, Oracle, SQL Server do this. MySQL InnoDB does not.
- **Compare-and-set**: `UPDATE SET value = new WHERE value = old` — only updates if unchanged.

In leaderless/multi-leader replication, lost update detection is harder. LWW (last-write-wins) does not prevent lost updates.

### Write Skew

Transaction reads some rows, makes a decision based on what it read, then writes. But by the time it writes, the premise of its decision is no longer true (another transaction changed the rows it read).

Classic example: two doctors try to go off-call simultaneously. Each checks "is someone else on call?" Both see yes, both go off-call. Now nobody's on call.

**Not** prevented by snapshot isolation. The rows read are not the rows written — so neither atomicity nor dirty write prevention helps.

Solutions:
- Explicit `SELECT ... FOR UPDATE` to lock rows read
- Detect and abort (requires true serializability)

**Phantom reads**: Write skew where the rows read don't yet exist (query returns empty set, then another transaction inserts matching rows). `SELECT FOR UPDATE` doesn't help on missing rows.

*Materializing conflicts*: Explicitly create lock rows in the DB for each possible phantom scenario. Hacky; prefer serializability.

---

## Serializability

The strongest isolation level. The execution of concurrent transactions is equivalent to *some* serial execution.

### Actual Serial Execution

Execute one transaction at a time on a single CPU thread. Only feasible because:
- RAM is cheap enough to fit the whole dataset in memory (no blocking disk I/O)
- OLTP transactions are typically short and touch few rows (contrast with analytics)

Stored procedures: to avoid multiple network round-trips per transaction, the entire transaction logic runs server-side as a single stored procedure call. Used by: VoltDB, H-Store, Redis (Lua scripts).

Partitioning: scale by partitioning data and running one thread per partition. Cross-partition transactions are expensive and require coordination.

### Two-Phase Locking (2PL)

Traditional approach. Many concurrent reads, but writes require exclusive access.

Rules:
- If transaction A reads an object and B wants to write it: B must wait for A to commit or abort.
- If A writes an object and B wants to read or write it: B must wait for A.

**Predicate locks / index-range locks**: Lock all rows matching a search condition — prevents phantoms. Index-range locks are more practical (lock an index range).

**Deadlocks**: Transaction A waits for B; B waits for A. DB detects and aborts one. Application should retry.

2PL has significantly worse performance than weak isolation (lock contention, deadlocks). Latency is highly variable.

### Serializable Snapshot Isolation (SSI)

Optimistic concurrency control. Transactions execute without blocking, then at commit time check if isolation was violated. If so, abort and retry.

Based on snapshot isolation + detecting *serialization conflicts*:
- Track reads from stale (now-updated) premise
- On commit, check if any premise became invalid

Better performance than 2PL in many workloads (no waiting). Works well with short transactions and low contention. PostgreSQL uses SSI. Contention-heavy workloads cause many aborts.

---

## Key Terms

| Term | Definition |
|---|---|
| ACID | Atomicity, Consistency, Isolation, Durability |
| Dirty read | Reading uncommitted data from another transaction |
| Dirty write | Overwriting uncommitted data from another transaction |
| Read skew | Seeing different snapshots of the DB during a single transaction |
| Snapshot isolation | Each transaction reads from a consistent snapshot at its start time |
| MVCC | Multi-Version Concurrency Control — DB stores multiple row versions for concurrent readers |
| Lost update | Two transactions read-modify-write, one's update gets silently overwritten |
| Write skew | Transaction makes decision based on read, but premise is invalid by the time it writes |
| Phantom read | Query returns different rows on two reads because another transaction inserted/deleted rows |
| Serializability | Strongest isolation — concurrent transactions equivalent to some serial order |
| 2PL | Two-Phase Locking — readers and writers block each other |
| SSI | Serializable Snapshot Isolation — optimistic concurrency with conflict detection at commit |
| Predicate lock | Lock on all rows matching a query condition (prevents phantoms) |
