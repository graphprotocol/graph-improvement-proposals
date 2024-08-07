---
GIP: "0042"
Title: A World of Data Services
Authors: jannis@edgeandnode.com, adam@edgeandnode.com
Created: 2022-12-01
Updated: 2024-07-02
Stage: Draft
Category: "Protocol Logic"
Discussions-To: https://forum.thegraph.com/t/gip-0042-a-world-of-data-services/3761
---

<!--

The 2024-07-02 update reformats this GIP into standard format for easier parsing by both humans and automated parsers, corrects lint errors, makes a few minor wording and emphasis corrections, and links to the forum post.

No material changes to the content of the proposal itself.

-->

## Abstract

This GIP aims to establish a framework that allows us to expand the data services/APIs offered by The Graph Network over time, without having to make invasive changes to the protocol every time. The proposal is rooted in the realization that subgraphs are not well-suited for a number of use cases and that a healthy and efficient indexer network requires specialization and reuse of data already generated by the network.

The primary change proposed by this GIP is to abstract subgraphs in the protocol contracts into a more general concept of data services that consist of publishable data sets. This necessarily also affects some of the logic around allocations and rewards as well as discovery of the new data services/sets.

## Motivation

Two definitions upfront:

1. A *data service* is a type of data set or API. Examples of data services are subgraphs, substreams or Firehoses.
2. A *data set* is a specific data set, e.g. a subgraph for Decentraland or a substream for Uniswap swaps.

In other words: a data service is the technology or type of API, a data set refers to a specific set of data.

The motivations for the proposed changes are manyfold. 

Firstly, based on feedback from the developer community over the past few years, it has become clear that subgraphs are not well-suited for a number of use cases such as analytics pipelines or combining subgraph data with external and potentially off-chain data sources. New developments in The Graph ecosystem, most notably Firehose and Substreams, are emerging as solutions to address some of these use cases. Other types of APIs will undoubtedly follow and it is vital for The Graph to be able to support them natively in the network.

Secondly, to maintain a scalable, healthy, diverse indexer network, The Graph needs to allow for specialization and outsourcing of processing power and storage among indexers. For example, one indexer might specialize in providing raw, low-level blockchain data in the form of a Firehose, another might focus entirely on substreams processing, yet another might focus on indexing and serving subgraphs. The interactions between these indexers require a decentralized data market. As luck would have it, The Graph has already established such a market around subgraphs. It merely needs to be extended to support more data services.

## High Level Description

This GIP proposes to make it possible to extend the data services offered by The Graph Network. The GIP only covers changes proposed at the protocol layer, i.e. in the smart contracts. More specific GIPs for the first additional data services will follow soon. This section describes how the contracts could be changed at a high level.

Three main changes are proposed:

1. **GNS contract**: Instead of assuming that everything that is created and published is a subgraph or a subgraph version, add a notion of data service types such that:
    - the council can add new data service types,
    - each data service type comes with basic meta data such as a name and description,
    - data sets of any supported data service type can be created,
    - new data set versions of any supported data service type can be published,
    - an IPFS manifest hash is required for every new data set version that is published.

2. **Staking contract**: Instead of assuming that all allocations are against subgraph deployments, allow indexers to specify the data service type when creating an allocation such that:
    - each allocation is associated with a data service type,
    - each allocation is associated with a data set ID.

3. **Verifiability router**: Instead of using a single dispute mechanism, expand the contracts to allow a different dispute mechanism for each supported data service type, and to allow this dispute mechanisms to change over time. The existing disputes mechanism is kept for subgraphs. Additional data service types can then specify their own contract based on disputes or verifiable proofs.

How consumers, gateways and indexers need to be updated to support new data service types is left to the GIPs that introduce these new data service types. The detailed specification below illustrates how the changes above can be introduced in a completely backwards-compatible way.

## Detailed Specification

TBD.

This will require input from the smart contract developers working on The Graph. This team will know how to change the contracts best to make the above possible.

## Backwards Compatibility

TBD.

It should be possible to maintain the current publishing and allocation behavior by introducing *additional* contract functions rather than changing the existing ones. But whether this is possible largely depends on the changes specified in *Detailed Specification*.

We anticipate some changes in the network subgraph to be necessary, but this too remains to be seen.

## Dependencies

None.

## Risks and Security Considerations

This proposal will likely require a lot of changes especially in the GNS contract. Since GNS publishing (and the discovery experience in e.g. The Graph Explorer) are decoupled from allocation management and rewards, we could do the development (and even the detailed specification) in phases, starting with just adding the notion of council-controlled data service types to the GNS and allowing to associate allocations with data service types.

This would immediately allow indexers to allocate towards new types of data sets (like substreams or Firehoses) and unblock integrating new data service types end to end. Developers could start using the new services and discovery of these new data sets could follow in a second phase.

Any changes proposed to the smart contracts will, of course, require an audit.

## Copyright Waiver

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
