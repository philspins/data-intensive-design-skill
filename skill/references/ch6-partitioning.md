# Chapter 6 — Partitioning

## When to Use This Chapter

Consult this chapter when the task involves any of the following:

- Designing a **sharding strategy** for a large dataset — key-range vs hash partitioning
- Dealing with **hot spots** or uneven data distribution across nodes
- Implementing or explaining **secondary indexes** in a partitioned system (local vs global indexes, scatter/gather)
- Planning a **rebalancing strategy** — fixed partitions, dynamic splitting, proportional-to-nodes
- Routing requests in a distributed cluster — ZooKeeper-based routing, gossip protocols, client-side routing
- Discussing specific systems: **Cassandra, DynamoDB, MongoDB, Elasticsearch, HBase, Couchbase, Kafka partitions**
- Questions like: *"How should I partition my data?"*, *"Why is my database node a hot spot?"*, *"How do I add a node without downtime?"*, *"How does consistent hashing work?"*, *"How does Elasticsearch handle distributed queries?"*

---

Also called: *sharding* (MongoDB, Elasticsearch, SolrCloud), *regions* (HBase), *tablets* (Bigtable), *vnodes* (Cassandra, Riak), *vBuckets* (Couchbase).

Goal: spread data and load across multiple machines. Needed when dataset exceeds a single machine's capacity.

Each record belongs to exactly one partition. Usually combined with replication (each partition stored on several nodes).

---

## Partitioning and Replication

Each node may be the leader for some partitions and follower for others. Partitioning and replication are orthogonal concerns. This chapter ignores replication.

---

## Partition of Key-Value Data

Goal: distribute evenly. *Skew*: some partitions have more data/load. A disproportionately loaded partition is a *hot spot*.

### Partitioning by Key Range

Assign contiguous ranges of keys to partitions (like encyclopedia volumes: A-B, C-D...).

- Range queries are efficient
- **Risk of hot spots**: If access pattern follows time (e.g., sensor readings with timestamp keys), all writes go to "today's" partition

Mitigate: prefix key with something other than timestamp (e.g., sensor name + timestamp), so writes spread across partitions.

### Partitioning by Hash of Key

Hash the key, assign ranges of hashes to partitions. Distributes load evenly. Used by: Cassandra, MongoDB, Couchbase, Voldemort.

- Cassandra uses MD5 for hashing
- Loses range query efficiency (adjacent keys go to random partitions). MongoDB sends range queries to all partitions; Cassandra uses *compound primary keys* — only first column is hashed, remaining columns are sorted within a partition.

**Consistent hashing**: Hash ring approach. Assigns random hash ranges to nodes. Avoids redistribution of all keys when nodes are added/removed. Term is often misused.

### Hot Spots

Even with hash partitioning, a single extremely hot key (celebrity user with millions of followers) creates a hot spot. Application-level mitigation: add random 2-digit suffix to the key → 100 partitions for that key, spread writes. Reads must then query all 100 and merge. Track which keys need this treatment.

---

## Partitioning and Secondary Indexes

Secondary indexes complicate partitioning because a secondary index value doesn't map to a single partition.

### Document-Partitioned Secondary Indexes (Local Indexes)

Each partition maintains its own secondary indexes for documents within that partition only.

**Scatter/gather reads**: A query on the secondary index must be sent to all partitions and results merged. Expensive but simple. Used by: MongoDB, Riak, Cassandra, Elasticsearch, SolrCloud, VoltDB.

### Term-Partitioned Secondary Indexes (Global Indexes)

The secondary index itself is partitioned — all values for a particular term (regardless of which primary partition they're in) go to the same secondary index partition.

- **Read-efficient**: Can query a single partition for a term
- **Write-expensive**: Writing a document may update several index partitions (that hold each secondary value)
- Writes often asynchronous — index may be slightly stale (DynamoDB, Riak, Oracle)

---

## Rebalancing Partitions

Moving data from one node to another as cluster membership changes. Requirements:
- After rebalancing, load (data + reads + writes) should be fairly distributed
- DB should continue accepting reads and writes during rebalancing
- Move only the minimum data necessary

### Strategies

**Fixed number of partitions**: Create far more partitions than nodes upfront (e.g., 1000 partitions for 10 nodes = 100 partitions/node). When adding a node, steal some partitions from existing nodes. Number of partitions is fixed from the start; partition size grows with dataset. Used by: Riak, Elasticsearch, Couchbase, Voldemort.

**Dynamic partitioning**: Partitions split automatically when they exceed a configured size (HBase: 10GB default), and merge when too small. Number of partitions adapts to data volume. Initial single-partition is a bottleneck until first split; mitigation: *pre-splitting* (configure initial set of partitions).

**Partitioning proportional to nodes**: Fixed number of partitions *per node* (Cassandra, Ketama). When a node joins, it splits some existing partitions randomly and takes half of each. With many partitions, splits are approximately even.

### Automatic vs Manual Rebalancing

Fully automatic rebalancing is convenient but dangerous — rebalancing is expensive and if done during a node failure/high load, can cascade into an outage. Many systems require administrator approval for each rebalance. Couchbase, Riak, Voldemort generate a suggested plan; admin executes it.

---

## Request Routing

How does a client know which node to contact for a given key?

Three approaches:
1. Contact any node; it forwards request to correct node (*gossip protocol*)
2. Dedicated routing tier (partition-aware load balancer)
3. Client knows partition assignment directly

**ZooKeeper** (or etcd): Coordination service that tracks cluster metadata (partition-to-node mapping). Nodes register themselves; routing tier subscribes to ZooKeeper for updates. Used by: HBase, SolrCloud, Kafka. Cassandra and Riak use gossip protocol instead.

---

## Key Terms

| Term | Definition |
|---|---|
| Partition / shard | Subset of a dataset assigned to one node |
| Skew | Uneven distribution of data or load across partitions |
| Hot spot | Partition with disproportionately high load |
| Key range partitioning | Assign contiguous key ranges to partitions (efficient range queries) |
| Hash partitioning | Hash key to assign to partition (even distribution; loses range queries) |
| Consistent hashing | Hash ring approach to partitioning; minimizes redistribution on node changes |
| Scatter/gather | Query all partitions and merge results (for document-partitioned indexes) |
| Local index | Secondary index maintained per partition (document-partitioned) |
| Global index | Secondary index partitioned by term, independent of data partitioning |
| Rebalancing | Moving partitions between nodes as cluster membership changes |
| Dynamic partitioning | Partitions split/merge automatically based on size |
| ZooKeeper / etcd | Coordination service for cluster metadata and routing |
