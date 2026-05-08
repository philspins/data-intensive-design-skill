---
name: ddia-reference
description: >
  Comprehensive reference for "Designing Data-Intensive Applications" (DDIA) by Martin Kleppmann.
  Use this skill whenever a user asks about distributed systems design, database internals, storage engines,
  data replication, partitioning/sharding, transactions, consistency models, consensus algorithms,
  stream or batch processing, or general data system architecture. Also trigger for questions about
  specific technologies covered in the book: LSM-trees, B-trees, SSTables, Kafka, Zookeeper, Flink,
  MapReduce, CRDT, two-phase commit, Raft/Paxos, linearizability, serializability, MVCC, etc.
  This is your go-to reference for any DDIA concept — use it even for casual questions like
  "how does replication lag work?" or "what's the difference between OLTP and OLAP?"
---

# Designing Data-Intensive Applications — Reference Skill

Notes from Martin Kleppmann's *Designing Data-Intensive Applications* (O'Reilly).

## How to use this skill

For any question about data systems, load the relevant chapter reference file below and answer from it.
If the question spans multiple chapters, load each relevant one.

## Chapter Index

| Chapter | Topic | Reference File |
|---|---|---|
| 1 | Reliability, Scalability, Maintainability | `references/ch1-foundations.md` |
| 2 | Data Models & Query Languages | `references/ch2-data-models.md` |
| 3 | Storage & Retrieval | `references/ch3-storage.md` |
| 4 | Encoding & Evolution | `references/ch4-encoding.md` |
| 5 | Replication | `references/ch5-replication.md` |
| 6 | Partitioning | `references/ch6-partitioning.md` |
| 7 | Transactions | `references/ch7-transactions.md` |
| 8 | Trouble with Distributed Systems | `references/ch8-distributed-systems.md` |
| 9 | Consistency & Consensus | `references/ch9-consistency.md` |
| 10–12 | Batch Processing, Stream Processing, Future of Data Systems | `references/ch10-ch12.md` |

## Quick Topic Lookup

- **Performance / percentiles / SLAs** → ch1
- **SQL vs NoSQL, document vs relational, graph DBs** → ch2
- **B-trees, LSM-trees, SSTables, indexes, OLTP vs OLAP, column storage** → ch3
- **Serialization formats (Protobuf, Avro, Thrift), schema evolution** → ch4
- **Leader/follower, replication lag, multi-leader, leaderless, quorums** → ch5
- **Sharding, consistent hashing, secondary indexes, rebalancing** → ch6
- **ACID, dirty reads, phantom reads, serializability, 2PL, SSI** → ch7
- **Network faults, clocks, process pauses, Byzantine faults** → ch8
- **Linearizability, causality, total order broadcast, Paxos, Raft, 2PC** → ch9
- **MapReduce, Hadoop, dataflow engines (Spark, Flink, Tez)** → ch10-ch12
- **Event streams, Kafka, change data capture, stateful processing** → ch10-ch12
- **Lambda architecture, unbundling databases, end-to-end correctness** → ch10-ch12
