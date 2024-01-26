---
GIP: "0008"
Title: Subgraph API Versioning and Feature Support
Authors: Brandon Ramirez <brandon@edgeandnode.com>, Adam Fuller <adam@edgeandnode.com>
Created: 2021-04-27
Updated: 2022-11-01
Stage: Proposal
Discussions-To: https://forum.thegraph.com/t/a-process-for-specifying-the-subgraph-api-version-and-feature-support-matrix/2004
Category: Process

---

# Abstract
This proposal defines a process for defining the canonical behavior of the subgraph API in the protocol as well as establishing the matrix of subgraph API features and their corresponding supported protocol features.

# Motivation
A core value proposition of The Graph, as a decentralized protocol for indexing and querying public data is that a Consumer can trust the integrity of the work performed by the network, with minimal or zero trust in any individual Indexer. There are a number of techniques for achieving this with varying degrees of trust minimization. These include off-chain reputation systems as well as mechanisms that may be combined with slashing such as arbitration, refereed games and cryptographic fraud or validity proofs.

In general, the more trustless the mechanism, the greater the research and engineering effort required to implement it. This proposal therefore describes a support matrix comprising which subgraph API features can be used in conjunction with which features of the protocol, as determined by the strength of the techniques available for guaranteeing the integrity of said features. Having this granular support matrix allows new subgraph API features to be continuously and immediately added to the protocol with additional protocol features supported for those features in later stages--all while driving query volume for these features to the decentralized network, as opposed to any centralized services that might be used to expose these features.

Additionally, all the mentioned techniques with the notable exception of reputation systems, require that the work that Indexer perform be defined deterministically. Therefore, this proposal also describes how a canonical version of the subgraph API may be established via decentralized governance. This is a hard requirement for supporting protocol features such as disputing and slashing Indexers for invalid indexing or query work.

# High Level Description
The high level process consists of the following elements:
1. Graph Node as the reference implementation of the Subgraph API
2. Graph Node versioned using SemVer
3. Graph Council defines canonical Graph Node version
4. Certain named features may not be fully supported on The Graph network
5. Graph Council defines subgraph feature support matrix
6. (Optional) Graph Council defines N-1 support window

## Graph Node as reference implementation
In the absence of a detailed technical specification of the subgraph API, Graph Node shall act as the reference implementation for the behavior of the subgraph API. This means that behavior implemented by Graph Node, even that which might be considered "buggy", is canonical from the standpoint of the protocol. The official codebase of Graph Node is located in [this repo](https://github.com/graphprotocol/graph-node), which is maintained by The Graph Foundation with the help of external core contributors.

## Graph Node versioned using SemVer
Graph Node should be versioned using the widely used [SemVer](https://semver.org/) conventions. Specifically, backwards incompatible changes to Graph Node must be accompanies by a major version bump, unless those changes are to features that are marked as "experimental". Experimental features may be changed arbitrarily across minor versions, but should be stable across patch versions.

## Graph Council defines canonical Graph Node version & configuration
The Graph Council should specify the canonical version and operational configuration of Graph Node via a Graph Governance Proposal. This proposal should include an epoch number in which the upgrade to the new canonical version of Graph Node will become active. This version and configuration should be kept up to date in the `networks.md` file [in the graphprotocol/indexer repository](https://github.com/graphprotocol/indexer/blob/main/docs/networks.md).

## Certain named features may not be fully supported on The Graph network

Features within the specified Graph Node version are assumed to be deterministic, and fully supported on the network. However there are certain situations where a feature or configuration Graph Node cannot be fully supported. In this case, the feature or configuration can be introduced as a named feature, with its level of support specified via Governance.

Graph Node will act as the source of truth for what named features are being used by a subgraph. The subgraph manifest should clearly identify which named features they are using, and if Graph Node detects that a named feature is being used by the subgraph that is not available in the manifest, then it should consider that subgraph invalid.

*Note: Listing named subgraph features in the manifest need not place an undue burden on the subgraph developer as these features can be automatically added at build time using the [Graph CLI](https://github.com/graphprotocol/graph-cli).*

Graph Node should also expose an endpoint that accepts a query for a given subgraph, returning the named features the subgraph is using.

## Graph Council defines the Feature support matrix
The Graph Council should specify the feature support matrix for the named features included in the canonical version of Graph Node.

This support matrix will include the named subgraph features as rows and their corresponding supported protocol features as columns (see [Subgraph feature support matrix](#subgraph-feature-support-matrix)).

## (Optional) Graph Council defines N-1 support window
When upgrading the canonical Graph Node version via a Graph Governance Proposal, The Graph Council may specify a support window during which time the previous version of Graph Node may also be considered valid.

This should be specified as an epoch number in which the previous version will no longer be considered valid. This epoch number must be equal to or greater than the epoch number in which the new canonical version of Graph Node becomes active.

# Detailed Design

As a general point of priniciple, Graph Protocol core developers should aim to only introduce features which can be fully supported on the network. However this is not always practical, and certain useful features may not be deterministic.

## Determinism of subgraph functionality
A subgraph feature may lack determinism for any number of reasons, including but not limited to:
- The feature is experimental and not intended to be stable across minor versions of Graph Node.
- The engineering work has not yet been done to validate that feature is completely deterministic.
- Open research questions exist as to how to implement the functionality in a deterministic manner.
- There is a known determinism bug in the feature that has not yet been fixed.
- The data required to provide deterministic indexing is not available on The Graph Network.

A subgraph may lack determinism in *either* the indexing or querying capabilities that comprise the feature. If the only non-deterministic functionality of a subgraph are related to querying, then the remainder of the subgraph functionality may be treated as deterministic. However, if indexing functionality used by a subgraph is non-deterministic, then no other indexing or query related functionality in the subgraph may be treated as deterministic.

It is the responsibility of the Graph Protocol core development teams to identify new features, or configurations in Graph Node which are not deterministic, and propose the appropriate level of support to The Graph Council.

## Subgraph Data Source Types vs. Subgraph Core Features
With respect to the protocol support matrix for subgraph features, we can break subgraph features down into the following broad categories:

1. Data Source Types
2. Data Source Specific Features
2. Core Features

**Data Source Types** correspond to specific blockchains and storage networks indexed by The Graph.

Subgraph data sources have a "kind", which specifies a protocol, for example `ethereum`, or `near`. This is then paired with a `network`, which specifies the specific implementation of that protocol, for example `mainnet`.

> GIP-0040 (Chain Aliases) proposes that for The Graph Network should adopt CAIP-2 identifiers for protocols and networks.

While all the potential sources of non-determinism apply here, Data Source Types in particular are relevant to the data availability problem. Specifically:
- For non-mainnet blockchains, the protocol needs access to indexed-chain block information so that indexers can close allocations for specified epochs.
- For features which rely on files, the network needs consensus on the availability of those files.

**Data Source Specific Features**
These are features for supporting a specific data source type, such as host functions for calling the data source, data source specific types, or deserialization functions.

An example of such a feature might be the ability to use `eth_call` w/ [EIP-1898](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-1898.md) support.

**Core Features** correspond to functionality for indexing and querying blockchain data, that are completely agnostic of the underlying data source type.

An example of such a feature might be the ability to send full-text search queries.

Furthermore, all subgraph features comprise one or both of the following two types of functionality:

1. **Indexing functionality.** These include changes to the mapping API and supported GraphQL schemas. These have the potential to impact Proofs of Indexing (PoIs) as well as query  Attestations.
2. **Query functionality.** Includes changes to subgraphs' GraphQL query API. These do not impact Proofs of Indexing (PoI) but will impact query Attestations.

Core features may comprise both indexing and query capabilities, while data source types and data source specific features primarily impact indexing functionality.

## Levels of trust minimization
There are two levels of trust minimization that exist in the protocol today:
1. Reputation
2. Arbitration

Additionally, there are other forms of trust minimization that may be added to the protocol in the future, including but not limited to:
3. Refereed games for indexing fraud proofs
4. Cryptographic query fraud proofs
5. Cryptographic query validity proofs

**Reputation**. If a Consumer notices that an Indexer that has likely provided an invalid query or Proof of Indexing (PoI), for example by comparing the result to that of another Indexer with high reputation, then they may lower their reputation score. Reputation may be local to an individual Consumer or shared amongst many Consumers.

**Arbitration**. An Arbitrator, nominated via decentralized governance, reproduces the work of an Indexer, and decides the outcome of query or indexing related disputes. An Indexer that loses a dispute is slashed, which means a portion of their staked tokens are forfeit.

Given that arbitration relies on being able to reproduce the work of an Indexer, it is only available for deterministic subgraph functionality. A subgraph query that uses non-deterministic query functionality may not be submitted for a dispute settled via arbitration. A subgraph that used non-deterministic indexing functionality may not be involved in an indexing dispute nor a query dispute.

## Protocol feature trust requirements

Certain features of the protocol either directly relate to or depend on the level of trustlessness supported by the indexing or query functionality of a subgraph. These features include:
- Query disputes and arbitration
- Indexing disputes and arbitration
- Indexing rewards

Indexing rewards are intended to reward real indexing work, thus if indexing work can not be verified by an Arbitrator, then it should not be rewarded at the protocol level.

Additionally, it is useful to to explicitly note that any feature that has been implemented in Graph Node supports the following feature, regardless of the level of trustlessness supported by the feature:
- Queries
- Query fees (for Indexers, Delegators, Curators)
- Agora cost models

## Subgraph feature support matrix
The subgraph feature support matrix defines the intersection of named features with their corresponding status or protocol features that may or may not be supported by them.

It also groups in such a way that corresponds to where the named feature would show up, either in the subgraph manifest, or in the query-specific feature detection endpoint in Graph Node.

Example structure:

| Subgraph Feature         | Aliases | Implemented | Experimental | Query Arbitration | Indexing Arbitration | Indexing Rewards |
|--------------------------|---------|-------------|--------------|-------------------|----------------------|------------------|
| **Core Features**        |         |             |              |                   |                      |                  |
| Full-text Search         |         | Yes         | No           | No                | Yes                  | Yes              |
| Non-Fatal Errors         |         | Yes         | Yes          | Yes               | Yes                  | Yes              |
| Grafting                 |         | Yes         | Yes          | Yes               | Yes                  | Yes              |
| **Data Source Types**    |         |             |              |                   |                      |                  |
| eip155:*                 | *       | Yes         | No           | No                | No                   | No               |
| eip155:1                 | mainnet | Yes         | No           | Yes               | Yes                  | Yes              |
| near:*                   | *       | Yes         | Yes          | No                | No                   | No               |
| cosmos:*                 | *       | Yes         | Yes          | No                | No                   | No               |
| arweave:*                | *       | Yes         | Yes          | No                | No                   | No               |
| **Data Source Features** |         |             |              |                   |                      |                  |
| IPFS on Ethereum         |         | Yes         | Yes          | No                | No                   | No               |
| ENS                      |         | Yes         | Yes          | No                | No                   | No               |


- Experimental features *may* be eligible for disputes & rewards, but in the event of a dispute, indexers who cooperate with the Arbitrator where experimental features are involved can expect greater leniency.
- Data Source Types adopt a `*` wildcard matching to indicate all chains for a given protocol.

## Graph governance proposal format
Governance proposals should specify the Epoch, the Graph Node version, and the updated Feature Support Matrix. Note that these proposals can _reduce_ as well as _increase_ the level of support for a given feature.

Example proposal:
```
graph-node: ^0.28.0
valid from: 630
upgrade window: 644

| Subgraph Feature         | Aliases | Implemented | Experimental | Query Arbitration | Indexing Arbitration | Indexing Rewards |
|--------------------------|---------|-------------|--------------|-------------------|----------------------|------------------|
| **Core Features**        |         |             |              |                   |                      |                  |
| Full-text Search         |         | Yes         | No           | No                | Yes                  | Yes              |
| Non-Fatal Errors         |         | Yes         | Yes          | Yes               | Yes                  | Yes              |
| Grafting                 |         | Yes         | Yes          | Yes               | Yes                  | Yes              |
| **Data Source Types**    |         |             |              |                   |                      |                  |
| eip155:*                 | *       | Yes         | No           | No                | No                   | No               |
| eip155:1                 | mainnet | Yes         | No           | Yes               | Yes                  | Yes              |
| near:*                   | *       | Yes         | Yes          | No                | No                   | No               |
| cosmos:*                 | *       | Yes         | Yes          | No                | No                   | No               |
| arweave:*                | *       | Yes         | Yes          | No                | No                   | No               |
| **Data Source Features** |         |             |              |                   |                      |                  |
| IPFS on Ethereum         |         | Yes         | Yes          | No                | No                   | No               |
| ENS                      |         | Yes         | Yes          | No                | No                   | No               |
```

> The feature support matrix is valid only for Graph Nodes running the latest specified version (based on the epoch).

## Subgraph Manifest format

In order to improve legibility, relevant features can be added to the manifest under `features`. This has been done for some of the existing experimental features:

```
features:
  - fullTextSearch
  - ipfsOnEthereumContracts
```

# Copyright Waiver

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
