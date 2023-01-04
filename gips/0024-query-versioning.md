---
GIP: 0024
Title: Query Versioning
Authors: Zac Burns <zac@edgeandnode.com>
Created: 2022-03-01
Updated: 2022-03-09
Stage: Draft
Discussions-To: https://forum.thegraph.com/t/gip-0024-query-versioning
Depends-On: GIP-0025 DataEdge
---

# Abstract

From time to time, it is necessary to evolve the GraphQL API to add, deprecate, or change the behavior of features. Without a carefully crafted and communicated policy on query versioning in The Network, dApp developers would be exposed to API breakage and subtle security issues.

This GIP describes The Graph's holistic query versioning policy and a new suite of tools to help dApp developers communicate their needs for backward compatibility while managing the risk of API breakage.

# High Level Description

Broadly, this GIP will describe the following changes:
* Using SemVer to describe the client's expected query behavior
* What kinds of query changes are allowed between versions
* Protocol determinism through attestation versioning
* Backward compatibility provided by graph-node, indexers, gateways, and libraries
* Managing the demand for outdated query versions
* Trust implications imposed by automatic versioning
* An on-chain query version registry
* Changes required in graph-node to implement network-wide versioning

# Detailed Specification

## Using SemVer

Semantic Versioning (SemVer) is a protocol for managing the risk of breakage caused by upgrading dependencies. Although it is possible for any minuscule change to break an arbitrary upstream component, in practice, the SemVer delta provides an indispensable signal to developers and automated systems for how much care should be taken when upgrading. We can leverage SemVer to version The Graph's query API.

Only the client knows the expected behavior for query execution. So, it is the client's responsibility to specify this behavior when making a request. The set of acceptable versions should be included in the URL's `api-version` query parameter. For example:

```

https://gateway.thegraph.com/api/[api-key]/subgraphs/id/[identifier]?api-version=[semver]

```

The SemVer interpretation used by The Graph will mimic cargo, the full specification of which can be found [here](https://doc.rust-lang.org/cargo/reference/specifying-dependencies.html). To understand the remainder of this GIP, it is only important to know that a SemVer specification can indicate a range of possible versions. For example, the SemVer `1.2.3` indicates the range `>=1.2.3, <2.0.0`, while the SemVer `=1.2.3` indicates only the exact version `=1.2.3`.

## Major, Minor, and Patch Changes

This proposal limits the nature of changes allowed between two query versions. If versions differ in MAJOR, no requirements restrict the possible differences in the responses. Features may be added, removed, or altered between versions.

If versions agree in MAJOR but differ in MINOR, the query may use new features added in the MINOR version. Otherwise, all responses must be identical when the query only uses features available in both MINOR versions (excepting PATCH changes).

If versions disagree only in PATCH, only bugfixes are allowed between versions. Example bugfixes are when a feature behaves in a way that is in clear violation of the documentation or the GraphQL spec. Otherwise, all responses must be identical between versions. There will never be any unilateral bugfixes or features that change responses without bumping the query version number.

If queries agree in MAJOR, MINOR, and PATCH, the response must be exactly the same (bitwise identical) even across multiple versions of graph-node. No matter which phase of verifiability a query uses (arbitration, fraud proofs, or validity proofs), it is vital to the decentralized network's security model that each query has a single deterministic response. Whenever the behavior of graph-node is changed such that any query may have a different response than the previous version, a new protocol version must be introduced to disambiguate the correct response. A notable exception to this is when fixing a query-time determinism bug. If a query has multiple possible outputs, it is acceptable to restrict the possible output set without incrementing the query version.

## Attestations

In the Arbitration stage of The Graph's verifiability roadmap, security relies on attestations - signed statements from the Indexer claiming that indexing and query have been performed correctly. In addition to other details that disambiguate the query, the attestation includes the query version in its EIP-712 domain struct.

Queries must be deterministic for disputes. Therefore, only query versions of the form `=MAJOR.MINOR.PATCH` are valid for attestations. This template is the minimal canonical form uniquely identifying a version. Other forms specifying ranges or pre-release identifiers may still be used when querying but must be disambiguated or disallowed for the attestation.

The query version is currently hardcoded to `"0"` in the contracts. This GIP proposes modifying the contracts so that calls to `createQueryDisputeConflict` and `createQueryDispute` specify the version used in the query. Instead of passing a query version string to contracts, calls will directly take a `bytes32` parameter for the EIP-712 domain separator to be passed into `encodeHashReceipt`. The EIP-712 domain separator passed in will uniquely identify the query version, saving calldata and hashing work.

## Backward Compatibility

Core developers are encouraged, but not required, to maintain backward compatibility by supporting as many query versions as is practical across releases. In particular, additive features from a MINOR version bump are expected to require little effort to support in a backward-compatible way.

In an ideal world, graph-node could support all previous query versions with each new release. But in practice, indefinite support for all previous versions in graph-node would be an undue burden to the core developers. Maintaining old code branches makes reasoning about query execution difficult. In the worst case, changes to the database may make it impossible to run some queries efficiently. Furthermore, subgraphs consumed by just a single dApp may only require a single version at a time. To maintain development velocity and produce a high-performance system, it is at the sole discretion of the core developers whether to maintain backward compatibility across releases and, if so, how much.

When the latest version of graph-node does not support an in-demand protocol version, the burden of backward compatibility falls to indexers. Indexers may fulfill the need for outdated query versions by setting up additional infrastructure to index and serve a copy of a subgraph against old database schemas and query logic. Indexers may offset their additional infrastructure costs and provide a forcing function for dApps to upgrade by increasing prices for queries using old versions. There is no limit to the number of versions that may be supported simultaneously in this way.

## Managing Demand for Outdated Versions

If dApps and consumers hardcode an exact version number, undesirable market demand for many old versions will emerge from the varying upgrade schedules of dApp source code. The result would be increased infrastructure requirements for indexers and increased costs for consumers as the capacity of indexers is fragmented.

To prevent this outcome, consumers should upgrade query versions automatically to limit the demand for outdated versions. This is the purpose of using SemVer in the protocol. By having a consumer specify a range of compatible versions like `1.2.3` instead of a specific version like `=1.2.3`, it becomes the job of indexer selection to match demand with a query version that is in supply. For example, indexer selection may choose an indexer supporting the range `>=1.3.0 <=1.3.5` (which overlaps with `1.2.3`) and send them a deterministic query for `=1.3.5`. Selection may prefer indexers on more recent versions or multiple indexers on the same version for cross-checking.

As long as the rules for what can be included in a version bump are adhered to in graph-node and consumers only rely on the latest features after a critical mass of indexers upgrade, then the network may rely on this system to guarantee smooth operation without requiring dApps and indexers to perform knife-edge upgrades.

It is still possible for a dApp to hardcode a specific version if it relies on buggy behavior from a query version no longer supported by the latest version graph-node. But, that dApp may expect to shoulder the burden of backward compatibility through increased cost and reduced choice in indexers.

## Trust implications

The specification for the desired behavior of indexing and query has similar trust requirements as a block hash. An attacker who can provide an arbitrary block hash into a query injects arbitrary insecure data into the result. An attacker who can provide an arbitrary version injects arbitrary insecure algorithms into the process. These are the same.

Any component that may automatically upgrade a version must be imbued with the same trust as would be used to make a query deterministic by injecting a block hash. In particular, an Indexer should never be trusted to select a query version that is unknown to the consumer.

Presently, some consumers choose to use a gateway such as the Edge & Node Gateway to inject block hashes. Or, they may use an open-source library such as The Graph Client by The Guild and trust the community to verify that the source implements the protocol correctly. In both cases, there is no additional trust assumption to delegate converting a SemVer range to a specific canonical SemVer if those components restrict the SemVer to a list of well-known, trusted versions.

## Query Version Registry
Due to the trust implications above, the manual effort required to update client code becomes the most prominent factor preventing timely upgrades. What is needed to manage demand for outdated versions is a source of trusted versions that clients can refer to when upgrading automatically.

The Query Version Registry provides an on-chain source of truth for versions backed by The Graph protocol's economic security guarantees. The Graph Council publishes supported versions to The Graph Protocol DataEdge. Subgraphs then make the list of supported versions available to components.

The DataEdge namespace for the Query Version Registry is `queryVersionRegistry(bytes _payload)`. The message format is a list of newly supported versions. The encoding for each version is a series of three integers in prefix varint format. These integers are interpreted as the SemVer `=[0].[1].[2]`.

Attesting to a query is automatically a slashable offense if the query version is not already in the query version registry when the signer's allocation is opened or added by the end of the epoch after the allocation is closed. This mechanism is meant to provide a strong deterrent to attempts to defraud users by injecting invalid query versions into a request.

The version `"0"` is grandfathered in as a valid version in the registry, but should be considered deprecated.

## Implementation in graph-node

The implementation for supporting query versions in graph-node is already underway: [#3024](https://github.com/graphprotocol/graph-node/issues/3024). There are several key features needed to implement this GIP.

Graph-node must parse the required version from the URL. If no version is specified in the URL, this will be interpreted as SemVer "1.*". Graph-node should always select the latest matching version when a range is specified. If graph-node cannot serve the query for the required range of versions, it must return a non-deterministic error. Graph-node must continue to treat non-deterministic queries as unattestable, including queries specifying a SemVer range. Graph-node must enumerate a list of supported SemVer versions in the indexing status API. Graph-node must include the query version in the attestation. Graph-node must consider the query version and return a deterministic response when serving a request for a schema, just as it does when serving a request for data. Lastly, graph-node should not knowingly serve non-deterministic queries when a (more) deterministic implementation is available. Core devs should not preserve non-determinism in query versions across graph-node versions. Appropriate alternative behaviors are throwing a non-deterministic error or running deterministic code that restricts the set of outputs from the previously non-deterministic version.

# Copyright Waiver

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).