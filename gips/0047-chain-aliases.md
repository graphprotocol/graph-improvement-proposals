---
GIP: 0047
Title: Chain identification & aliasing for The Graph Protocol
Authors: Adam Fuller <adam@edgeandnode.com>
Created: 2022-07-28
Updated: 2022-08-12
Stage: Candidate
---

# Abstract

The Graph community is in the process of adding support for many more protocols & networks on The Graph Network. Graph Node, and The Graph Network currently relies on a freeform string label to match subgraph manifests with the correct upstream providers of blockchain data (RPC endpoints, firehoses). This GIP proposes that The Graph Network adopts CAIP-2 naming conventions for addition of chains to the network, with a list of supported aliases for backwards compatibility & readability.

# Motivation

This will bring The Graph Network in line with a community-developed standard, and make for a more robust process as more protocols and chains are added in future.

# Detailed Specification

Currently, only Ethereum mainnet subgraphs are supported on The Graph Network. These are identified via `mainnet` identifier, due to the Ethereum origins of The Graph. The Epoch Block Oracle will unlock addition of further indexed chains. This is therefore a timely moment to re-align on how chains are identified within the protocol.

[Chain Agnostic Improvement Proposals](https://github.com/ChainAgnostic/CAIPs) (CAIPs) describe standards for blockchain projects that are not specific to a single chain. [CAIP-2](https://github.com/ChainAgnostic/CAIPs/blob/master/CAIPs/caip-2.md) defines a way to identify a blockchain (e.g. Ethereum Mainnet, Görli, Bitcoin, Cosmos Hub) in a human readably, developer friendly and transaction-friendly way.

The chain_id is a case-sensitive string in the form

```
chain_id:    namespace + ":" + reference
namespace:   [-a-z0-9]{3,8}
reference:   [-a-zA-Z0-9]{1,32}
```

Each namespace covers a class of similar blockchains. Usually it describes an ecosystem or standard, such as e.g. cosmos or eip155. These are equivalent to the `protocol` specified on a subgraph manifest.

Examples:

```
# Ethereum mainnet
eip155:1

# Cosmos Hub (Tendermint + Cosmos SDK)
cosmos:cosmoshub-2
cosmos:cosmoshub-3

# Solana Mainnet
solana:4sGjMW1sUnHzSxGspuhpqLDx6wiyjNtZ

# Solana Devnet
solana:8E9rvCKLFQia2Y35HXjjpWzj8weVo44K
```

Namespaces are maintained in a [dedicated repository](https://github.com/ChainAgnostic/namespaces).

This GIP proposes that The Graph Network adopts CAIP-2 identifiers for the addition of new chains, via GIP-0008 (Subgraph API Versioning and Feature Detection). When The Graph Council approves support for new kinds of data sources, it should approve a CAIP chain_id. It may also approve a list of aliases as a comma separated list, for backwards compatibility.

For example, approving the addition of [Polygon](https://polygon.technology/):
```
feature: eip155:137
aliases: polygon,matic
experimental: true
queryDisputes: true
indexingDisputes: true
indexingRewards: true
```

Once The Graph Council has approved a given addition, it should be added to a dedicated "Protocols & networks" section of the `networks.md` ([link](https://github.com/graphprotocol/indexer/blob/main/docs/networks.md)):

| Network    | Alias         |
|------------|---------------|
| eip155:1   | mainnet       |
| eip155:137 | polygon,matic |

If it introduces a new protocol, that should also be added accordingly:

| Protocol | Aliases  |
|----------|----------|
| eip155   | ethereum |
| cosmos   |          |

If the network is approved for indexing rewards, that network should then be added to the Subgraph Oracle, so that subgraphs can be approved for indexing rewards, and the Epoch Block Oracle, so that indexers can close allocations. Indexers themselves may also need to update their Graph Node and Indexer configuration to support the new network, and any aliases.

Removal of rewards for a given network can be achieved by a similar governance decision, followed by removal from the network from the same components.

This GIP will require changes to support aliasing in several components:
- Graph Node will need to support aliases for network names & protocol names as part of its configuration. This is already a requirement, given the name changes on certain EVM networks (e.g. Polygon, GnosisChain)
- The Indexer Agent will need to match a subgraph's stated network with the Epoch Block Oracle's when closing allocations
- The Subgraph Oracle will need to be updated to support a growing list of protocols & networks which are eligible for indexing rewards

## Handling hard forks

CAIP-2 does not have explicit handling of hard forks, and some of the drawbacks of this approach are discussed in [this thread](https://github.com/ChainAgnostic/CAIPs/issues/22). Hard forks introduce a challenge for indexers - if the fork is contentious, it becomes unclear which fork should be processed for a given chain identifier.

This GIP and the Epoch Block Oracle GIP propose that The Graph Council approves CAIP identifiers. In a case where both forks are "viable" (which is to say there is meaningful activity present on both forks), it is reasonable to expect that The Graph Community will want to support applications on both chains. As such, participants will need to be able to specify which fork they are interested in.

In cases such as Ethereum, where EIP-155 integers are not fork friendly, the majority chain will be unchanged, so will already be supported on the network. The minority chain will need to be added, once the new chain identifier is established, via governance approval.

In cases where identifiers are fork-friendly, then both new identifiers may need to be explicitly added. If the hardfork is planned in advance, The Graph Council could pre-approve the chain identifiers, but it is more likely that this will be done after the fork has taken place.

If for whatever reason The Graph Council does not want to support a given fork, then that network's rewards eligibility can be removed.

There is a chance that The Graph community takes a different view on the alias for different CAIPs (for example which CAIP should be considered "mainnet"). However this GIP proposes that aliases are tightly coupled to CAIPs, even if that impacts the "sovereignty" of the community, as to change the CAIP corresponding to a given alias could cause significant confusion.

# Backwards Compatibility

Alias support in Graph Node, and a list of accepted aliases for networks, will provide backwards compatibility for this feature.

The Epoch Block Oracle has been implemented with CAIP-2 identifiers as the expected network identifiers.

# Rationale and Alternatives

- Continue to rely on custom string identifiers across networks
- Fully adopt CAIP end-to-end, which would not be backwards compatible

# Copyright Waiver

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).