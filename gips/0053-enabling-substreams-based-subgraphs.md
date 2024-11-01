---
GIP: "0053"
Title: <Enabling-Substreams-Based-Subgraphs>
Authors: <Alex Bourget alex@dfuse.io, Adam Fuller adam@edgeandnode.com>
Created: <2023-06-01>
Updated: <2023-06-21>
Stage: <Draft>
Discussions-To: <https://forum.thegraph.com/t/ggp-0025-enabling-substreams-based-subgraphs/4287, https://forum.thegraph.com/t/substreams-into-subgraphs-a-simple-integration/3542>
---

# Abstract

Substreams are a new streaming-first blockchain indexing technology developed for The Graph Network, focused on performance and composability.

A substreams package is a collection of interdependent modules, which combine to extract and transform blockchain events in a highly configurable and parallelisable way, producing a streaming output.

This GIP describes the simplest possible integration of Substreams with Subgraphs, to take that final streaming output and make it persistent and queryable.

# Motivation

To ensure that Indexers will index substreams-powered subgraphs, the GIP proposes to add Oracle support for dataSource.kind =="substreams" to the Feature Matrix. 

This would make an important moment where the performance promises we have made come to fruition. With a very important indexing-time performance boost, as well as an important injection-time performance boost.

Adding Substreams support to Graph Node is the fastest way to bring the performance and composability benefits of substreams to The Graph Network. 


# Detailed Specification

```typescript
specVersion: 0.0.X
schema:
  file: ./src/schema.graphql
dataSources:
  - kind: substreams
    name: Uniswap
    network: mainnet
    source:
      package:
        moduleName: output
        file:
          /: /ipfs/QmbHnhUFZa6qqqRyubUYhXntox1TCBxqryaBM1iNGqVJzT
          # This IPFs path would be generated from a local path at deploy time
    mapping:
      kind: substreams/graph-entities
      apiVersion: 0.0.X
```  

This introduces a new `substreams` “kind” of dataSource. This dataSource is identified on the `subgraph.yaml` by a name and a filepath. This filepath will point locally to a Substreams package, which are uploaded to IPFS on deployment.

> Subgraphs with a substreams dataSource can only have that single dataSource.


#### Updated Feature Matrix

This GIP proposes initial support for indexing rewards only on Ethereum mainnet (`mainnet`) `substreams`. However, it doesn't mean indexing rewards will be supported on a chain basis: `mainnet` is the starting point, based on early data determinism assurances and Indexer readiness to operate an Ethereum Firehose at scale, whose implementation has been battle-tested by core developers and the Indexer community during the [MIPs program](https://thegraph.com/migration-incentive-program/). More on data determinism can be found in [Deterministic indexing](#deterministic-indexing) below.

The proposed updated Feature Matrix, including the new `Substreams data sources` subgraph feature, with indexing rewards support for `mainnet` substreams:

More on Subgraph API versioning and feature support can be found in [GIP-008](https://github.com/graphprotocol/graph-improvement-proposals/blob/main/gips/0008-subgraph-api-versioning-and-feature-support.md).

| Subgraph Feature         | Aliases       | Implemented | Experimental | Query Arbitration | Indexing Arbitration | Indexing Rewards |
| ------------------------ | ------------- | ----------- | ------------ | ----------------- | -------------------- | ---------------- |
| **Core Features**        |               |             |              |                   |                      |                  |
| Full-text Search         |               | Yes         | No           | No                | Yes                  | Yes              |
| Non-Fatal Errors         |               | Yes         | Yes          | Yes               | Yes                  | Yes              |
| Grafting                 |               | Yes         | Yes          | Yes               | Yes                  | Yes              |
| **Data Source Types**    |               |             |              |                   |                      |                  |
| eip155:*                 | *             | Yes         | No           | No                | No                   | No               |
| eip155:1                 | mainnet       | Yes         | No           | Yes               | Yes                  | Yes              |
| eip155:100               | gnosis        | Yes         | Yes          | Yes               | Yes                  | Yes              |
| near:*                   | *             | Yes         | Yes          | No                | No                   | No               |
| cosmos:*                 | *             | Yes         | Yes          | No                | No                   | No               |
| arweave:*                | *             | Yes         | Yes          | No                | No                   | No               |
| eip155:42161             | artbitrum-one | Yes         | Yes          | Yes               | Yes                  | Yes              |
| eip155:42220             | celo          | Yes         | Yes          | Yes               | Yes                  | Yes              |
| eip155:43114             | avalanche     | Yes         | Yes          | Yes               | Yes                  | Yes              |
| eip155:250               | fantom        | Yes         | Yes          | Yes               | Yes                  | Yes              |
| eip155:137               | polygon       | Yes         | Yes          | Yes               | Yes                  | Yes              |
| **Data Source Features** |               |             |              |                   |                      |                  |
| ipfs.cat in mappings     |               | Yes         | Yes          | No                | No                   | No               |
| ENS                      |               | Yes         | Yes          | No                | No                   | No               |
| File data sources: IPFS  |               | Yes         | Yes          | No                | Yes                  | Yes              |
| Substreams data sources  | mainnet       | Yes         | Yes          | Yes               | Yes                  | Yes              


#### Graph Node configuration
In order to support substreams-powered subgraphs, Graph Node is Substream aware. An additional substream endpoint is required for each network which Graph Node will want to support

This can be an extension of the existing provider configuration in Graph Node:
```typescript
[chains.mainnet]
shard = "main"
protocol = "ethereum"
provider = [
  { label = "substreams-provider-mainnet",
    details = { type = "substreams",
    url = "https://mainnet-substreams-url.grpc.substreams.io/",
    token = "exampletokenhere" }},
]
```
Substreams may require dedicated error handling / retry logic, though a lot of the requirements should be covered by the Firehose cursor-based integration.

#### Mapping entities
Substreams have their own protobuf schemas. However to directly copy the raw substream schema would be to lose the ability to link and traverse entities in the resulting GraphQL schema.

Utilities are provided to auto-generate a `.graphql` file from a substream protobuf schema, with intelligent type mapping. Once generated, developers are able to update the `.graphql` schema, to identify connections between entities, and to add derived fields. Therefore there is still value in having a schema file as part of the subgraph definition.

This integration does assume a tight coupling between entity names in the substream, and entity names in the subgraph

#### Updating entities
In order to support performant querying, Graph Node stores full entities at every block where those entities change. It supports “upsert” type behaviour, where saved values are merged with existing entities.

Substreams currently provide entity “deltas”, which just capture the changed values. Therefore there is a requirement to fetch any prior entity values to create the full entity to store. This is done on the Graph Node side of the integration, so we should work through the trade offs & alternatives.

Graph Node will still need to “close” the block range for any existing entities, which could negatively impact indexing performance.

#### Deterministic indexing
A fundamental requirement is deterministic indexing, specifically the generation of a PoI, which can then be cross-checked across indexers.

> Graph Node makes the assumption that all upstream datasources are deterministic, and will make the same assumption about Substreams

Depending on the implementation in Graph Node, this could leverage the existing PoI setup, or it may need a dedicated PoI in Graph Node.

#### Handling re-orgs
Graph Node will still need to be re-org aware. The substreams connection should provide the relevant information (which blocks to remove). Graph Node will need to remove entities from those blocks, and re-open the block range for older entities. This functionality already exists in Graph Node, but might need to be updated for the new data source.

#### Linear process & batch back-filling
Graph Node will need to support linear processing, e.g. when indexing at the chain head. There is a possibility that linear processing historical blocks becomes a significant performance bottleneck. In that case, a dedicated “batch” insert implementation exists, but it is not the focus of the implementation.

#### Monitoring
Graph Node’s instrumentation & monitoring may need to be updated for the new type of indexing (e.g. tracking indexing time per block etc).

#### Graph CLI
Graph CLI will need to be updated for the new type of data source. Graph CLI could also include the helpers to auto-generate `.graphql` files for a given substream.

# Backwards Compatibility

This functionality is purely additive. 

# Dependencies

The release Release v1.4.4 · streamingfast/firehose-ethereum · GitHub 1 is required for deterministic execution.

- Substream support for a given network requires a Firehose implementation for that network.
- Requires determinism from the upstream Firehose & Substreams
- Substreams introduce additional infrastructural requirements for indexers
- This integration may require some changes to parts of the core Substreams engine

# Risks and Security Considerations

Discovery of new indeterminisms are still possible, but mitigated by greater facility to cross-check at different stages of the pipeline. For example, the tool sfeth tools compareblocks can easily compare the source of Substreams between indexers if we need to find discrepancies. Substreams execution can produce flat files that can be again compared with great ease. And PoI can be compared like we always do.

# Validation

With the launch of Uniswap v3, lots of efforts have been put in stabilizing the engine. In doing so, multiple indeterminism issues were resolved. StreamingFast has done extensive testing for determinism, have validated substreams-graph-load (the high-speed injector) and graph-node loading Proof of Indexing (PoI) parity, as well as the gentle hand-off to graph-node with continued parity.

StreamingFast’s testing also showed convergence of PoIs during live segments of the chain, even if hit by potential reorgs (this uses the standard support in graph-node so shouldn’t introduce any new risks).


- Determinism - are indexing results consistent?
- Performance - are the performance gains as expected?
- Scaleability - can Graph Node support 1-2K concurrent subgraphs powered by substreams?

# Rationale and Alternatives

- Alternative Postgres Sink
- Less opaque integration

# Copyright Waiver

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
