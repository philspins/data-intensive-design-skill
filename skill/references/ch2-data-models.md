# Chapter 2 — Data Models and Query Languages

## When to Use This Chapter

Consult this chapter when the task involves any of the following:

- Choosing between **relational, document, or graph databases** for a new project
- Discussing **SQL vs NoSQL**, schema flexibility, or schema-on-read vs schema-on-write trade-offs
- Designing a **data model** for an application — one-to-many, many-to-many, hierarchical, or graph-shaped relationships
- Questions about **query languages**: SQL, MongoDB aggregation pipeline, Cypher, SPARQL, Datalog, MapReduce
- Explaining **impedance mismatch**, ORM trade-offs, or JSON vs relational storage
- Questions like: *"Should I use a document database or relational database for this?"*, *"When should I use a graph database?"*, *"What's the difference between schema-on-read and schema-on-write?"*, *"How does Cypher work?"*

---

Each layer in a software system hides the complexity below it via a clean data model. The choice of data model affects the code you write and the problems you can solve.

---

## Relational Model vs Document Model

**Relational model** (SQL): Data is organized in *relations* (tables) of *tuples* (rows). Proposed by Edgar Codd (1970). Goal was to hide storage implementation details behind a clean query interface.

**Impedance mismatch**: If your application uses objects/structs but the DB uses tables, there's an awkward translation layer (ORM helps but doesn't fully eliminate it).

**Document model** (JSON/BSON): Represents data as self-contained documents. Reduces impedance mismatch for hierarchical data. Better *locality* — all related data is in one document, one query retrieves it. Schema flexibility.

### When to use each:
- **Document model**: Data has a document-like, hierarchical, one-to-many tree structure. No need for many-to-many joins.
- **Relational model**: Many-to-one and many-to-many relationships are common. Joins are efficient.
- **Hybrid**: PostgreSQL, MySQL support JSON columns. MongoDB added multi-document transactions.

### Schema-on-write vs Schema-on-read
- **Schema-on-write** (relational): Schema enforced at write time (like static type checking). Migration required to change structure.
- **Schema-on-read** (document): Schema is implicit, interpreted at read time (like dynamic type checking). More flexible for heterogeneous data or externally-determined data structures.

---

## Historical Context

- **IMS** (IBM, 1970s): Hierarchical model. Good for one-to-many, bad for many-to-many. No joins.
- **CODASYL / Network model**: Each record can have multiple parents. Access via *access paths* (navigating pointer chains). Inflexible — changing query required changing application code.
- **Relational model**: Replaced both. Query optimizer decides access path automatically. Much easier to add new features.
- **NoSQL** (2010s): Driven by need for greater scalability, open source preference, specialized query ops, more dynamic schemas.

---

## Graph-Like Data Models

Best when many-to-many relationships are ubiquitous (social networks, road maps, web links, org charts).

**Vertices** (nodes/entities) + **Edges** (relationships/arcs).

Well-known algorithms work on graphs: shortest path, PageRank.

### Property Graph Model
*(Neo4j, Titan, InfiniteGraph)*

Each **vertex** has: unique ID, outgoing edges, incoming edges, key-value properties.  
Each **edge** has: unique ID, tail vertex, head vertex, label (relationship type), key-value properties.

Any vertex can connect to any other vertex — no schema restriction. Highly flexible and evolvable.

**Cypher**: Declarative query language for property graphs (created by Neo4j).
```cypher
MATCH (person) -[:BORN_IN]-> () -[:WITHIN*0..]-> (us:Location {name:'United States'})
RETURN person.name
```

### Triple-Store Model
*(Datomic, AllegroGraph)*

All data stored as *(subject, predicate, object)* triples. E.g., *(Jim, likes, bananas)*.

**SPARQL**: Query language for triple-stores using the RDF data model.

**Datalog**: Foundation for SPARQL and Cypher. Expresses data as predicates: `predicate(subject, object)`. Rules can be defined and composed, enabling complex recursive queries.

### Graph Queries in SQL
Possible using recursive CTEs (`WITH RECURSIVE`), but awkward. Cypher is far more natural for variable-length traversals.

---

## Query Languages

### Declarative vs Imperative
- **Imperative**: Tell computer *how* to do it, step by step.
- **Declarative**: Specify *what* you want; the engine decides how. SQL is declarative.

Declarative advantages:
- Hides implementation details (DB can optimize freely)
- Parallelizable (no specified execution order)
- Concise

Analogy: CSS (declarative) vs JavaScript DOM manipulation (imperative). CSS is clearer and more maintainable.

### MapReduce
Programming model for processing large datasets across many machines. Used in MongoDB, Hadoop.

The `map` and `reduce` functions must be **pure** (no side effects, no additional queries) — enables the DB to run them anywhere, in any order, and rerun on failure.

Downside: requires two carefully coordinated functions. Less opportunity for query optimization.

MongoDB added the **aggregation pipeline** as a declarative alternative to MapReduce:
```js
db.observations.aggregate([
  { $match: { family: "Sharks" } },
  { $group: { _id: { year: { $year: "$observationTimestamp" }, month: { $month: "$observationTimestamp" } },
              totalAnimals: { $sum: "$numAnimals" } } }
]);
```

---

## Key Terms

| Term | Definition |
|---|---|
| Impedance mismatch | Translation overhead between object model and relational model |
| Schema-on-read | Schema enforced at read time (document DBs) |
| Schema-on-write | Schema enforced at write time (relational DBs) |
| Property graph | Graph model where nodes and edges have key-value properties |
| Triple-store | Graph model storing (subject, predicate, object) triples |
| Cypher | Declarative graph query language (Neo4j) |
| SPARQL | Query language for RDF triple-stores |
| Datalog | Logic-based query language; foundation of SPARQL/Cypher |
| Shredding | Relational technique for splitting document-like structure into multiple tables |
