---
GIP: "0050"
Title: File data sources
Authors: Adam Fuller (adam@edgeandnode.com), Leo Schwarzstein (leo@edgeandnode.com), Zachary Burns
Created: 2021-10-01
Updated: 2023-03-01
Stage: Draft
---

# Abstract

The current mapping API for IPFS makes the overall indexing output depend on whether or not the index node was able to find a file at the moment ipfs.cat was called. This is not deterministic and therefore not acceptable for the decentralized network. It also is not performant or reliable, as handler processing waits for the file to be found (or not) before proceeding.

A proposed solution consists of dynamic **"file" data sources** which correspond to content-addressed data (e.g. from IPFS), created by blockchain-based data sources. These file data sources are isolated, both in terms of being processed asynchronously from the main chain-based process, and in updating a discrete set of entities. This addresses some of the reliability & performance concerns, while laying the foundations for deterministic indexing (based on file availability consensus) in the future.

This GIP describes the implementation of file data sources in Graph Node, starting with IPFS as the first type of file data source.

# Motivation

As on-chain transaction costs have gone up, and dApp use-cases have expanded from DeFi into NFTs & other areas, the importance of indexing off-chain data has increased. Graph Node currently supports accessing data from IPFS, but that implementation is not deterministic, performant, or reliable. Meanwhile other data storage solutions, such as Arweave and Filecoin, are not supported at all.

In order to be useful to these emerging projects, Graph Node needs to be able to reliably, performantly and deterministically index subgraphs which depend on these off-chain data storage protocols.

# Prior Art

[Prior IPFS Datasource RFC](https://github.com/graphprotocol/network-rfcs/blob/master/rfcs/0003-data-availability-and-IPFS-data-sources.md)
[IPFS data sources graph-node issues](https://github.com/graphprotocol/graph-node/issues/1017)
[Existing IPFS documentation](https://thegraph.com/docs/developer/assemblyscript-api#ipfs-api)

# High Level Description

A new "kind" of data source will be introduced: `file/ipfs`. These data sources can be statically defined, based on a content addressable hash, or they can be created dynamically by chain-based mappings.

Upon instantiation, Graph Node will try to fetch the corresponding data from IPFS. If the file is retrieved, a designated handler will be run. That handler will only be able to save entities specified for the data source on the manifest, and those entities will not be editable by main chain handlers. This isolation means that variable file availability will not impact the determinism of the chain-based indexing.

# Detailed Specification

Our running example will be a subgraph with the following user information:

```graphql
User @entity {
  id: ID!
  name: String!
  email: String
  bio: String
}
```

In this case the `bio` and the `email` are found in an IPFS file, while the ID and name are sourced from Ethereum. This separation & merging at query time can be defined in the subgraph `schema.graphql` as follows:

```graphql

UserMetadata @entity {
  id: ID!
  email: String
  bio: String
}

User @entity {
  id: ID!
  name: String!
  metadata: UserMetadata
}
```

> We now have an entity which is _only_ dependent on IPFS (`UserMetadata`). This can therefore be considered separately for Proof-of-indexing.

A static file data source is then declared as:

```yaml
- name: UserBioIPFS
  kind: file/ipfs
  source: # Omitted in templates
    file: Qm...
  mapping:
    apiVersion: 0.0.6
    language: wasm/assemblyscript
    file: ./src/mappings/Mapping.ts
    handler: handleFile
    entities:
      - UserMetadata
```

- In this case `ipfs` is the type of `file`, indicating that this file is to be found on the IPFS network. This pattern could be extended to `file/arweave` and `file/filecoin` in the future.
- Other than `file/ipfs` we can imagine kinds such as `fileLines/ipfs` to replace `ipfs.map` and handle files line by line or `directory/ipfs` to stream files in a directory, however those precise definitions are out of scope for this document.
- The `entities` specified under the mapping are important in guaranteeing isolation - entities specified under file data sources should not be accessible by other data sources (and the file data source itself should only create those entities). This should be checked at compile time, and should also break in run-time if the `store` API is used to update a file data source.
- There can only be one handler per file data source.

In our example, the data source would not be static but a template, in which case the `source` field would be omitted. To instantiate the template the current `create` API can be used as:

```typescript
// Set the `User` entity.
let user = new User(newUserEvent.userId);
user.name = newUserEvent.username;
user.metadata = newUserEvent.userDataHash;
user.save();

// Get the bio from IPFS.
DataSourceTemplate.create("UserMetadata", newUserEvent.userDataHash);
```

The file handler would look like:

```typescript
export function handleFile(file: Bytes, content: Bytes) {
  let userMetadata = new UserMetadata(file.toHexString());
  userMetadata.bio = content.toString();
  userMetadata.save();
}
```

Note that in this case, the mapping of the Ethereum-based entity to the IPFS-based entity takes place entirely in the Ethereum data mapping, which allows for multiple Ethereum entities to reference the same file-based entity.

It is possible for an identical file data source to be created more than once (e.g. if multiple ERC721s share the same tokenURI on-chain). In this case the corresponding file handler should only be run once, and it will be up to the subgraph author to handle the many to one relationship (as in the example), or a scenario where an older file is re-used by handling the reference on the main chain mapping.

## Indexing IPFS data sources

When a file data source is created, Graph Node tries to find that file from the node's configured IPFS gateway. If Graph Node is unable to find the file, it should retry several times, backing off over time. On finding a file, Graph Node will execute the associated handler, updating the store. The associated entity updates will **not** be part of the subgraph PoI.

### Entities and block ranges

Entities which are created, updated and deleted by blockchain data sources are only limited by their main chain block range (`chain_range`) - i.e. no change is required for those entities.

Entities created by File data sources also have a `chain_range`, which is: `[startBlock,]`, where `startBlock` is the block when the underlying data source was instantiated on the source chain.

### Interacting with the store

- File data source mappings can only load entities from chain data source entities, up to the file data sources chain create block.
- Chain data source entities cannot interact with file data source entities in any way.

### Handling entities with the same ID

There is a use-case for different file data sources to update the same entity (i.e. an entity with the same ID).

> The initial implementation will not allow creation of entities with the same ID. Developers should be able to generate unique entities for each File data source, and should be able to specify the required pointers in the mappings for the main chain. This is a constraint, and developer feedback will be important in determining whether this constraint is loosened in future.

### Proof of indexing

For the initial implementation, file data sources will not impact the Proof of Indexing - entities created by file data sources can be isolated and excluded. With the introduction of an availability consensus mechanism for the indexed files, a proof of indexing inclusive of file data source entities could be constructed, but that is outside the scope of this GIP.

## Querying subgraphs using file-based data sources

Entities saved by file data sources will appear at query time the same as entities saved by chain-based data sources (i.e. no special handling is required).

> This would change with the introduction of a file availability consensus mechanism

## Monitoring

For monitoring purposes, the subgraph will want to keep track of its file data sources, including but not limited to:

- Number of file data sources
- Number of file data sources where the file has been found and processed

## Restrictions and extensions

- Chain-based data sources cannot load entities created by file data sources.
- File data sources cannot load entities not created by itself. This might be relaxed in the future, since in principle it could load entities in their state at the moment of the data source creation, or it could even be implicitly re-created at each chain block on which the data it depends on changes, but for now use the most restrictive semantics while we learn about the real world use cases.
- It seems useful to allow the originating data source to remove a file data source, stopping it and closing the `chain_range` on all entities. This could be done as `dataSource.remove("dsName")`, which would also require passing an unique name when creating the data source. This is a future extension.
- File data sources cannot create other data sources, of any type. We plan on allowing this but it is left to a future GIP.
- Other kinds of file data sources such as `lines` or `directory` are future extensions, as well as other types of file identifier (e.g. Arweave, HTTP)
- The use case of overwriting data from a new file is covered by the current design by simply spawning a new file data source which updates the resultant entities. However some use cases may require reading from the previous file to update the data, in which case an file data source will need to handle more than one file. This is left as a future extension.
- Establish a simple declarative way to merge entities by ID, to improve developer experience when merging entities created by file data sources, and chain-based data sources.
- Allow file data sources to create further file data sources, to facilitate indexing of recursive files or directories.
- A significant extension is the introduction of a file availability consensus mechanism, to allow for deterministic indexing of file data sources

# Backwards Compatibility

This will become a breaking change with the removal of the current ipfs API, which will require a new `apiVersion`. This is not currently planned for, and regardless, existing subgraphs will be able to use the existing implementation under prior `apiVersions`. Subgraph authors will be encouraged to upgrade their implementation though, to benefit from the improved reliability and performance.

# Dependencies

n/a

# Risks and Security Considerations

There are considerable drawbacks and risks with this proposal, such as:

- Users finding it difficult to understand the API or finding that it does match their expectation or does not fit the data model for their dApp. This design is based on what should be sufficient to migrate our current IPFS subgraphs, but it is hard to predict whether it is a good fit for future patterns that may emerge in dApps. The extensibility of the proposal mitigates this risk. Good documentation will also be important.
- Data sources concurrently modifying the store is significant change to the architecture. The implementation risks of introducing concurrency need to be evaluated in the engineering plan.
- We are assuming that the pattern described will work for other data storage solutions, such as Arweave and filecoin

# Copyright Waiver

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
