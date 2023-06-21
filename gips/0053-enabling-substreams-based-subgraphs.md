---
GIP: <0053>
Title: <Enabling-Substreams-Based-Subgraphs>
Authors: <Alex Bourget alex@dfuse.io, Adam Fuller adam@edgeandnode.com>
Created: <2022-07-19>
Updated: <2023-06-21>
Stage: <Draft>
Discussions-To: <https://forum.thegraph.com/t/ggp-0025-enabling-substreams-based-subgraphs/4287, https://forum.thegraph.com/t/substreams-into-subgraphs-a-simple-integration/3542>
---

# Abstract

Substreams are a new streaming-first blockchain indexing technology developed for The Graph Network, focused on performance and composability.

A substreams package is a collection of interdependent modules, which combine to extract and transform blockchain events in a highly configurable and parallelisable way, producing a streaming output.

This GIP describes the simplest possible integration of Substreams with Subgraphs, to take that final streaming output and make it persistent and queryable.

# Motivation

As Substreams become first-class citizens with The Graph Network, the StreamingFast team believes that Substreams software has been battle tested enough to be deemed fully production ready with The Graph Network. To ensure that Indexers will index substreams-back subgraphs, the GIP proposes to add Oracle support for dataSource.kind =="substreams" to the Feature Matrix shared below. 

This would make an important moment where the performance promises we have made in the last 2 years come to fruition. With a very important indexing-time performance boost, as well as an important injection-time performance boost.

Adding Substream support to Graph Node is the fastest way to bring the performance and composability benefits of substreams to The Graph Network.

# High Level Description

With the launch of Uniswap v3, lots of efforts have been put in stabilizing the engine. In doing so, multiple indeterminism issues were resolved. StreamingFast has done extensive testing for determinism, have validated substreams-graph-load (the high-speed injector) and graph-node loading PoI parity, as well as the gentle hand-off to graph-node with continued parity.

StreamingFast’s testing also showed convergence of POIs during live segments of the chain, even if hit by potential reorgs (this uses the standard support in graph-node so shouldn’t introduce any new risks).

The release Release v1.4.4 · streamingfast/firehose-ethereum · GitHub 1 is required for deterministic execution.

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

This introduces a new `substreams` “kind” of dataSource. This dataSource is identified on the `subgraph.yaml` by a name and a filepath. This filepath will point locally to a Substreams package, which will be uploaded to IPFS on deployment.

> Subgraphs with a substreams dataSource can only have that single dataSource.

#### Graph Node configuration
In order to support substream-based subgraphs, Graph Node will need to be Substream aware. An additional substream endpoint will be required for each network which Graph Node will want to support

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

Utilities should certainly be provided to auto-generate a `.graphql` file from a substream protobuf schema, with intelligent type mapping. However once generated, developers should be able to update the `.graphql` schema, to identify connections between entities, and to add derived fields. Therefore there is still value in having a schema file as part of the subgraph definition.

This proposal does assume a tight coupling between entity names in the substream, and entity names in the subgraph

#### Updating entities
In order to support performant querying, Graph Node stores full entities at every block where those entities change. It supports “upsert” type behaviour, where saved values are merged with existing entities.

Substreams currently provide entity “deltas”, which just capture the changed values. Therefore there is a requirement to fetch any prior entity values to create the full entity to store. This could be done on either the Substream or Graph Node side of the integration, so we should work through the trade offs & alternatives.

Graph Node will still need to “close” the block range for any existing entities, which could negatively impact indexing performance.

#### Deterministic indexing
A fundamental requirement is deterministic indexing, specifically the generation of a Proof of Indexing, which can then be cross-checked across indexers.

> Graph Node makes the assumption that all upstream datasources are deterministic, and will make the same assumption about Substreams

Depending on the implementation in Graph Node, this could leverage the existing Proof of Indexing setup, or it may need a dedicated Proof of Indexing in Graph Node.

#### Handling re-orgs
Graph Node will still need to be re-org aware. The substreams connection should provide the relevant information (which blocks to remove). Graph Node will need to remove entities from those blocks, and re-open the block range for older entities. This functionality already exists in Graph Node, but might need to be updated for the new data source.

#### Linear process & batch back-filling
Graph node will need to support linear processing, e.g. when indexing at the chain head. There is a possibility that linear processing historical blocks becomes a significant performance bottleneck. In that case, a dedicated “batch” insert implementation might be required to load historical data into the subgraph.

#### Monitoring
Graph Node’s instrumentation & monitoring may need to be updated for the new type of indexing (e.g. tracking indexing time per block etc).

#### Graph CLI
Graph CLI will need to be updated for the new type of data source. Graph CLI could also include the helpers to auto-generate `.graphql` files for a given substream.*

# Backwards Compatibility

This functionality is purely additive. Graph Node may need a new `specVersion` given the new structure of the manifest, but this could also be handled in the same way as the introduction of new protocols, where Graph Node versioning provides the necessary guarantees.

# Dependencies

- Substream support for a given network requires a Firehose implementation for that network.
- Requires determinism from the upstream Firehose & Substreams
- Substreams introduce additional infrastructural requirements for indexers
- This integration may require some changes as part of Substreams

# Risks and Security Considerations

Discovery of new indeterminisms are still possible, but mitigated by greater facility to cross-check at different stages of the pipeline. For example, the tool sfeth tools compareblocks can easily compare the source of Substreams between indexers if we need to find discrepancies. Substreams execution can produce flat files that can be again compared with great ease. And POI can be compared like we always do.

# Validation

- Determinism - are indexing results consistent?
- Performance - are the performance gains as expected?
- Scaleability - can Graph Node support 1-2K concurrent subgraphs powered by substreams?

# Rationale and Alternatives

- Alternative Postgres Sink
- Less opaque integration

# Copyright Waiver

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
