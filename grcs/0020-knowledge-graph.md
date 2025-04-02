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

## Abstract

This GRC introduces a standard for storing and representing knowledge on The Graph. It can be used by any application that wants to create and consume interoperable information across applications.

Data is defined as raw bits that are stored and transmitted. Information is data that is decoded and interpreted into a useful structure. Knowledge is created when information is linked and labeled to attain a higher level of understanding. This document outlines the valid serialization format for the knowledge data that is anchored onchain, shared peer-to-peer or stored locally. Using this standard, any application can access the entire knowledge graph and produce knowledge that can become part of The Graph.

## Motivation

Knowledge graphs are the most flexible way of representing information. Knowledge is highly interconnected across applications and domains, so flexibility is important to ensure that different groups and individuals can independently extend and consume knowledge in an interoperable format. Most apps define their own schemas. Those schemas work for a specific application but getting apps to coordinate on shared schemas is difficult. Once data is produced by an app, custom code has to be written to integrate with each app. Migrations, versioning and interpreting data from different apps becomes even more difficult as schemas evolve. These are some of the reasons that instant composability and interoperability across apps is still a mostly unsolved problem. The original architects of the web understood this and tried to build the semantic web which was called Web 3.0 twenty years ago. Blockchains and decentralized networks give us the tools we need to build open and verifiable information systems. Solving the remaining composability challenges will enable an interoperable web3.

Computer scientists widely understand triples to be the best unit for sharing knowledge across organizational boundaries. RDF is the W3C standard for triples data. A new standard is necessary for web3 for several reasons: IDs in RDF are URIs which typically point to servers that are controlled by corporations or individuals using https. This breaks the web3 requirement of not having to depend on specific server operators. Additionally, RDF doesn't support the property graph model, which is needed to describe facts about relationships between entities. Some of RDF's concepts are cumbersome and complex which has hindered its adoption beyond academic and niche enterprise settings. For these reasons, a new standard is being proposed that is web3 native, benefits from the latest advancements in graph databases and can be easily picked up by anyone who wants to build the decentralized web.

## Specification

Knowledge graphs are comprised of entities that represent people, places, things, concepts, or anything else. These entities have relations that link them into a graph. Knowledge is published onto The Graph using onchain transactions. Data can be stored offchain on a content addressed storage network and then anchored onchain. Each space for a community or individual has its own knowledge graph and its own governance or access controls. Any entity can reference any other entity across the entire graph, creating a global unified graph. Knowledge graph data doesn't have to be onchain, it can also be transmitted peer-to-peer between users and stored locally.

### 1. Encoding

Data is encoded using Protobufs. Protocol Buffers are a language-neutral, platform-neutral extensible mechanism by Google for serializing structured data. It's chosen for speed, compactness and broad language support.

### 2. IDs

All IDs must be globally unique and 22 characters using Base58 encoding. They should be created using the UUID4 standard and stripping out the dashes (bringing the length from 36 bytes to 32 bytes) and then Base58 encoding. If an entity is coming from a system that already has globally unique IDs of arbitrary length, they can be deterministically converted into valid globally unique 22 character IDs by taking an md5 hash of the string, seeding that into a UUID4 generator, stripping the dashes and Base58 encoding. IDs must be randomly generated and not be altered to contain any identifiable strings in order to prevent collisions.

Example ID: `Gw9uTVTnJdhtczyuzBkL3X`

### 3. Spaces

A space is a logical grouping that represents a group or individual, a knowledge graph and a governance process or access control system for updating information or making changes to the space. A space can be a personal space or a public space. Personal spaces can be private and run local first on a user's device, be shared with a group of people or be published onto a public blockchain for global consistency. Public spaces must be onchain with a public governance system.

### 4. Triples

The structure of knowledge on The Graph is built on simple primitives that compose to create more complex structures. Triples are the atomic unit. Triples are combined into entities. Entities are linked together to form a graph.

Triples represent a single atomic fact about an entity. They consist of an (Entity, Attribute, Value) tuple. There is also an implied space ID for every triple since different spaces can have different points of view on the same (Entity, Attribute) pair. For example, a space for Americans can assert that the Name of an entity is "Fries" and a British space might call the same entity "Chips" - but they would both reference the same entity ID.

If there are multiple triples for a single (Space, Entity, Attribute) combination, subsequent values get treated as an upsert, replacing the previous value for that triple. This applies to the native types and doesn't include relations which support many to many relationships.

#### 4.1 Entity

The entity field of a triple is a valid ID that represents the entity. It is used to associate any additional triples to the same entity. The first time an entity is created, the client is responsible for generating a globally unique ID. Enforcement of global ID uniqueness is expected of each space's governance.

#### 4.2 Attribute

The attribute field must be a valid ID that references the attribute entity that this triple is for. For example, the attribute might be for Name, Description, Birthday or any other property of the entity. The attribute may or may not be added as an attribute on one of the entity's types.

#### 4.3 Value

The value is an object that represents the value for the triple. This set of fields contains additional type and formatting information for the value.

| Name            | Value | Description                                               |
| --------------- |-------| --------------------------------------------------------- |
| Type            | 1     | An enum representing 1 of 6 native value types            |
| Value           | 2     | The encoding for the value depends on the value type      |
| Options         | 3     | Value type specific options                               |

The corresponding protobuf message is:

```proto
message Value {
  ValueType type = 1;
  string value = 2;
  Options options = 3;
}
```

Options can be included with a value and is specific to the value type. Additional options can be added in future spec versions.

```proto
message Options {
  string format = 1;
  string unit = 2;
  string language = 3;
}
```

```proto
message Triple {
  string entity = 1;
  string attribute = 2;
  Value value = 3;
}
```

#### Example Triple in JSON

```jsonc
{
  "entity": "Gw9uTVTnJdhtczyuzBkL3X",
  "attribute": "7UiGr3qnjZfRuKs3F3CX61",
  "value": {
    "type": 1, // Text
    "value": "San Francisco"
  }
}
```

### 5. Native value types

There are 6 built in native value types defined as an enum. A princple used for naming the native types is to prioritize accessibility and familiarity for non-developers. There will be many different types of applications producing and consuming knowledge and the more consistent, familiar and accessible producing and consuming raw knowledge is, the more people will be able to participate in the knowledge economy.

| Name            | Value | Description                                               |
| --------------- |-------| --------------------------------------------------------- |
| Text            | 1     | A string of characters                                    |
| Number          | 2     | A decimal value                                           |
| Checkbox        | 3     | Can either be `true` or `false`                           |
| URL             | 4     | A URI to a resource with a protocol identifier            |
| Time            | 5     | A date and time or interval                               |
| Point           | 6     | For geo locations or other coordinates                    |

The equivalent protobuf enum is:
```proto
enum ValueType {
  TEXT = 1;
  NUMBER = 2;
  CHECKBOX = 3;
  URL = 4;
  TIME = 5;
  POINT = 6;
}
```

For each native value type, if a value is provided that is invalid, the triple should be dropped and treated as unset. Null cannot be provided as a value. Each native value type is UTF-8 encoded as a string.

#### 5.1 Text

An arbitrary length string of characters. Text may include newline characters or markdown. Clients may choose which attributes to support markdown for. Example: "Radiohead"

Options: `language`

A language option can be provided to specify a language for the text. The language must be an entity ID for a Language entity. If no language is specified and the text is linguistic, it is assumed to be in US English.

#### 5.2 Number

Numbers are encoded as strings. One decimal point may be used. An optional negative sign (-) can be included at the beginning for negative values. No other symbols such as commas may be used. Additional metadata for units and formatting can be included as options.

Examples:
* "1234.56"
* "-789.01"

Options: `format`, `unit`

The format option uses the ICU decimal format. For example:
`¤#,##0.00` would render a value like `$1,234.56`.

The unit option can be included to define a specific currency or other unit to use. The value must be the entity ID of a Currency or Unit entity.

Additional attributes can be added to Number attributes to specify hints for database optimizations. This can allow application developers to add additional constraints that can be useful for choosing more efficient storage types.

#### 5.3 Checkbox

A checkbox can either be `true` or `false`. It's a boolean field named for normal people. The string encoding of the checkbox field must be "1" for `true` and "0" for `false`. Checkbox values can be rendered with alternate UI components such as switches and toggles.

#### 5.4 URL

A URL value type is technically a URI that starts with the protocol identifier followed by a :// and the resource. The initial supported protocols are: graph, ipfs, ar, https.

Examples:
* "ipfs://QmVGF2e9WqF8g39W81eCQxecz7Bdi5o2DFSJ5BQxwfveYB"
* "ht<span>tps://</span>geobrowser.io"

#### 5.5 Time

Time is represented as an ISO-8601 format string. A time value can be used to describe both date and time as well as durations. Clients should include timezones when creating times. In rare exceptions, if the timezone is omitted, the time is interpreted as being in the consumer's local time. Example: `2024-11-03T11:15:45.000Z`

Options: `format`

A format string using the [Unicode Technical Standard #35](https://www.unicode.org/reports/tr35/tr35-dates.html#Date_Field_Symbol_Table).

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

#### 5.6 Point

A point is a location specified with x and y coordinates string encoded. Points can be used for geo locations or other cartesian coordinate systems. Clients may choose how many dimensions to support. Example: `"12.554564, 11.323474"`

The value must have valid decimal numbers separated by a comma. A space after the comma is optional. For geo locations, the latitude and longitude must be provided with latitude first. Positive/negative signs indicate north/south for latitude and east/west for longitude respectively.

### 6. Types

Types describe the structure of information. In an interconnected graph that can be used by any application, types should be helpful but not get in the way. Applications should be able to evolve independently without breaking others. The primary goal is composition and reuse. There should be a draw towards reuse and sharing, while empowering people to customize whatever they want.

Every entity has an implicit Types relation and can have zero, one or many types.

| Name      | Type | ID                     |
| --------- |------|----------------------- |
| Type      | Type | Jfmby78N4BCseZinBmdVov |

Types may have zero, one or many attributes and relation types defined on them.

| Name           | Type                      | ID                     |
| -------------- |---------------------------|----------------------- |
| Attributes     | Relation \<Attribute>     | 9zBADaYzyfzyFJn4GU1cC  |
| Relation Types | Relation \<Relation Type> | TarUQath6Kyh3jfnRv3hy7 |

### 7. Attributes

Entities have attributes which are themselves defined as entities. An attribute describes a property of an entity.

| Name            | Type | ID                     |
| --------------- |------|----------------------- |
| Attribute       | Type | GscJ2GELQjmLoaVrYyR3xm |

The Attribute type has the following attributes:

| Name           | Type   | Description                                                          |
| -------------- |--------| -------------------------------------------------------------------- |
| Value Type     | Text   | The type ID a triple for this attribute should have. There are entities for the native types |

Attributes are used for native types like Text, Number and Time. The Value Types should reference the entity IDs created for the native types.

| Name        | ID                     |
| ------------| ---------------------- |
| Value Type  | WQfdWjboZWFuTseDhG5Cw1 |

The native type IDs are:

| Name         | ID                     |
| -------------| ---------------------- |
| Text         | LckSTmjBrYAJaFcDs89am5 |
| Number       | LBdMpTNyycNffsF51t2eSp |
| Checkbox     | G9NpD4c7GB7nH5YU9Tesgf |
| URL          | 5xroh3gbWYbWY4oR3nFXzy |
| Time         | 3mswMrL91GuYTfBq29EuNE |
| Point        | UZBZNbA7Uhx1f8ebLi1Qj5 |

### 8. Relations

Relations describe the edges of a graph. Relations are themselves entities that include details about the relationship. For example a Company can have Team Members. Each Team Member relation can have an attribute describing when the person joined the team. This is a model that is commonly called a property graph.

Relations are ordered. They can be reordered independently without conflicts using Fractional Indexing with an arbitrary level of precision between the indices of the items the user wants to move the item between. A fractional index must include randomness to allow items to be inserted between items with minimal chance of collision. This way moving the order of an item doesn't require changing the index of any of the other items.

Relations must have Type: Relation

| Name         | Type      | ID                     |
| ------------ |-----------|----------------------- |
| Relation     | Type      | QtC4Ay8HNLwSd1kSARgcDE |

A relation has the following attributes:

| Name         | Type   | Description                                                                    |
| ------------ |--------| ------------------------------------------------------------------------------ |
| From entity  | Text   | The entity ID this relation is from                                            |
| To entity    | Text   | The entity ID this relation is pointing to                                     |
| Index        | Text   | An alphanumeric fractional index describing the relative position of this item |

| Name            | Type      | ID                     |
| --------------- |-----------|----------------------- |
| From entity     | Attribute | RERshk4JoYoMC17r1qAo9J |
| To entity       | Attribute | Qx8dASiTNsxxP3rJbd4Lzd |
| Index           | Attribute | WNopXUYxsSsE51gkJGWghe |

From entity, To entity and Index are required.

#### 8.1 Relation Types

A Relation Type is a definition of a relation similar to an attribute but specifically for one to one, one to many, and many to many relations. A Relation Type can have a name like `Team member`, `City` or a more graph style name like `Born in`. The relevent Relation Types are added as types along with the `Relation` type to a relation. Adding Relation Types to Relations is optional but strongly encouraged. An individual relation cannot have more than one Relation Type.

| Name          | Type      | ID                     |
| ------------- |-----------|----------------------- |
| Relation Type | Type      | 3WxYoAVreE4qFhkDUs5J3q |

A Relation Type is also of type `Type` and has the following attributes:

| Name                     | Type              | Description                                                                    |
| ------------------------ |-------------------| ------------------------------------------------------------------------------ |
| Relation value types     | Relation \<Type>  | A list of types that the relation To entity may have. Used as a hint           |
| One max                  | Checkbox          | Whether to limit the number of relations from a given entity to no more than 1 |

| Name                     | Type          | ID                     |
| ------------------------ |---------------|----------------------- |
| Relation value types     | Relation Type | SeDR2b1bdnb6tj2qmdcFdg |
| One max                  | Checkbox      | K1M9AEyda5oNB2WS38bDDW |

The term properties is used to describe the union of an entity's attributes and relation types.

### 9. Implicit entity properties

All entities implicitly have the Entity type which contains the following properties:

| Name            | Type               | Description                                               |
| --------------- |--------------------| --------------------------------------------------------- |
| Name            | Text               | The name that shows up wherever this entity is referenced |
| Types           | Relation \<Type>   | The one or many types that this entity has                |
| Description     | Text               | A short description that is often shown in previews       |
| Cover           | Image              | A banner style image that is wide and conveys the entity  |
| Blocks          | Relation \<Block>  | The blocks of rich content associated with this entity    |

The ideal cover image dimensions is 2384x640px though other dimensions can be used.

| Name        | ID                     |
| ------------| ---------------------- |
| Name        | LuBWqZAu6pz54eiJS5mLv8 |
| Types       | Jfmby78N4BCseZinBmdVov |
| Description | LA1DqP5v6QAdsgLPXGF3YA |
| Cover       | 7YHk6qYkNDaAtNb8GwmysF |
| Blocks      | QYbjCM6NT9xmh2hFGsqpQX |


### 10. System properties

System derived properties are not serialized but they are part of the standard for data that is computed by indexers and made available on entities. Clients should show these fields as non-editable.

| Name           | Type                | Description                                                                             |
| -------------- |---------------------| --------------------------------------------------------------------------------------- |
| Created at     | Time                | The time this entity was first seen by the indexer                                      |
| Updated at     | Time                | The most recent time this entity was updated by the indexer                             |
| Created by     | Text                | The blockchain address of the account that signed the transaction to create this entity |
| Versions       | Relation \<Version> | A reference to previous versions of this entity                                         |


### 11. Space type

A space is also a type but public and personal spaces are special because they also include the deployment of a smart contract for governance and access controls. Any entity can also be a space. Once enough interest and specialization forms around an entity, it can be converted into its own space with its own governance.

| Name      | Type | ID                     |
| --------- |------|----------------------- |
| Space     | Type | 7gzF671tq5JTZ13naG4tnr |

A space has the following properties:

| Name            | Type               | Description                                                        |
| --------------- |--------------------| ------------------------------------------------------------------ |
| Broader spaces  | Relation \<Space>  | The spaces that this is a subspace of                              |
| Subspaces       | Relation \<Space>  | Spaces that are more granular for drilling down                    |

The Broader [thing] and Sub[thing] is a convention for relations to denote drilling down or up when something is hierarchical, chosen for natural language. The Broader spaces and Subspaces relations are system properties that can be queried as standard properties but are derived from the space contracts.

### 12. Entity content types

A separate Content spec will be provided with a standard set of content types. Two types are specified here since they are referenced in the _Implicit entity properties_ section.

#### 12.1 Image

Images are defined as a type with properties that can be used for provenance, attribution and other metadata. This way images can be reused in many places with proper attribution.

| Name           | Type                          | Description                                      |
| -------------- |-------------------------------| ------------------------------------------------ |
| Width          | Number                        | The width in pixels                              |
| Height         | Number                        | The height in pixels                             |

| Name       | Type | ID                     |
| ---------- |------|----------------------- |
| Image      | Type | Q1LaZhnzj8AtCzx8T1HRMf |

#### 12.2 Block

Every entity can have blocks comprising the content for a page. Apps can choose to support different block types. The standard set of block types will be published in the Content spec.

| Name       | Type          | ID                     |
| ---------- |---------------|----------------------- |
| Block      | Relation Type | QYbjCM6NT9xmh2hFGsqpQX |

### 13. Ops

An op represents a single atomic operation that produces or modifies knowledge. Ops are provided as an ordered list and must be processed in sequence.

```proto
message Op {
  OpType type = 1;
  Triple triple = 2;
}
```

| Name                   | Value |
| ---------------------- |-------|
| Set triple             | 1     |
| Delete triple          | 2     |

```proto
enum OpType {
  SET_TRIPLE = 1;
  DELETE_TRIPLE = 2;
}
```

#### 13.1 Set triple

When the OpType is `Set triple`, all 3 EAV fields of the triple must be included.

| Name            | Value | Description                                        |
| --------------- |-------| -------------------------------------------------- |
| Entity          | 1     | The entity ID for the triple that's being created  |
| Attribute       | 2     | The attribute ID for the triple being created      |
| Value           | 3     | A value with type, value, and optional options     |

#### 13.2 Delete triple

When the OpType is `Delete triple`, just the EA fields of the triple should be included.

| Name            | Value | Description                               |
| --------------- |-------| ----------------------------------------- |
| Entity          | 1     | The entity ID to delete the triple for    |
| Attribute       | 2     | The attribute ID to delete the triple for |

### 14. Edits

Changes are published to the knowledge graph through `edits`. An edit can include any number of changes to any entity in a space.

| Name            | Type            | Value | Description                                               |
| --------------- |-----------------|-------| --------------------------------------------------------- |
| Version         | string          | 1     | A semver version string                                   |
| Type            | ActionType      | 2     | The type of action this edit is encoding                  |
| ID              | string          | 3     | A valid ID representing this edit                         |
| Name            | string          | 4     | The name of the edit (short commit message)               |
| Ops             | Op[]            | 5     | A list of triple operations to perform                    | 
| Authors         | string[]        | 6     | The list of authors who worked on this edit               |

The corresponding protobuf message is:

```proto
message Edit {
  string version = 1;
  ActionType type = 2;
  string id = 3;
  string name = 4;
  repeated Op ops = 5;
  repeated string authors = 6;
}
```

#### Example Edit in JSON

```jsonc
{
  "version": "1.0.0",
  "type": 1, // Add edit
  "id": "JVrauVCjqsuKqArK3dutYb",
  "name": "Add a new city",
  "ops": [
    {
      "type": 1, // Create triple
      "triple": {
        "entity": "Gw9uTVTnJdhtczyuzBkL3X",
        "attribute": "7UiGr3qnjZfRuKs3F3CX61",
        "value": {
            "type": 1, // Text
            "value": "San Francisco"
        }
      }
    }
  ],
  "authors": ["7UiGr3qnjZfRuKs3F3CX61"]
}
```

### 15. Imports

Spaces are abstracted from the underlying infrastructure. A space contract can be deployed on one L2 and then move to another L2. An import file includes all of the data from a previous space and can be used to bootstrap the space in the new environment.

```proto
message Import {
  string version = 1;
  ActionType type = 2;
  string previousNetwork = 3;
  string previousContractAddress = 4;
  repeated string edits = 5;
}
```

#### Example Import in JSON

```json
{
  "version": "1.0.0",
  "type": "import",
  "previousNetwork": "7UiGr3qnjZfRuKs3F3CX61",
  "previousContract": "0x1A39E2Fe299Ef8f855ce43abF7AC85D6e69E05F5",
  "edits": [
    "ipfs://QmVGF2e9WqF8g39W81eCQxecz7Bdi5o2DFSJ5BQxwfveYB"
  ]
}
```

### 16. Action Types

Actions are logical functions that can be performed on a space in different environments. Public spaces should secure these actions behind a public governance system, personal spaces can secure these actions behind an access control system, and private spaces can run these actions locally. Actions can be used to describe a log of user intents as well as a log of a space's accepted actions.

```proto
enum ActionType {
  ADD_EDIT = 1;
  ADD_SUBSPACE = 2;
  REMOVE_SUBSPACE = 3;
  IMPORT_SPACE = 4;
  ARCHIVE_SPACE = 5;
}
```

#### 16.1 Add edit

Params: `edit` - The URI to the edit protobuf file. Edit files must be hosted on decentralized storage.

#### 16.2 Add subspace

Params: `subspace` - The space entity ID to add as a subpace of the current space.

#### 16.3 Remove subspace

Params: `subspace` - The space entity ID to remove as a subpace of the current space.

#### 16.4 Import space

Params: `import` - The URI to the import protobuf file. Import files must be hosted on decentralized storage.

#### 16.5 Archive space

Flags the space as unused. Indexers and clients can still index the space if desired but an archived space is no longer maintained. Public spaces whose data was stored on decentralized storage and anchored onchain should continue to be available forever to anyone who wants to use or reference them.

## Copyright waiver

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

