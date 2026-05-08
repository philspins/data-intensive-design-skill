# Chapter 3 — Storage and Retrieval

## When to Use This Chapter

Consult this chapter when the task involves any of the following:

- Explaining or comparing **storage engine internals**: B-trees, LSM-trees, SSTables, hash indexes, Bloom filters
- Choosing between **write-optimized vs read-optimized** storage (LSM vs B-tree trade-offs)
- Discussing **indexing strategies**: primary, secondary, clustered, covering, multi-column, full-text/fuzzy indexes
- Designing or explaining **OLTP vs OLAP** systems, data warehouses, ETL pipelines, star schemas
- Understanding **column-oriented storage**, bitmap encoding, materialized views, or data cubes
- Discussing specific engines: **RocksDB, LevelDB, Cassandra, HBase, Lucene, Elasticsearch, Bitcask, Redshift**
- Questions like: *"Why are LSM-trees faster for writes?"*, *"How does Elasticsearch index data?"*, *"What is write amplification?"*, *"What's the difference between a data warehouse and a database?"*, *"How does column storage work?"*

---

Two families of storage engines: **log-structured** (LSM-trees) and **page-oriented** (B-trees). Plus a separate path for analytics: **column-oriented storage**.

---

## Log-Structured Storage

### Hash Indexes

Simplest index: in-memory hash map mapping each key → byte offset in append-only log file.

**Bitcask** (Riak's default engine) uses this. Requirement: all keys fit in RAM; values can exceed RAM (loaded from disk on demand).

**Compaction**: To reclaim disk space, break the log into *segments* of fixed size. Compact segments by discarding duplicate keys (keeping only the most recent). Merge multiple segments simultaneously in a background thread. Reads switch to the new merged segment; old segments are deleted.

Limitations of hash indexes:
- Hash table must fit in memory (on-disk hash maps perform poorly)
- Range queries are inefficient (can't scan keys in order)

---

### SSTables and LSM-Trees

**SSTable** (Sorted String Table): A segment file where key-value pairs are **sorted by key**. Each key appears only once per segment (compaction guarantees this).

Advantages over hash-indexed log segments:
1. **Merging is efficient** (use mergesort; keep most recent value for duplicates)
2. **Sparse in-memory index is sufficient** — only need the offset of one key per few KB; binary search + scan finds any key
3. **Block compression possible** — read requests scan a range anyway, so group records into compressed blocks

**Building an SSTable in memory**: Use a balanced BST (*memtable*): red-black tree or AVL tree. Writes go to the memtable. When it exceeds a threshold (few MB), flush to disk as an SSTable. Reads check memtable first, then most recent SSTable, then older SSTables.

**Crash safety**: Keep a separate unsorted write-ahead log (WAL) on disk. Discard WAL when memtable is flushed to SSTable.

**LSM-Tree** (Log-Structured Merge-Tree): The overall storage engine pattern — merge and compact sorted files. Used by: LevelDB, RocksDB, Cassandra, HBase, Lucene (for its term dictionary).

**Bloom filters**: Probabilistic data structure that quickly answers "does this key exist?" Avoids expensive disk lookups for missing keys.

**Compaction strategies**:
- *Size-tiered*: Newer, smaller SSTables merged into older, larger ones (HBase, Cassandra option)
- *Leveled*: Key range split into smaller SSTables; older data moved to separate levels (LevelDB, RocksDB, Cassandra option). More disk-efficient.

**LSM performance**: Even with terabytes of data on disk, writes are fast (sequential). The sorted structure means range queries are efficient.

---

### B-Trees

Most widely used index structure. The standard for relational databases (and many non-relational ones).

**Structure**: Keys sorted (like SSTables), but stored in fixed-size *pages* (blocks), typically 4KB. Pages form a tree: root → internal pages → leaf pages (values or references to row locations).

**Branching factor**: Number of child references per page. Typically several hundred. A B-tree with *n* keys has depth O(log n). 4-level tree with 4KB pages can store 256TB.

**Writes**: Overwrite the page on disk. If a page splits (overflow), update parent page too. Contrast with LSM-trees which only append.

**Write-ahead log (WAL / redo log)**: B-trees maintain a WAL to enable crash recovery. Before modifying pages, append the change to the WAL. After crash, replay WAL to restore B-tree to consistent state.

**Concurrency**: Multiple threads require *latches* (lightweight locks) on B-tree internals.

### B-Trees vs LSM-Trees

| | B-Trees | LSM-Trees |
|---|---|---|
| Read speed | Faster (single location per key) | Slower (check multiple SSTables) |
| Write speed | Slower (write-amplification from overwriting pages + WAL) | Faster (sequential appends; compaction in background) |
| Write amplification | Higher | Lower |
| Disk space | Higher (fragmentation from partial pages) | Lower (compression more effective) |
| Predictability | More predictable (no compaction interference) | Can have compaction spikes |
| Transactional semantics | Strong (each key in exactly one place; range locks attach to tree) | Harder (multiple copies of key may exist) |

Compaction can saturate disk I/O and starve reads/writes if not configured properly. Monitor that compaction keeps up with write rate.

---

### Other Index Structures

- **Primary key index**: Maps key → row (heap file or clustered index)
- **Secondary index**: Values are not unique. Implement by (a) maintaining a list of row IDs per value, or (b) appending row ID to make key unique
- **Clustered index**: Stores the actual row data within the index (e.g., InnoDB's primary key index)
- **Covering index**: Stores some columns within the index — "covers" certain queries
- **Concatenated (multi-column) index**: Combines multiple fields; efficient for range queries on the leading column(s) (e.g., geospatial R-trees)
- **Full-text / fuzzy indexes**: Allow searching for similar keys (synonyms, typos). Lucene uses a finite-state automaton (similar to a trie) over its term dictionary

---

### In-Memory Databases

Disk advantages: durability and low cost per GB. But RAM is fast, and datasets are increasingly small enough to fit.

**Durability approaches for in-memory DBs**:
- Periodic snapshots to disk
- Append-only WAL to disk
- Replication to other machines
- Special battery-backed RAM

On restart, reload state from disk or replica. Disk is just the durability log; reads are served entirely from RAM.

**Products**: VoltDB, MemSQL, Oracle TimesTen, Redis, Memcached (cache only — intentionally not durable), Couchbase.

**Performance advantage**: Not just avoiding disk I/O — in-memory DBs avoid the *encoding overhead* of serializing in-memory data structures to a disk-compatible format.

**Richer data models**: In-memory DBs can offer data structures hard to implement with disk-based indexes (e.g., Redis's sorted sets, priority queues).

---

## OLTP vs OLAP

| | OLTP | OLAP |
|---|---|---|
| Read pattern | Small number of records per query, fetched by key | Aggregate over millions of records |
| Write pattern | Random-access, low-latency writes | Bulk import (ETL) or event streams |
| Used by | End user/application | Internal analyst |
| Data represents | Latest state of data | History of events |
| Dataset size | GB to TB | TB to PB |

### Data Warehousing

A *data warehouse* is a separate read-only database for analysts, populated via ETL (Extract-Transform-Load) from OLTP systems. Keeps OLTP systems unaffected by heavy analytic queries.

Common engines: Amazon Redshift (hosted ParAccel), Hive, Spark SQL, Cloudera Impala, Presto, Drill.

**Star Schema**: Fact table at the center (individual events, can be 100s of columns) surrounded by dimension tables (who, what, where, when, how, why of each event). Queries typically join the fact table with several dimension tables.

*Snowflake schema*: Dimensions have sub-dimensions (more normalized). Star schema is more common in practice.

---

## Column-Oriented Storage

In OLTP, storage is row-oriented (all values of a row stored together). In OLAP, queries typically read only a few columns out of hundreds.

**Column-oriented storage**: Store all values from each column together. A query only needs to read the columns it accesses.

### Column Compression

Column values often have low cardinality (few distinct values) → highly compressible.

**Bitmap encoding**: For a column with n distinct values, store n bitmaps (one per value). Bit i is 1 if row i has that value. If n is small, the bitmaps are sparse → run-length encode them.

Bitmap indexes are efficient for vectorized (SIMD) processing — CPU can process 64 column values per clock cycle.

### Sort Order in Column Storage

Choose a sort key based on common query patterns. Sorted columns compress better. A second sort key can be chosen for additional compression.

Multiple sort orders can be stored (like having multiple secondary indexes) — Vertica does this. Each replica can store data sorted differently.

### Writing to Column Stores

In-place updates are hard (sorted structure). Use LSM-trees: writes go to an in-memory store, periodically merged into column files on disk. Queries must check both.

### Aggregation: Materialized Views

Precompute and cache aggregates (e.g., COUNT, SUM, AVG, MIN, MAX) using *materialized views* — actual copies of query results on disk, not virtual like standard SQL views.

**Data cube** (OLAP cube): Precomputed grid of aggregates across multiple dimensions. Very fast for supported queries, but inflexible for ad hoc queries.

---

## Key Terms

| Term | Definition |
|---|---|
| WAL (Write-Ahead Log) | Append-only log recording changes before they're applied to the main data structure |
| Memtable | In-memory sorted tree buffer for LSM-tree writes |
| SSTable | Sorted String Table — sorted, immutable file of key-value pairs |
| LSM-Tree | Log-Structured Merge-Tree — storage engine based on merging sorted files |
| Compaction | Process of merging SSTable segments and discarding obsolete entries |
| Bloom filter | Probabilistic structure for fast "key does not exist" checks |
| B-tree | Balanced tree index with fixed-size pages; standard for relational DBs |
| Branching factor | Number of child references per B-tree page |
| Write amplification | One logical write causes multiple physical writes |
| OLTP | Online Transaction Processing — low-latency reads/writes for applications |
| OLAP | Online Analytical Processing — aggregate queries over large datasets |
| ETL | Extract-Transform-Load — populating data warehouse from OLTP systems |
| Star schema | Data warehouse schema: central fact table surrounded by dimension tables |
| Column-oriented storage | Store column values together (vs row-oriented) for analytic efficiency |
| Materialized view | Precomputed query result stored on disk |
| Data cube | Precomputed aggregate grid across multiple dimensions |
