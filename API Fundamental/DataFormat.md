# System Design: Data Formats

> **Topic:** Data Formats in System Design  
> **Source:** algomaster.io/learn/system-design/data-formats

---

## Why Data Formats Matter

Every time two systems communicate, they must agree on a **common language** for exchanging data. The format you choose impacts:

- **Performance** – serialization/deserialization speed
- **Bandwidth** – payload size over the network
- **API evolvability** – ability to add/remove fields over time
- **Debuggability** – how easy it is to inspect data in production
- **Interoperability** – how well different languages/platforms handle the format

---

## Text-Based vs Binary Formats

| Property | Text-Based (JSON, XML) | Binary (Protobuf, Avro, Thrift) |
|---|---|---|
| Human-readable | ✅ Yes | ❌ No |
| Payload size | Larger | Smaller (2–10× compressed) |
| Parse speed | Slower | Faster |
| Debugging ease | Easy | Harder (needs tooling) |
| Schema required | Optional | Usually required |

---

## JSON (JavaScript Object Notation)

```json
{
  "userId": 101,
  "name": "Alice",
  "active": true
}
```

**Pros:**
- Human-readable and widely supported
- Native to JavaScript; works seamlessly with REST APIs
- No schema required
- Easy to debug

**Cons:**
- Verbose — field names repeated in every record
- No native support for binary data (needs Base64 encoding)
- No strict types (e.g., integers vs floats)
- Slower to parse at high volume

**Best for:** Public REST APIs, config files, debugging, browser-to-server communication

---

## XML (Extensible Markup Language)

```xml
<user>
  <userId>101</userId>
  <name>Alice</name>
  <active>true</active>
</user>
```

**Pros:**
- Self-describing with rich metadata support
- Supports namespaces, attributes, and complex document structures
- Strong tooling (XPath, XSLT, XSD validation)

**Cons:**
- Very verbose — much larger payload than JSON
- Slow to parse
- Complex to work with in modern languages

**Best for:** Enterprise integrations (SOAP, WSDL), document-centric data, legacy systems

---

## Protocol Buffers (Protobuf)

Developed by **Google**. Uses a `.proto` schema file to define message structure.

```proto
message User {
  int32 user_id = 1;
  string name = 2;
  bool active = 3;
}
```

**Pros:**
- Very compact binary format (3–10× smaller than JSON)
- Extremely fast serialization/deserialization
- Strongly typed with backward/forward compatibility
- Auto-generates code for many languages

**Cons:**
- Not human-readable
- Requires schema (`.proto` file) and code generation step
- Harder to debug without tooling

**Best for:** High-performance internal microservice communication, gRPC APIs, mobile apps, streaming pipelines

---

## Apache Avro

Developed by the **Apache Hadoop** project. Schema is stored in **JSON format** and typically embedded with data or in a schema registry.

```json
{
  "type": "record",
  "name": "User",
  "fields": [
    {"name": "userId", "type": "int"},
    {"name": "name",   "type": "string"},
    {"name": "active", "type": "boolean"}
  ]
}
```

**Pros:**
- Compact binary encoding
- Schema is stored alongside data → excellent for schema evolution
- No field IDs needed (unlike Protobuf)
- First-class support in the Kafka/Hadoop ecosystem

**Cons:**
- Schema must be available at read time
- Less multi-language support than Protobuf
- Not human-readable

**Best for:** Big data pipelines (Kafka, Spark, Flink), data lakes, event streaming

---

## MessagePack

A binary serialization format that mirrors JSON structure but encodes it compactly.

**Pros:**
- Drop-in replacement for JSON — same data model
- Smaller and faster than JSON
- Wide language support

**Cons:**
- Not human-readable
- Less compact than Protobuf or Avro
- No built-in schema enforcement

**Best for:** Real-time applications, gaming, IoT, mobile APIs where JSON is too slow but full schema adoption is not needed

---

## Apache Thrift

Developed by **Facebook**. Similar to Protobuf — uses an IDL (Interface Definition Language) to define types and services.

```thrift
struct User {
  1: i32 userId,
  2: string name,
  3: bool active
}
```

**Pros:**
- Supports multiple serialization protocols (binary, compact, JSON)
- Built-in RPC framework
- Strong typing and code generation

**Cons:**
- Less popular than Protobuf today
- Tooling and documentation less mature

**Best for:** Internal RPC services, especially in environments already using the Facebook/Meta stack

---

## Comparison: Choosing the Right Format

| Format | Readability | Size | Speed | Schema | Best Use Case |
|---|---|---|---|---|---|
| **JSON** | ✅ High | Large | Moderate | Optional | Public APIs, config, debugging |
| **XML** | ✅ High | Very large | Slow | Optional | Enterprise/legacy, SOAP |
| **Protobuf** | ❌ Binary | Small | Very fast | Required | gRPC, microservices, mobile |
| **Avro** | ❌ Binary | Small | Fast | Required | Kafka, Hadoop, data pipelines |
| **MessagePack** | ❌ Binary | Medium | Fast | Optional | Real-time, IoT, gaming |
| **Thrift** | ❌ Binary | Small | Fast | Required | Internal RPC (Meta ecosystem) |

---

## Schema Evolution: The Hidden Complexity

As systems evolve, data schemas change. Schema evolution rules determine compatibility:

### Types of Compatibility

| Type | Description |
|---|---|
| **Backward compatible** | New code can read data written by old code (add optional fields) |
| **Forward compatible** | Old code can read data written by new code (ignore unknown fields) |
| **Full compatible** | Both directions work — safest for rolling deployments |

### Rules for Safe Evolution

- ✅ **Add** new optional fields
- ✅ **Remove** fields that were optional
- ❌ **Never rename** existing fields (Protobuf uses field numbers, not names)
- ❌ **Never change** the type of an existing field
- ❌ **Never remove** required fields

### Schema Registry

For formats like **Avro** in Kafka pipelines, a **Schema Registry** (e.g., Confluent Schema Registry) stores and manages schema versions centrally, ensuring producers and consumers always agree on the current schema.

---

## Practical Tips

1. **Default to JSON** for public-facing APIs — readability and tooling support outweigh performance costs at typical scale.

2. **Switch to Protobuf/Avro** when you need high throughput, low latency, or are handling millions of events per second.

3. **Use Avro with Kafka** — it integrates naturally with schema registries and handles schema evolution gracefully.

4. **Avoid XML** for new systems unless integrating with legacy enterprise services.

5. **Always version your schemas** — even for JSON APIs. Use `version` fields or URL versioning (`/v1/`, `/v2/`).

6. **Benchmark your format choice** — serialization costs can be significant at scale. Measure before optimizing.

7. **Consider the full lifecycle** — a format easy to write today might be hard to evolve or debug in production a year later.

---

## Quick Decision Guide

```
Is human readability important?
├── YES → JSON (or XML for legacy/enterprise)
└── NO
    ├── Using Kafka / big data pipelines? → Avro
    ├── Using gRPC or high-perf microservices? → Protobuf
    ├── Need JSON-like model, just faster? → MessagePack
    └── Already in Meta/Thrift ecosystem? → Thrift
```

---

*Notes compiled from: [algomaster.io – System Design: Data Formats](https://algomaster.io/learn/system-design/data-formats)*