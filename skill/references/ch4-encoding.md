# Chapter 4 — Encoding and Evolution

## When to Use This Chapter

Consult this chapter when the task involves any of the following:

- Choosing or comparing **serialization formats**: JSON, XML, Protocol Buffers, Thrift, Avro, MessagePack
- Designing systems with **schema evolution** — adding/removing fields while maintaining backward and forward compatibility
- Discussing **API versioning**, rolling upgrades, or migrating data formats without downtime
- Evaluating **RPC frameworks** (gRPC, Thrift, Avro RPC) vs REST vs message queues for service communication
- Understanding **message brokers**, async messaging, or the actor model
- Questions like: *"How do I evolve a Protobuf schema without breaking clients?"*, *"What's the difference between Avro and Protobuf?"*, *"Should I use REST or gRPC?"*, *"What are the trade-offs of JSON vs binary encoding?"*, *"How does Avro schema resolution work?"*

---

Applications change over time. Schemas change. Old and new code must coexist.

- **Backward compatibility**: Newer code can read data written by older code.
- **Forward compatibility**: Older code can read data written by newer code.

---

## Formats for Encoding Data

### Language-Specific Formats

Java serialization, Python pickle, Ruby Marshal — tied to a language, often have security problems, poor versioning support. **Avoid for anything cross-system or long-lived.**

### JSON, XML, CSV

Human-readable. Widely supported. Problems:
- Ambiguous number encoding (no integer vs float distinction in JSON; no precision guarantees)
- No built-in binary string support in JSON/XML (workarounds: Base64)
- Schema optional (CSV has none, XML/JSON have schemas but rarely used)
- Forward/backward compatibility relies on careful convention

Good enough for many uses. Not ideal for high-performance or compact storage.

### Binary Encoding

More compact, faster. Two main approaches:

#### Schema-based (most compact and safe)

**Thrift** (Facebook): Two binary protocols — BinaryProtocol and CompactProtocol. Fields identified by *field tags* (numbers). Tags must not change; required/optional is a soft constraint.

**Protocol Buffers** (Google): Similar to Thrift CompactProtocol. Each field has a *field number* tag. Field names in schema only; binary contains only field numbers + values.

**Schema evolution rules (Protobuf / Thrift)**:
- *Forward compat*: Old code reads new data — ignore unknown field tags.
- *Backward compat*: New code reads old data — new required fields must have defaults; removed optional fields not present in old data.
- You can change a field's datatype *if* the encoding is compatible (e.g., int32 → int64 in Protobuf is OK because values are widened).

**Avro** (Hadoop): Different approach — no field tags. Schema stored separately. Binary encoding relies on exact field ordering matching the schema.
- Uses *writer's schema* (schema used when writing) and *reader's schema* (schema code expects).
- Avro resolves fields by *name matching*. Schema evolution: add/remove fields with defaults.
- Writer's schema stored in file header (for large files), schema registry (for databases), or negotiated in connection handshake (for network comms).
- Friendly to dynamically generated schemas (e.g., from database columns) since no field tag assignment needed.

---

### Comparison of Binary Formats

| Format | Field identification | Schema | Compactness | Evolvability |
|---|---|---|---|---|
| Thrift BinaryProtocol | Field tag (16-bit) | Required at encoding | Moderate | Good |
| Thrift CompactProtocol | Field tag (variable-int) | Required | High | Good |
| Protocol Buffers | Field tag (varint) | Required | High | Good |
| Avro | Field order / name | Required (writer + reader) | Highest | Good (name-based matching) |

---

## Modes of Dataflow

### Dataflow Through Databases

Writer encodes data; future reader (maybe different code version) decodes it. Data outlives the code. Must maintain backward *and* forward compatibility.

Subtle hazard: if new code adds a field, then old code reads + rewrites the record, it must preserve the unknown field even though it doesn't know what it is. Avro, Protobuf, Thrift handle this — unknown fields are preserved on roundtrip.

**Schema migrations**: In large databases, running `ALTER TABLE` can take hours (lock the table). Many databases let you declare the new schema and do the migration lazily on read (null defaults), or use tools like gh-ost.

### Dataflow Through Services: REST and RPC

**REST**: Not a protocol but a design philosophy built on HTTP. Uses URLs for resource identification, HTTP verbs for actions, JSON/XML for data. OpenAPI (Swagger) for documentation.

**SOAP**: XML-based protocol. Verbose, complex, unfashionable.

**RPC** (Remote Procedure Call): Makes a network call look like a local function call. Fundamentally flawed abstraction:
- Network requests are unpredictable (packets lost, slow, duplicated)
- Retries can cause duplicate writes unless idempotent
- Latency is variable and often high
- Remote objects can't be passed by reference

Modern RPC frameworks (gRPC, Thrift, Avro) are explicit about being network calls. gRPC uses Protocol Buffers + HTTP/2.

**RPC schema evolution**: Easier than DB migration because client and server can be updated independently. Need backward compat for requests and forward compat for responses.

### Dataflow Through Message Passing (Async)

Messages sent via a *message broker* (message queue / message-oriented middleware). Decouples sender from receiver.

Advantages over direct RPC:
- Buffer if recipient is overloaded
- Retry automatically if recipient crashes (durability)
- Sender doesn't need to know recipient's address
- One message to many recipients (fan-out / pub-sub)
- Decouples sender from receiver logically

**Brokers**: RabbitMQ, ActiveMQ, HornetQ, NATS, Apache Kafka.

**Actor model**: Unit of concurrency is an actor (has local mutable state, communicates via async messages). Frameworks: Akka, Orleans, Erlang OTP. Messages are encoded/decoded — schema evolution rules apply.

---

## Key Terms

| Term | Definition |
|---|---|
| Encoding / serialization | Translating in-memory objects to a byte sequence |
| Decoding / deserialization | Reverse: byte sequence → in-memory objects |
| Backward compatibility | New code reads old data |
| Forward compatibility | Old code reads new data |
| Field tag | Integer identifier for a field in Thrift/Protobuf (stable across schema changes) |
| Writer's schema | Schema used when data was written (Avro) |
| Reader's schema | Schema expected by the reading code (Avro) |
| REST | Architectural style for HTTP-based APIs |
| RPC | Remote Procedure Call — network call disguised as local function call |
| Message broker | Intermediary that stores and forwards messages |
| Actor model | Concurrency model using isolated actors communicating via async messages |
