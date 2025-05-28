---
GRC: "20"
Title: Knowledge Graph
Authors: Yaniv Tal, Byron Guina
Created: 2024-11-21
Stage: Draft
Version: 0.1.0
Discussions-To: https://forum.thegraph.com/t/grc-20-knowledge-graph/6161
---

# GRC-20: Knowledge Graph

## 1. Introduction

### 1.1 Abstract

This GRC-20 specification defines a standard for storing and representing knowledge on The Graph. It enables any application to create, share, and consume interoperable knowledge across applications and domains. 

Data is defined as raw bits that are stored and transmitted. Information is data that is decoded and interpreted into a useful structure. Knowledge is created when information is linked and labeled to attain a higher level of understanding.

This document specifies the serialization format for knowledge. Using this standard, any application can access the entire knowledge graph and produce knowledge that can become part of The Graph.

### 1.2 Motivation & Scope

#### Motivation

Knowledge graphs are the most flexible way of representing information. Knowledge is highly interconnected across applications and domains, so flexibility is important to ensure that different groups and individuals can independently extend and consume knowledge in an interoperable format. Most apps define their own schemas. Those schemas work for a specific application but getting apps to coordinate on shared schemas is difficult. Once data is produced by an app, custom code has to be written to integrate with each app. Migrations, versioning and interpreting data from different apps becomes even more difficult as schemas evolve. These are some of the reasons that instant composability and interoperability across apps is still a mostly unsolved problem. The original architects of the web understood this and tried to build the semantic web which was called Web 3.0 twenty years ago. Blockchains and decentralized networks give us the tools we need to build open and verifiable information systems. Solving the remaining composability challenges will enable an interoperable web3.

Early efforts like the Semantic Web (Web 3.0) leveraged RDF tiples for sharing knowledge across organizational boundaries. RDF is the W3C standard for triples data. A new standard is necessary for web3 for several reasons: IDs in RDF are URIs which typically point to servers that are controlled by corporations or individuals using https. This breaks the web3 requirement of not having to depend on specific server operators. Additionally, RDF doesn't support the property graph model, which is needed to describe facts about relationships between entities. Some of RDF's concepts are unnecessarily complex which has hindered its adoption beyond academic and niche enterprise settings. For these reasons, a new standard is being proposed that is web3 native, benefits from the latest advancements in graph databases and can be easily picked up by anyone who wants to build the decentralized web.

#### Scope

This specification covers:

* **Core data model:** Entities, Relations, Types, Properties
* **Native value types:** Text, Number, Checkbox, URL, Time, Point
* **Identifier conventions:** UUID guidelines
* **Serialization & encoding:** Protobuf schemas and JSON examples
* **Operations & versioning:** Ops, Edits, Imports
* **Spaces:** Personal vs. Public spaces, access controls

This specification does *not* prescribe:

* Detailed smart contract and governance implementations beyond anchoring patterns
* Content-type extensions outside the minimum needed for implicit properties (e.g., Image, Block)
* Indexer specifications beyond some basic context


## 2. Terminology & Concepts

## 3. Architecture Overview

This section describes the high-level components and interactions that make up a GRC-20 knowledge graph deployment, from authoring and storage through anchoring, indexing, and consumption.

---

### 3.1 Logical Components

* **Spaces**
  Logical domains—either **public**, **personal** or **private** that encapsulate a knowledge graph along with its governance and access controls. Public spaces are backed by onchain smart contracts with public governance, personal spaces are publicly visible and anchored onchain but are controlled by an individual or entity, and private spaces are stored locally on-device or can be encrypted and shared within a group.
* **Content-Addressed Storage**
  Offchain networks (IPFS, Arweave, etc.) that hold the full protobuf-encoded graph payloads. Each payload is retrievable by its cryptographic hash.
* **Onchain Anchoring**
  Smart contracts record minimal anchors (content hashes, version IDs) on a blockchain. Anchors guarantee immutability and global ordering without storing large amounts of data onchain.
* **Indexers**
  Services that listen for onchain anchor events, fetch the corresponding offchain payloads, deserialize them, and maintain an up-to-date, queryable graph index.
* **Clients**
  End-user or application libraries that create “edits,” submit them for publishing, and query the index (or peers) for entities, relations, and types.

---

### 3.2 Storage & Anchoring Layers

1. **Edit Serialization**
   Clients package changes into a protobuf-encoded `Edit` message (see §10).
2. **Offchain Storage**
   The serialized `Edit` is stored in a content-addressed network; its CID (hash) becomes the canonical reference.
3. **Onchain Anchor**
   A lightweight transaction in the Space contract records the CID (and any minimal metadata), ensuring a tamper-proof history and global ordering.

By separating payload storage from onchain anchoring, we minimize gas costs while preserving verifiability.

---

### 3.3 P2P & Local-First Modes

* **Peer-to-Peer Distribution**
  Spaces can propagate edits directly between clients using gossip, pub/sub, or CRDTs using the same logical model before or instead of onchain anchoring.
* **Local-First Workflows**
  Private spaces may operate entirely offline, storing edits for later anchoring or sharing. Conflict resolution (e.g., merging branches) follows the same edit-ordering rules once synchronized.

---

### 3.4 End-to-End Data Flow

1. **Create/Edit:** User or app constructs an `Edit` with one or more Ops.
2. **Serialize:** The `Edit` is marshaled into a compact protobuf message.
3. **Publish:** The binary is uploaded to a content-addressed network; its content hash (CID) is returned.
4. **Anchor:** The Space contract records the CID in a transaction, establishing a global sequencing for that space.
5. **Index:** Indexers detect new anchors, retrieve and deserialize the payload, then update their local graph store.
6. **Consume:** Clients and downstream apps query the index (or fetch directly from peers) to read entities, relations, types, and properties.

---

### 3.5 Cross-Space Interoperability

* Any entity in one space can reference entities in another by including their UUIDs along with optional space IDs in relations.
* This design forms a **global unified graph**, where independent communities can interlink information without centralized coordination.

## 4. Core Data Model

This section defines the building blocks of the graph—**Entities**, **Relations**, **Types**, and **Properties**—along with their minimal on-wire representation.

---

### 4.1 Entities

* **Definition:**
  An *Entity* is a node in the graph representing any thing (person, place, thing, concept, etc.).
* **Structure:**

  * **ID**: a globally unique UUID4 (byte-encoded on-wire).
  * **Values**: zero or more key/value pairs (`Value` messages) attaching data to the entity.
  * **Relations**: handled separately (see §4.2).

```proto
message Entity {
  bytes id        = 1;  // UUID4
  repeated Value values = 2;
}
```

#### 4.1.1 Value

A `Value` bundles a property reference with its literal content and optional per-value metadata.

```proto
message Options {
  oneof value {
    TextOptions   text   = 1;
    NumberOptions number = 2;
    TimeOptions   time   = 3;
  }
}


message Value {
  bytes property = 1;
  string value = 2;
  optional Options options = 3;
}

message TextOptions {
  optional bytes language = 1;
}

message NumberOptions {
  optional string format = 1;
  optional bytes unit = 2;
}

message TimeOptions {
  optional string format = 1;
  optional bytes timezone = 2;
  optional bool has_date = 3;
  optional bool has_time = 4;
}
```

* **TextOptions** (e.g. language entity ID)
* **NumberOptions** (format string, unit ID)
* **TimeOptions** (format, timezone, date/time flags)

---

### 4.2 Relations

Relations form the **edges** of the graph. They are compose of two components:

1. **Lightweight Relation:** minimal, index-friendly fields
2. **Relation Entity:** optional node carrying additional properties or types about the relationship itself

#### 4.2.1 Lightweight Representation

| Field          | Type        | Description                                                     |
| -------------- | ----------- | --------------------------------------------------------------- |
| `id`           | bytes       | Relation’s own UUID4                                            |
| `type`         | bytes       | Relation-type UUID                                              |
| `from_entity`  | bytes       | Source entity UUID                                              |
| `from_space`   | bytes ⋅opt  | (optional) source Space UUID                                    |
| `from_version` | bytes ⋅opt  | (optional) source Edit/Version UUID                             |
| `to_entity`    | bytes       | Target entity UUID                                              |
| `to_space`     | bytes ⋅opt  | (optional) target Space UUID                                    |
| `to_version`   | bytes ⋅opt  | (optional) target Edit/Version UUID                             |
| `entity`       | bytes       | UUID of the “relation entity” for attaching extra properties    |
| `position`     | string ⋅opt | (optional) fractional position key for ordering                 |
| `verified`     | bool ⋅opt   | (optional) whether the source space trusts the target reference |

> **Immutability:** `id`, `type`, `from_entity`, and `to_entity` are fixed once created.
> **Mutable:** `to_space`, `verified`, `index`, and the *relation entity*’s own values.

#### 4.2.2 Relation Entity

To annotate a relation (e.g. “joined on” date), the client creates the relation entity (UUID in `entity` field) and then issues `Value` edits against that entity.

#### 4.2.3 Protobuf Snippet

```proto
message Relation {
  bytes id            = 1;
  bytes type          = 2;
  bytes from_entity   = 3;
  optional bytes from_space   = 4;
  optional bytes from_version = 5;
  bytes to_entity     = 6;
  optional bytes to_space     = 7;
  optional bytes to_version   = 8;
  bytes entity        = 9;
  optional string position   = 10;
  optional bool   verified    = 11;
}

```

---

### 4.3 Types & Subtyping

**Types** describe the structure of information. In an interconnected graph that can be used by any application, types should be helpful but not get in the way. Applications should be able to evolve independently without breaking others. The primary goal is composition and reuse. There should be a draw towards reuse and sharing, while empowering people to customize whatever they want. Every node implicitly has the `Entity` type.

* **Types Relation:** each entity has zero or more outgoing `Types` relations.
* **Subtyping:**

  * **Broader types** → parent type(s)
  * **Subtypes** inherit properties and relation-type hints from their broader types.

| Type Property     | Relation‹Type› ID                      |
| ----------------- | -------------------------------------- |
| **Properties**    | `01412f83-8189-4ab1-8365-65c7fd358cc1` |
| **Broader types** | `14df474e-4f48-4117-9619-d99e4b3afd3e` |

---

### 4.4 Properties

A **Property** is itself an entity that defines an attribute name, its value type, and optional metadata.

* **Default Type:** if omitted, `Text`
* **Attributes of a Property entity:**

| Name             | Type           | Description                                      |
| ---------------- | -------------- | ------------------------------------------------ |
| **Value type**   | Relation‹Type› | Entity ID indicating the native value type       |
| **Unit**         | Relation‹Unit› | Optional unit (e.g., currency)                   |
| **Format**       | Text           | Optional formatting string (ICU-style)           |
| **Has language** | Checkbox       | If true, Text values must include a language tag |

| Property Type Name | Type | UUID                                   |
| ------------------ | ---- | -------------------------------------- |
| Property           | Type | `808a04ce-b21c-4d88-8ad1-2e240613e5ca` |

```proto
message Property {
  bytes id   = 1; // property UUID
  bytes type = 2; // should always be the Property-type ID
  // values for Name, Value type, Unit, Format, Has language are stored as Values on this entity
}
```

Once a property is published with a specific type, that type is immutable. If the user wants to change the type of a property, a new property should be created instead. By modeling Properties as first-class entities, the graph can evolve new attributes and users can share or extend schemas without coordination.

---

This core model—Entities carrying Values, Relations linking nodes, Types governing schema, and reusable Property entities—forms the foundation for the GRC-20 knowledge graph.

## 5. Native Value Types

The GRC-20 standard defines six built-in “native” value types. Each is encoded as a UTF-8 string literal. If a value fails to conform to its type’s syntax, it is dropped (treated as unset); `null` is not permitted.

| Name     | UUID                                   | Description                                           |
| -------- | -------------------------------------- | ----------------------------------------------------- |
| Text     | `9edb6fcc-e454-4aa5-8611-39d7f024c010` | A sequence of Unicode characters                      |
| Number   | `9b597aae-c31c-46c8-8565-a370da0c2a65` | A decimal numeric value                               |
| Checkbox | `7aa4792e-eacd-4186-8272-fa7fc18298ac` | A boolean flag (`true` or `false`)                    |
| URL      | `283127c9-6142-4684-92ed-90b0ebc7f29a` | A URI with protocol identifier (e.g. `https`, `ipfs`) |
| Time     | `167664f6-68f8-40e1-976b-20bd16ed8d47` | An ISO-8601 timestamp or interval                     |
| Point    | `df250d17-e364-413d-9779-2ddaae841e34` | One or more comma-separated coordinates               |

---

### 5.1 Text

A `Text` value is an arbitrary-length UTF-8 string. It may include newlines or Markdown.

* **Validation:** any valid UTF-8 sequence
* **Options:**

  * `language` (entity ID referencing a Language); defaults to US English if omitted and the text is linguistic
* **Example:**

  ```json
  "Radiohead"
  ```

---

### 5.2 Number

A `Number` value is a UTF-8 string representing a decimal number.

* **Syntax:**

  * Optional leading `-` for negatives
  * Digits plus at most one `.`
  * No commas or currency symbols
* **Options:**

  * `format` (ICU decimal format string, e.g. `¤#,##0.00`)
  * `unit` (entity ID for a currency or measurement unit)
* **Examples:**

  ```json
  "1234.56"
  "-789.01"
  ```

---

### 5.3 Checkbox

A `Checkbox` is a boolean encoded as:

* `"1"` = `true`
* `"0"` = `false`

Clients may render it as a switch or toggle.

---

### 5.4 URL

A `URL` is any valid URI beginning with a supported protocol and `://`.

* **Supported protocols:** `graph`, `ipfs`, `ar`, `https`
* **Examples:**

  ```json
  "ipfs://QmVGF2e9WqF8g39W81eCQxecz7Bdi5o2DFSJ5BQxwfveYB"
  "https://geobrowser.io"
  ```

---

### 5.5 Time

A `Time` value is an ISO-8601 string representing a timestamp or duration.

* **Recommendation:** include a timezone; if omitted, interpret in the consumer’s local time
* **Options:**

  * `format` A format string using the [Unicode Technical Standard #35](https://www.unicode.org/reports/tr35/tr35-dates.html#Date_Field_Symbol_Table).

* **Example value:**

  ```json
  "2024-11-03T11:15:45.000Z"
  ```
* **Example formats:**

Examples:

* `h:mmaaa, EEEE, MMMM d, yyyy` - 4:45pm, Thursday, July 4, 2024
* `h:mmaaa, MMMM d, yyyy` - 4:45pm, July 4, 2024
* `h:mmaaa, MMM d, yyyy` - 4:45pm, Jul 4, 2024
* `h:mmaaa, MMM d, yy` - 4:45pm, Jul 4, 24
* `EEEE, MMMM d, yyyy` - Thursday, July 4, 2024
* `MMMM d, yyyy` - July 4, 2024
* `MMM d, yyyy` - Jul 4, 2024
* `MMM d, yy` - Jul 4, 24
* `MM/dd/yyyy` - 07/04/2024
* `MM/dd/yy` - 07/04/24
* `h:mmaaa` - 4:45pm

---

### 5.6 Point

A `Point` is a coordinate or vector encoded as comma-separated decimal numbers.

* **Syntax:** one or more numeric values (e.g. `x,y` or `x,y,z`); optional space after commas
* **Geolocation convention:** latitude, longitude (positive north/east; negative south/west)
* **Example:**

  ```json
  "12.554564, 11.323474"
  ```

## 6. Identifiers

Every core object in the GRC-20 graph (Entities, Relations, Types, Properties, Edits, etc.) is referenced by a **globally unique identifier**. To ensure decentralization, interoperability, and minimal collision risk, we adopt **UUID version 4** for all IDs.

### 6.1 UUID Generation

* **Standard:** RFC 4122, version 4 (random).
* **Entropy Source:** Cryptographically secure random number generator.
* **Collision Risk:** Negligible at 2⁻¹²²; no central coordination or registration required.

### 6.2 On-Wire (Binary) Encoding

* **Format:** 16-byte array, network byte order (big-endian).
* **Protobuf Definition:**

  ```proto
  // Any `bytes id = 1;` field carries a raw 16-byte UUID4.
  message SomeEntity {
    bytes id = 1;  // 16 bytes
    // …
  }
  ```

### 6.3 JSON & String Representation

* **Canonical Form:** 36-character, lowercase hexadecimal string with dashes.
* **Example:**

  * Standard UUID4: `2a5fe010-078b-4ad6-8a61-7272f33729f4`

### 6.4 ID Conversion

For systems that already emit long or proprietary unique IDs, map them deterministically into GRC-20 UUID4s:

1. Compute the MD5 hash of the original ID string (UTF-8).
2. Use the 16-byte MD5 digest to seed a UUID4 generator (i.e., set version bits to 4, variant bits per RFC 4122).
3. Encode the resulting UUID as 16-byte wire format or hex string.

This process yields a stable, collision-resistant UUID for any existing globally unique identifier.

---

By enforcing a single, decentralized UUID4 scheme—binary on-wire and dash-stripped in text—GRC-20 ensures all nodes and edges can reference each other unambiguously across spaces, chains, and storage layers.

## 7. Spaces & Governance

A **Space** encapsulates a self-contained knowledge graph along with its governance and access-control rules. Spaces let communities or individuals manage who can read, write, and anchor edits under a consistent policy.

---

### 7.1 Space Categories

* **Personal Spaces**

  * **Onchain only:** backed by a deployed smart contract.
  * **Custom access controls:** personal spaces are publicly visible but centrally controlled by an individual or entity.

* **Public Spaces**

  * **Onchain only:** backed by a deployed smart contract.
  * **Public governance:** proposals, voting, and execution enforced by onchain rules.

* **Private Spaces**

  * **Local-first:** operate entirely on a user’s device; edits queue offline.
  * **Shared peer-to-peer:** propagate changes within a trusted group.

---

### 7.2 Lifecycle & Conversion

Any entity can evolve into a Space when it requires its own governance:

1. **Promote an Entity:** choose an existing entity UUID.
2. **Deploy Space Contract:** publish a smart contract (L1 or L2) encoding roles, voting, or ACLs.
3. **Bind Entity to Contract:** map the entity’s UUID to the contract address, enabling onchain anchoring.
4. **Migrate Edits:** replay or import prior edits into the new Space (see §11 Importing & Migrations).

---

### 7.3 Space Type

We model Space itself as a special **Type** so it can be referenced, inherited, and extended like any other schema element.

| Name  | Type | UUID                                   |
| ----- | ---- | -------------------------------------- |
| Space | Type | `362c1dbd-dc64-44bb-a3c4-652f38a642d7` |

---

### 7.4 Space Hierarchy

Spaces can form hierarchies via relations. Their UUIDs are defined in the UUID glossary (Appendix §13).

| Property           | Relation‹Space› | Description                                  |
| ------------------ | --------------- | -------------------------------------------- |
| **Broader spaces** | parent space(s) | The immediate parent(s) in a space hierarchy |
| **Subspaces**      | child space(s)  | More specialized or granular subdomains      |

This convention—Broader/Sub—mirrors natural language (“this is a subspace of that”) and lets clients traverse up or down a space tree.

---

### 7.5 Governance & Access Controls

* **Public Spaces (On-Chain):**

  * Smart contracts define who can propose, endorse, or apply edits.

* **Personal Spaces:**

  * Client libraries enforce local ACLs (e.g., device keys, group permissions).

* **Private Spaces (Off-Chain):**

  * Peer-to-peer protocols carry edits; conflicts resolved via the same ordered Ops (§10).

Indexers and clients **must** respect each Space’s governance policy when accepting, anchoring, and indexing edits.

## 8. Serialization & Encoding

GRC-20 uses Protocol Buffers (proto3) for all core serialization, ensuring compact, language-neutral messages. For human-facing APIs or debugging, clients may also emit and consume equivalent JSON representations, following the conventions below.

---

### 8.1 Protocol Buffer Definitions

> **All** implementations **must** use these proto3 schemas verbatim to guarantee cross-language interoperability. Full definitions, including option messages (`TextOptions`, `NumberOptions`, `TimeOptions`, etc.), appear in Appendix §13.

```proto
syntax = "proto3";
package grc20;

// --- Core Objects ---

message Entity {
  bytes id        = 1;  // 16-byte UUID4
  repeated Value values = 2;
}

message Value {
  bytes property = 1;    // Property UUID
  string value   = 2;    // UTF-8 literal
  oneof options {
    TextOptions   text_options   = 3;
    NumberOptions number_options = 4;
    TimeOptions   time_options   = 5;
  }
}

message Relation {
  bytes id            = 1;
  bytes type          = 2;
  bytes from_entity   = 3;
  optional bytes from_space   = 4;
  optional bytes from_version = 5;
  bytes to_entity     = 6;
  optional bytes to_space     = 7;
  optional bytes to_version   = 8;
  bytes entity        = 9;
  optional string position   = 10;
  optional bool   verified    = 11;
}


message Property {
  bytes id   = 1;   // property UUID
  bytes type = 2;   // must equal the Property-type UUID
}

// --- Operations & Versioning ---

enum OpType {
  UPDATE_ENTITY       = 1;
  CREATE_RELATION     = 2;
  UPDATE_RELATION     = 3;
  DELETE_RELATION     = 4;
  UNSET_ENTITY_VALUES = 5;
  MOVE_ENTITY         = 6;
  MERGE_ENTITIES      = 7;
  BRANCH_ENTITY       = 8;
}

message Op {
  OpType    type     = 1;
  Entity    entity   = 2;
  Relation  relation = 3;
  Property  property = 4;
}

message Edit {
  bytes id = 1;
  string name = 2;
  repeated Op ops = 3;
  repeated bytes authors = 4;
  optional bytes language = 5;
}

message ImportEdit {
  bytes id = 1;
  string name = 2;
  repeated Op ops = 3;
  repeated bytes authors = 4;
  bytes created_by = 5;
  string created_at = 6;
  bytes block_hash = 7;
  string block_number = 8;
  bytes transaction_hash = 9;
}

message Import {
  // these strings are IPFS cids representing the import edit message
  repeated string edits = 1;
}

message File {
  string version = 1;

  oneof payload {
    Edit add_edit = 2;
    Import import_space = 3;
    bytes archive_space = 4;
  }
}

---

### 8.2 JSON Interchange Conventions

When exposing HTTP/REST or CLI interfaces, clients may map Protobuf messages to JSON with these rules:

* **`bytes` fields** → lowercase, dash-stripped hex strings
* **`enum` fields** → numeric values
* **`oneof`** → include only the present option
* **`repeated`** → JSON arrays

#### 8.2.1 Example: Entity

```json
{
  "id": "2a5fe010-078b-4ad6-8a61-7272f33729f4",
  "values": [
    {
      "property": "6ab0887f-b6c1-4628-9eee-2234a247c1d2",
      "value": "San Francisco"
    }
  ]
}
```

#### 8.2.2 Example: Relation

```json
{
  "id":           "29785ffb-5ce6-4a9b-b0dc-b138328ba334",
  "type":         "0509279a-713d-4667-a412-24fe3757f554",
  "from_entity":  "2a5fe010-078b-4ad6-8a61-7272f33729f4",
  "to_entity":    "386d9dde-4c10-4099-af42-0e946bf2f199",
  "entity":       "7f9b1c0a-9fbc-4d63-a2d3-da5c1e1a2b3c",
  "position":     "a",
  "verified":     false
}

```

#### 8.2.3 Example: Edit

```json
{
  "version": "1.0.0",
  "type": 1,
  "id": "386d9dde-4c10-4099-af42-0e946bf2f199",
  "name": "Add a new city",
  "ops": [
    {
      "type": 2,
      "entity": {
        "id": "29785ffb-5ce6-4a9b-b0dc-b138328ba334",
        "values": [
          {
            "property": "7f9b1c0a-9fbc-4d63-a2d3-da5c1e1a2b3c",
            "value": "San Francisco"
          }
        ]
      }
    }
  ],
  "authors": ["0x1A39E2Fe299Ef8f855ce43abF7AC85D6e69E05F5"],
  "language": "en"
}
```

#### 8.2.4 Example: Import

```json
{
  "version": "1.0.0",
  "type": 4,
  "previousNetwork": "Ethereum Mainnet",
  "previousContractAddress": "0x1A39E2Fe299Ef8f855ce43abF7AC85D6e69E05F5",
  "edits": [
    "ipfs://QmVGF2e9WqF8g39W81eCQxecz7Bdi5o2DFSJ5BQxwfveYB"
  ]
}
```

---

By combining a compact binary format for production with clear JSON fallbacks, GRC-20 ensures both performance and ease of integration across diverse environments.

## 9. System & Implicit Properties

Entities in GRC-20 carry two classes of built-in properties:

1. **Implicit Entity Properties** – always present on every entity, exposed to clients as editable fields.
2. **System Properties** – computed by indexers, read-only to clients.

---

### 9.1 Implicit Entity Properties

Every entity automatically has these core fields. Clients **must** render them alongside any user-defined properties.

| Name            | Type            | Description                                                            | UUID                                   |
| --------------- | --------------- | ---------------------------------------------------------------------- | -------------------------------------- |
| **Name**        | Text            | Human-readable label for the entity                                    | `a126ca53-0c8e-48d5-b888-82c734c38935` |
| **Types**       | Relation‹Type›  | Zero or more classification types assigned to the entity               | `8f151ba4-de20-4e3c-9cb4-99ddf96f48f1` |
| **Description** | Text            | Brief summary shown in previews or tooltips                            | `9b1f76ff-9711-404c-861e-59dc3fa7d037` |
| **Cover**       | Image           | Wide banner image conveying the entity                                 | `34f53507-2e6b-42c5-a844-43981a77cfa2` |
| **Blocks**      | Relation‹Block› | Ordered content blocks (rich text, media, etc.) attached to the entity | `beaba5cb-a677-41a8-b353-77030613fc70` |

> **Cover Image Guidelines:**
>
> * Recommended dimensions: **2384 × 640 px** (other aspect ratios allowed)
> * Clients should fetch image metadata (width/height) via the Image content spec before rendering.

---

### 9.2 System Properties

These properties are **not** serialized in edits or exports. Indexers compute and populate them; clients **must** display them as read-only.

| Name           | Type              | Description                                                                                  |
| -------------- | ----------------- | -------------------------------------------------------------------------------------------- |
| **Created at** | Time              | Timestamp when the entity was first indexed                                                  |
| **Updated at** | Time              | Timestamp of the most recent edit applied                                                    |
| **Created by** | Text              | Blockchain address (or key ID) of the author who anchored the creating edit                  |
| **Versions**   | Relation‹Version› | References to prior `Edit` versions for this entity (e.g., for history or rollback purposes) |

> **Note:** Clients should fetch `Versions` relations to present an edit history or allow “undo” operations, but must not modify these relations directly.

## 10. Edits, Imports & Operations

This section specifies how atomic changes (“Ops”) are composed into versioned edits, how full-space migrations are handled, and how all payloads are enveloped for IPFS storage.

---

### 10.1 Operations (`Op`)

An **Op** is the smallest unit of change. It uses a `oneof` payload so each operation is unambiguous:

```proto
message Op {
  oneof payload {
    Entity                update_entity         = 1;  // Create or upsert an entity’s values
    bytes                 delete_entity         = 2;  // Remove an entity and its values
    Relation              create_relation       = 3;  // Add a new graph edge
    RelationUpdate        update_relation       = 4;  // Modify mutable edge fields
    bytes                 delete_relation       = 5;  // Remove an edge link
    UnsetEntityValues     unset_entity_values   = 6;  // Remove specific properties from an entity
    UnsetRelationFields   unset_relation_fields = 7;  // Clear specific fields on a relation
  }
}
```

* **`update_entity`**: carries a full `Entity` message with its `id` and new `values`.
* **`delete_entity`**: the entity’s `id` to remove.
* **`create_relation`** / **`update_relation`** / **`delete_relation`**: manage the lifecycle of relations, with `Relation` and `RelationUpdate` messages.
* **`unset_entity_values`**: lists property-UUIDs to clear on an entity without deleting it.
* **`unset_relation_fields`**: lists relation fields (`from_space`, `position`, etc.) to clear.

---

### 10.2 Edit

An **Edit** packages one or more Ops into a single change set:

```proto
message Edit {
  bytes         id        = 1;  // UUID4 of this edit
  string        name      = 2;  // Short “commit” summary
  repeated Op   ops       = 3;  // Ordered operations
  repeated bytes authors  = 4;  // UUIDs of contributors
  optional bytes language = 5;  // Default language tag for Text values
}
```

* **`id`**: uniquely identifies the edit for de-duplication and history.
* **`name`**: human-friendly description (e.g. “Add city entity”).
* **`ops`**: must be applied in sequence by indexers.
* **`authors`**: supports multi-authoring and attribution.
* **`language`**: if set, applies to all Text values lacking individual language tags.

Indexers deserialize each `Edit`, apply its `ops` in order, and record the edit’s `id` to support potential rollbacks or forks.

---

### 10.3 ImportEdit

For onchain migrations or space bootstrapping, an **ImportEdit** extends `Edit` with blockchain metadata:

```proto
message ImportEdit {
  bytes         id               = 1; // Edit UUID4
  string        name             = 2;
  repeated Op   ops              = 3;
  repeated bytes authors         = 4;
  bytes         created_by       = 5; // Address or key ID that anchored this import
  string        created_at       = 6; // ISO-8601 timestamp
  bytes         block_hash       = 7; // Onchain block hash
  string        block_number     = 8; // Block number as decimal string
  bytes         transaction_hash = 9; // Transaction hash anchoring the import
}
```

Each `ImportEdit` encapsulates a full replay of prior edits along with provenance data, allowing downstream indexers to reconstruct not only state but the exact onchain context.

---

### 10.4 Import Manifest

A lightweight **Import** manifest lists the IPFS CIDs of one or more `ImportEdit` messages:

```proto
message Import {
  repeated string edits = 1;  // IPFS CIDs, in chronological order
}
```

Clients upload `ImportEdit` payloads to IPFS, collect their CIDs, then publish this manifest. Indexers fetch each CID, parse the corresponding `ImportEdit`, and apply its `ops` in sequence.

---

### 10.5 File Payload Envelope

All edits, imports, and archival actions are wrapped in an **`File`** envelope for storage and retrieval:

```proto
message File {
  string version = 1;  // Spec version, e.g. "1.0.0"
  oneof payload {
    Edit   add_edit      = 2;  // New content update
    Import import_space  = 3;  // Migration manifest
    bytes  archive_space = 4;  // Raw CID for archival marker
  }
}
```

* **`version`** ensures clients can detect deprecated formats.
* **`add_edit`** carries an `Edit` to append to the space.
* **`import_space`** carries an `Import` manifest for migrations.
* **`archive_space`** is a raw CID indicating the archival of a space (no further edits).

Clients and indexers must parse the `payload` one-of to dispatch accordingly:

* On **`add_edit`**, apply the contained `Edit`.
* On **`import_space`**, fetch and replay each `ImportEdit` in the manifest.
* On **`archive_space`**, mark the space as read-only.

---

With this model—fine-grained Ops, concise Edits, metadata-rich ImportEdits, and a unified IPFS envelope—GRC-20 provides a clear, verifiable, and extensible mechanism for producing and evolving decentralized knowledge graphs.


## 11. Imports & Migrations

When a Space must be bootstrapped from—or migrated to—a new environment (e.g. L2 upgrade, new chain), we use **ImportEdit** messages and a lightweight **Import** manifest, all wrapped in the standard IPFS envelope. This section shows the updated definitions, workflow, and examples.

---

### 11.1 `ImportEdit`

An `ImportEdit` is a special edit that replays one or more prior Ops **and** carries onchain provenance metadata:

```proto
message ImportEdit {
  // Core edit fields
  bytes         id               = 1;  // UUID4 of this import-edit
  string        name             = 2;  // e.g. “Import from Mainnet v1”
  repeated Op   ops              = 3;  // All ops to replay
  repeated bytes authors         = 4;  // UUIDs or addresses of original authors

  // Onchain context
  bytes         created_by       = 5;  // Address or key ID that anchored this import
  string        created_at       = 6;  // ISO-8601 timestamp of the anchor tx
  bytes         block_hash       = 7;  // Hash of the block containing the anchor
  string        block_number     = 8;  // Decimal block number
  bytes         transaction_hash = 9;  // Tx hash anchoring the import
}
```

* **`ops`** should replay the exact sequence of original edits.
* **`created_by`**, **`created_at`**, **`block_hash`**, **`block_number`**, and **`transaction_hash`** allow indexers to verify and audit the import against the target chain.

---

### 11.2 `Import` Manifest

A minimal manifest that lists the IPFS CIDs of one or more `ImportEdit` payloads:

```proto
message Import {
  // IPFS CIDs of ImportEdit messages, in chronological order
  repeated string edits = 1;
}
```

* Clients assemble this after uploading each `ImportEdit`; they need only publish the manifest.

---

### 11.3 Workflow

1. **Deploy New Space Contract**
   Release your Space contract on the target chain or L2.

2. **Generate `ImportEdit` Messages**
   For each batch of historic Ops (e.g. per original block or logical grouping), create an `ImportEdit` that replays those Ops and fills in the onchain metadata from the source network.

3. **Publish to IPFS**

   * Serialize each `ImportEdit` as raw protobuf.
   * Upload to IPFS; note each resulting CID (e.g. `QmAbC123…`).

4. **Construct `Import` Manifest**

   ```proto
   Import {
     edits: [
       "QmAbC123…",
       "QmXyZ789…",
       …
     ]
   }
   ```

5. **Envelope in `File`**

   ```proto
   File {
     version: "1.0.0";
     payload {
       import_space: <Import manifest>;
     }
   }
   ```

6. **Anchor on-Chain**
   Submit a transaction to the new Space contract that records the CID of the `File` envelope (with `import_space` payload).

7. **Index & Replay**

   * Indexers detect the onchain anchor, fetch the `File`, and parse out `import_space`.
   * For each CID in `Import.edits`, fetch the corresponding `ImportEdit`, deserialize it, and apply its `ops` in order.
   * Record imported edit IDs to support deduplication and provenance queries.

8. **Verify State**
   Confirm the reconstructed graph matches the original Space’s index on the source chain.

---

### 11.4 JSON Examples

#### 11.4.1 Import Manifest

```json
{
  "edits": [
    "ipfs://QmAbC123…",
    "ipfs://QmXyZ789…"
  ]
}
```

#### 11.4.2 `File` Envelope

```json
{
  "version": "1.0.0",
  "payload": {
    "import_space": {
      "edits": [
        "ipfs://QmAbC123…",
        "ipfs://QmXyZ789…"
      ]
    }
  }
}
```

Clients and indexers following this pattern gain a transparent, auditable migration path—preserving both content and onchain context—while minimizing onchain footprint.


## 12. Entity Content Types

Certain content-rich entities (e.g. videos, pdfs) rely on specialized types. A full Content Spec will define a comprehensive set; here we introduce the two types referenced by implicit entity properties (§9).

---

### 12.1 Image

An **Image** entity encapsulates an asset plus metadata for provenance, attribution, and rendering. Clients store or retrieve the binary via a URL/URI property (e.g. IPFS or HTTP), then use these properties for layout or display.

| Property | Value Type | Description            | UUID                 |
| -------- | ---------- | ---------------------- | -------------------- |
| Width    | Number     | Image width in pixels  | (Native Number type) |
| Height   | Number     | Image height in pixels | (Native Number type) |

| Name  | Type | UUID                                   |
| ----- | ---- | -------------------------------------- |
| Image | Type | `772172cb-7abf-422e-8b8b-10ef17738402` |

Clients **must** include Width and Height when known; this enables correct aspect-ratio rendering before full asset download.

---

### 12.2 Block

A **Block** represents a discrete piece of rich content—text, media embed, code snippet, etc.—that together form the body of a page or document. Blocks are ordered via relations to the parent entity.

| Name  | Type          | UUID                                   |
| ----- | ------------- | -------------------------------------- |
| Block | Relation Type | `beaba5cb-a677-41a8-b353-77030613fc70` |

Each Block relation points to a separate content-typed entity (defined in the Content Spec). Clients render Blocks in ascending fractional index order (§4.2).

---

## 13. Appendix

### 13.1 Complete Protobuf Schemas

```proto
syntax = "proto3";
package grc20;

// --- Value Option Messages ---

message Options {
  oneof value {
    TextOptions   text   = 1;
    NumberOptions number = 2;
    TimeOptions   time   = 3;
  }
}

message TextOptions {
  optional bytes language = 1;
}

message NumberOptions {
  optional string format = 1;
  optional bytes unit = 2;
}

message TimeOptions {
  optional string format = 1;
  optional bytes timezone = 2;
  optional bool has_date = 3;
  optional bool has_time = 4;
}

// --- Core Graph Objects ---

message Entity {
  bytes id        = 1;   // 16-byte UUID4
  repeated Value values = 2;
}

message Value {
  bytes property = 1;
  string value = 2;
  optional Options options = 3;
}

message Relation {
  bytes id             = 1;  // relation UUID4
  bytes type           = 2;  // relation-type UUID4
  bytes from_entity    = 3;  // source entity UUID4
  optional bytes from_space   = 4;  // (opt) source Space UUID4
  optional bytes from_version = 5;  // (opt) source Edit/Version UUID4
  bytes to_entity      = 6;  // target entity UUID4
  optional bytes to_space     = 7;  // (opt) target Space UUID4
  optional bytes to_version   = 8;  // (opt) target Edit/Version UUID4
  bytes entity         = 9;  // UUID4 of the rich-relation entity
  optional string position    = 10; // (opt) fractional ordering key
  optional bool   verified     = 11; // (opt) trust flag
}

message Property {
  bytes id   = 1;   // property UUID4
  bytes type = 2;   // must equal Property-Type UUID4
  string name = 3;  // human-friendly property name
}

// --- Operations & Versioning ---

enum OpType {
  UPDATE_ENTITY       = 1;
  CREATE_RELATION     = 2;
  UPDATE_RELATION     = 3;
  DELETE_RELATION     = 4;
  UNSET_ENTITY_VALUES = 5;
  MOVE_ENTITY         = 6;
  MERGE_ENTITIES      = 7;
  BRANCH_ENTITY       = 8;
}

message Op {
  OpType   type     = 1;
  Entity   entity   = 2;
  Relation relation = 3;
  Property property = 4;
}

enum ActionType {
  ADD_EDIT        = 1;
  ADD_SUBSPACE    = 2;
  REMOVE_SUBSPACE = 3;
  IMPORT_SPACE    = 4;
  ARCHIVE_SPACE   = 5;
}

message Edit {
  string       version   = 1;  // SemVer, e.g. "1.0.0"
  ActionType   type      = 2;
  string       id        = 3;  // Edit UUID4
  string       name      = 4;  // commit-like message
  repeated Op  ops       = 5;
  repeated string authors = 6; // author IDs (e.g. wallet addresses)
  string       language  = 7;  // default Text language tag
}

message Import {
  string        version                  = 1;  // SemVer
  ActionType    type                     = 2;  // IMPORT_SPACE
  string        previousNetwork          = 3;  // e.g. "Ethereum Mainnet"
  string        previousContractAddress  = 4;  // origin Space contract
  repeated string edits                  = 5;  // ordered Edit URIs (CIDs)
}
```

---

### 13.2 UUID Constants

| Concept                     | UUID                                 |
| --------------------------- | ------------------------------------ |
| **Type (schema)**           | e7d737c5-3676-4c60-9fa1-6aa64a8c90ad |
| **Relation Type**           | c167ef23-fb2a-4044-9ed9-45123ce7d2a9 |
| **Property Type**           | 808a04ce-b21c-4d88-8ad1-2e240613e5ca |
| **Space Type**              | 362c1dbd-dc64-44bb-a3c4-652f38a642d7 |
| **Image Type**              | f3f790c4-c74e-4d23-a0a9-1e8ef84e30d9 |
| **Block Relation Type**     | beaba5cb-a677-41a8-b353-77030613fc70 |
| **Property ▶ Value Type**   | ee26ef23-f7f1-4eb6-b742-3b0fa38c1fd8 |
| **Property ▶ Unit**         | 386d9dde-4c10-4099-af42-0e946bf2f199 |
| **Property ▶ Format**       | c5d181a2-6593-4c2c-b0da-8396bab2d8fb |
| **Property ▶ Has language** | (Checkbox on Property)               |
| **Relation ▶ Value Types**  | 3b2dca52-f1bf-45c0-ab1c-b4f883cf51d1 |
| **Relation ▶ One Max**      | dfeb3c65-28f5-4962-9d66-6df94e2f9bb6 |
| **Relation ▶ Cascades**     | 69a2489b-1e2e-42d9-93f3-0779e44ebca6 |
| **Native Text**             | 9edb6fcc-e454-4aa5-8611-39d7f024c010 |
| **Native Number**           | 9b597aae-c31c-46c8-8565-a370da0c2a65 |
| **Native Checkbox**         | 7aa4792e-eacd-4186-8272-fa7fc18298ac |
| **Native URL**              | 283127c9-6142-4684-92ed-90b0ebc7f29a |
| **Native Time**             | 167664f6-68f8-40e1-976b-20bd16ed8d47 |
| **Native Point**            | df250d17-e364-413d-9779-2ddaae841e34 |
| **Implicit ▶ Name**         | a126ca53-0c8e-48d5-b888-82c734c38935 |
| **Implicit ▶ Types**        | 8f151ba4-de20-4e3c-9cb4-99ddf96f48f1 |
| **Implicit ▶ Description**  | 9b1f76ff-9711-404c-861e-59dc3fa7d037 |
| **Implicit ▶ Cover**        | 34f53507-2e6b-42c5-a844-43981a77cfa2 |
| **Implicit ▶ Blocks**       | beaba5cb-a677-41a8-b353-77030613fc70 |

---

With these full schemas and UUID tables, implementers have a single source of truth for all message formats, option types, and constant identifiers in the GRC-20 knowledge-graph standard.

## Copyright waiver

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
