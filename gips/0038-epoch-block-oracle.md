---
GIP: "0038"
Title: Epoch Block Oracle
Authors: Adam Fuller (adam@edgeandnode.com), Ariel Barmat (ariel@edgeandnode.com), Zachary Burns
Created: 2022-02-11
Updated: 2022-11-01
Stage: Draft
Category: Protocol Interfaces
---

# Abstract

Introduce an "Epoch Block Oracle" to specify the currentEpochBlock on other chains, unlocking indexing rewards and network growth.

# Motivation

When indexers are indexing a subgraph, they allocate some of their stake towards that subgraph. This allocation indicates that they are indexing the subgraph, so they should be able to serve queries, and they will be entitled to collect query rewards.

In order to claim indexing rewards for a given epoch, they can close their allocation:

> An allocation must be closed with a valid proof of indexing (POI) that meets the standards set by the arbitration charter in order to be eligible for rewards.

> A POI for the first block of the current epoch must be submitted when closing an allocation for that allocation to be eligible for indexing rewards

[Source](https://thegraph.com/docs/indexing)

Information about the current Epoch, and the block that should be used for closing allocations, is provided by the [EpochManager contract](https://github.com/graphprotocol/contracts/blob/dev/contracts/epochs/EpochManager.sol):

```tsx
/**
     * @dev Return block where the current epoch started.
     * @return The block number when the current epoch started
     */
    function currentEpochBlock() public view override returns (uint256) {
        return lastLengthUpdateBlock.add(epochsSinceUpdate().mul(epochLength));
    }
```

Epochs are currently defined as 6646 blocks on Ethereum mainnet (approximately 24 hours). This epoch length is configured in the EpochManager contract, which can be [inspected on mainnet here](https://etherscan.io/address/0x64F990Bf16552A693dCB043BB7bf3866c5E05DdB#readProxyContract). This parameter can be updated via Graph Protocol governance.


If the network is to support indexing rewards for subgraphs indexing networks other than Ethereum mainnet, Indexers need to know which block POI to submit when closing an allocation.

Supporting additional indexing networks is a high priority for The Graph Protocol, given the growth of protocols and applications outside mainnet Ethereum.

### Glossary

- Protocol chain: the chain where The Graph's protocol contracts reside (Ethereum mainnet)
- Indexed chain: the chain indexed by a specific subgraph

# Detailed Specification

We will introduce an Epoch Block Oracle subgraph, indexing a simple on-chain [Data Edge](https://forum.thegraph.com/t/gip-0025-dataedge/3161), which will track the "Epoch Block" for all networks supported for indexing rewards.  Indexers will use block hashes specified by the subgraph to close their allocations for subgraphs indexing a given network.

[Reference implementation](https://github.com/edgeandnode/block-oracle)

###  Data Edge

This is a simple contract with a `fallback` function, which accepts any payload ([discussion](https://forum.thegraph.com/t/gip-0025-dataedge/3161)). There is no execution as part of the contract, the payload format is decoded and processed in the subgraph. This is a maximally gas-efficient approach ([~25K gas per update](https://ropsten.etherscan.io/tx/0xf26811133681e070a9c3134e4a1a59ccac1c4e6ec01329ca7059eac531d617c7)): no data is processed, there is no storage as part of Ethereum Mainnet state (only as calldata), all the logic is off-chain. 


```solidity
// SPDX-License-Identifier: GPL-2.0-or-later

pragma solidity ^0.8.12;

/// @title Data Edge contract is only used to store on-chain data, it does not
///        perform execution. On-chain client services can read the data
///        and decode the payload for different purposes.
contract DataEdge {
    /// @dev Fallback function, accepts any payload
    fallback() external {
        // no-op
    }
}
```

[Source](https://github.com/edgeandnode/block-oracle/blob/main/packages/contracts/contracts/DataEdge.sol)
[Sample deployment on Ropsten](https://ropsten.etherscan.io/address/0xae6add894f8a1bcac10b153dc59cab1da9656836)

> **This contract is generic** - it makes no assumptions about the purpose of the calldata, and could be re-used for other oracle use-cases in the future (either the same subgraph could be extended, or a new subgraph could be created for the new use-case).

### Oracle messages

In order to minimise the operational cost of the subgraph, we will make the calldata to update the oracle as small as possible. The [Data Edge GIP](https://forum.thegraph.com/t/gip-0025-dataedge/3161) goes into more detail on some of the associated trade-offs and considerations.

For our purposes, there are several messages types which the Epoch Block Oracle might want to submit. The most efficient approach is to pass an array of such messages in a single call, using a single generic selector. The logic of parsing that array of messages can then live purely in the subgraph.

> You could theoretically use different selectors, but this would mean making multiple different transactions, instead of a single one.

[In-progress example implementation](https://github.com/edgeandnode/block-oracle/pull/1)

#### SetBlockNumbersForEpoch

This message updates the oracle with relevant Epoch Block numbers, for specified networks. This is a "normal operations" oracle update, providing the block numbers to be used to close allocations on different networks (identified by a network id).

This data lends itself to stateful compression. The message data is a single integer indicating the epoch delta, followed by an array of N integers where N is the number of currently registered networks under this owner. Each integer is the next entry of the series of values in the 2nd derivative of the sequence of epoch block numbers for a given network. The order of the networks in the array is the currently registered networks sorted by network id.

An analysis of a small subset of recent Ethereum data showed that each entry would require between 1 and 2 bytes, averaging 1.35 bytes.

It is worth clarifying why we are using block numbers, rather than block hashes. This makes for highly efficient data encoding, and controls for variation in block hashes across networks. In taking this approach we make the assumption that all networks have block numbers, and all networks have temporary availability of block hashes. On the latter assumption:

- Data availability is required because without it an indexer cannot verify that the block hash they want to close with is a part of the root specified in the event of a CorrectEpochs message.
- This data availability only needs to be temporary, as the cap on the query dispute process means that permanent data availability is not necessary.

#### CorrectEpochs

If it becomes non-obvious which block hash a block number corresponds to the canonical chain head (because a deep re-org overwrites a block though to be re-org safe), then a CorrectEpochs message may be used to disambiguate.

The CorrectEpochs message consists of an integer-length prefixed array. Each item in the array is an integer for networkId, followed by a block hash. This block hash disambiguates all parent epochs.
From time to time, a CorrectEpochs message may be necessary to associate a chain fork with a network id.

#### Register networks

This message will be required whenever a new network is to be supported by the Epoch Block Oracle, or when a network is to be removed. Added networks are strings encoded as bytes. Removed networks are identified by their network id.

**New networks will only be added to the subgraph when the caller is the Graph Council multisig**. To allow for operational agility, the owner will be able to remove and re-add networks, in case (for example) a network needs to be temporarily removed.

#### UpdateVersion

The owner may at some point want to change the encoding used by its messages. To ease corresponding versioning in the subgraph, it is simplest if the owner provides an explicit Update Version message.

This message data will of a single integer delta of the current encoding version. This initial implementation has only one encoding (version 0), so this message is unused but necessary to make future upgrades to the message format.

If the UpdateVersion message is encountered within a message block, it signifies the end of that message block and the start of the following message block.

#### Reset
This message has no associated data and signifies that the Epoch Subgraph should reset all known chains, effectively reverting to an empty network list

### Epoch Block subgraph

This subgraph will process the calls to `fallback` function in the Data Edge. For ease of use, the encoding will include a selector which can be used in human-readable callhandlers:

```
  callHandlers:
    - function: crossChainEpochOracle(bytes)
      handler: handleCrossChainEpochOracle
```

The subgraph will first verify that the caller is a specified Owner account, or the Graph Council Multisig, and if it is, it will process the calldata (based on the messages described above), updating the Epoch & EpochBlock Entities accordingly.

#### Indicative Schema

```graphql
type Epoch @entity {
  id: ID!
  epochNumber: BigInt!
  blockNumbers: [NetworkEpochBlockNumber!]! @derivedFrom(field:"epoch")
}

type NetworkEpochBlockNumber @entity {
  id: ID!
  acceleration: BigInt!
  delta: BigInt!
  blockNumber: BigInt!
  network: Network!
  epoch: Epoch!
}
```

[Source](https://github.com/edgeandnode/block-oracle/tree/main/packages/subgraph)

This Subgraph will become a key operational part of the network, and its availability is therefore very important (similar to existing network subgraphs). Maintenance will be the responsibility of the core development teams.

Indexers will be expected to index it as a part of their operations, though some may rely on versions run by trusted parties across the network.

The canonical version, including the current "Owner", will be kept up-to-date on [`networks.md`](https://github.com/graphprotocol/indexer/blob/main/docs/networks.md), with any upgrades proactively communicated to the community.

### Oracle operations

Given the Data Edge, agreed-upon messages, and a subgraph which can decode and present the resulting data, the primary requirement is to then operationalise the Oracle.

At the outset, this will be the responsibility of Graph Protocol core development teams, who will run the oracle altruistically (in line with other protocol components, such as the subgraph oracle and the E&N Gateway). In future, this responsibility could be distributed, and incentivized.

There are three types of activity: routine (i.e. daily updating of the Epoch block across networks), planned (e.g. adding a new network, or updating the encoding version) and emergency (e.g. correcting epochs, in the event of an error or a deep fork).

#### Connecting to supported networks

The Oracle will need to connect to clients for all the networks which are supported by The Graph Protocol, fetching a block from each for a given Epoch. Precision (in terms of aligning blocks exactly, in time) is less important than reliability (having a block that is on the main chain in an appropriate time-frame). The existing rule for Ethereum Mainnet (6646 blocks / epoch) could be used as the starting point, fetching corresponding block hashes on different networks.

> The oracle will be responsible for identifying deep forks, or other anomalies or edge cases which might disrupt normal operations, or require an "emergency" activity.

#### Generating the encoded data

Given an array of selected blocks from the relevant networks, the Epoch Oracle will need to encode the data, in line with [the specification](#Oracle-messages). This may involve fetching previous data from the subgraph, in order to aid data compression.

As well as generation of calldata for the Epoch Block Oracle application to execute itself, the Graph Council will need the relevant calldata prepared when they add support for new networks via the multisig.

*Work in progress [Pull Request](https://github.com/edgeandnode/block-oracle/pull/1/files)*

#### Updating the Data Vault

The Oracle will then need to call the Data Edge on Ethereum Mainnet with the generated calldata. This will then be automatically indexed by the Epoch Oracle subgraph, and the latest data will be available to indexers.

As such the active account will need to have sufficient ETH balance for ongoing gas fees, and a robust retry mechanism will be required, in case transactions are not mined successfully.

#### Planned updates

The Oracle will need to respond to governance or other decisions, for example to add a new network, or to upgrade the encoding version. Note that upgrading the encoding version will require coordinated upgrades across Oracle operations, and the Epoch Block Oracle subgraph. As such it is proposed that the Oracle Operations team is also responsible for the Epoch Block Oracle subgraph.

#### Monitoring & decision-making

Oracle Operations will need appropriate monitoring in order to identify oracle malfunctions (e.g. software failure, low balance etc.), and to identify indexed-chain problems (e.g. deep forks). This will be necessary to mitigate problems before they have an impact on indexers on the network

In some cases (for example when a deep fork is identified), decision making may be required, for example choosing to submit a `CorrectEpochs` message to disambiguate the fork for indexers on the network.

### Indexer Agent

The Indexer Agent currently fetches the `currentEpochBlock` from the `EpochManager` ([link](https://github.com/graphprotocol/indexer/blob/main/packages/indexer-agent/src/agent.ts#L179)), and finds the blockHash by fetching the block from an Ethereum client.

This will need to be updated to fetch the latest Epoch Block for the relevant network from the Epoch Block subgraph, depending on which network a subgraph is indexing.

This may also require cross-checking that the indexed block hash for the block number they want to close with is a part of the root, specified in the Epoch Oracle Subgraph in the event of a Correct Blocks message.

> This ongoing operational requirement makes it clear that the Epoch Block Oracle interface should not be changed frequently, at least without proactive communication with the Indexer Components working group, so as to not break the Indexer Agent.

### Update the Subgraph Oracle

The Subgraph Oracle will need to check with the Epoch Block Oracle if there is an Epoch Block available for the network a subgraph is indexing. If there is, then the subgraph will be eligible for indexing rewards.

Alternatively, the network subgraph could be updated to also parse information from the Data Edge in identifying whether a subgraph is eligible for indexing rewards. This has the downside of splitting that responsibility between two separate components.

> Beyond the scope of this GIP, but the Subgraph Oracle could also be updated to use a Data Edge model.

### Update the Arbitration Charter

The Arbitration Charter will need to reflect the requirements for allocation closure - in order to be eligible for Indexing Rewards, an allocation must be closed with a POI for the block specified by the Epoch Block Oracle Subgraph, for the relevant epoch:network.

To allow for cases where allocations are being closed when the epoch is being updated, the existing N-1 policy should remain in place.

### Ecosystem application compatibility

There are ecosystem applications (for example the Graph Explorer and Subgraph Studio) which currently assume that only Mainnet subgraphs will be supported for indexing rewards, for example showing warnings or preventing certain actions for non-mainnet subgraphs. These should be updated as and when additional networks are added to the Epoch Block Oracle, as they will now be fully eligible.

### Governance process for adding new chains

Adding a chain to the block oracle is an economically meaningful action for the protocol, if it is to enable indexing rewards. There should therefore be a ratification process for the addition of new networks.

**This GIP proposes that The Graph Council should approve addition of new chains via a governance proposal.**

This approval should fall under the scope of GIP-0008 (Subgraph API versioning and feature support).

When assessing whether a given chain should be added, The Graph Council should require confirmation (from core developers, or otherwise) that an integration with Graph Node exists which allows for deterministic indexing of the chain. The Graph Council may also consider the chain itself, its ecosystem (for decentralisation, client availability, stability), and any Graph ecosystem preparation.

> To reiterate: a chain being eligible for indexing rewards is an indication that subgraphs indexing a given chain can be verified as part of The Graph Protocol. It is not a statement on indexer readiness to support subgraphs on that chain.

Once a network has been added to the feature support matrix:
- The chain should be added to [networks.md](https://github.com/graphprotocol/indexer/blob/main/docs/networks.md), including any relevant aliases to be used in subgraph manifests.
- The Epoch Block Oracle implementation should allow for easy fetching of the calldata required to add a new chain, which can then be prepared and executed. The Council should then sign and execute a transaction to Register a new network on the Data Edge via their Multisig, resulting in addition of that network to the subgraph.
- The Epoch Block Oracle & subgraph should be upgraded to support fetching blocks for the new chain. This may be a simple configuration change.

Once included in the Epoch Block subgraph, the Subgraph Oracle should then allow indexing rewards for all subgraphs indexing the newly supported network. This technical & economic support should then catalyse activity on the network, supported by ecosystem members and core developers.

# Future development

- The architecture & implementation may need extending as the protocol is redeployed on layer 2 ([link](https://forum.thegraph.com/t/gip-0034-the-graph-arbitrum-devnet-with-a-new-rewards-issuance-and-distribution-mechanism/3418))
- Increasing the committee of validators / proposers
    - This will improve the resiliency & trustworthiness of the oracle

- Migrating to a longer term solution for cross-chain Block data availability (see "Alternatives")

# Risks and Security Considerations

There are several failure modes to consider.

### The Owner fails to update the Epoch Block Oracle

In this case, indexers will be able to close, re-open and close allocations without doing any further indexing work.

The Owner's infrastructure should be redundant & resilient, with appropriate monitoring to avoid this situation. This failure mode can be resolved quickly, and if necessary by hand, so it is acceptable. As the Epoch Block Oracle is decentralised, the risk of this situation decreases significantly.

### It is not clear which branch is the canonical branch (in the case of a long fork)

In this case it will be unclear which fork can be used to use to close allocations.

The Owner will be able to quickly resolve the situation by calling the contract with a `CorrectEpochs` message, which will then disambiguate all parent epochs.

# Validation

Feedback from indexers & other ecosystem participants.

# Rationale and Alternatives

There is a broad solution-space here, and no perfect solutions. Within the constraints of an Oracle-based solution, we [explored a range of options](https://hackmd.io/LZP5yeKhTUey1ELXPxaZaA), outlined here.

The Oracle approach itself is proposed as the simplest way to support indexing on multiple chains, at the expense of decentralisation in the immediate term. Given upcoming plans to move more protocol logic to L2, a simpler, more portable implementation is particularly appealing. There is also a path to decentralising an oracle solution over time (and perhaps moving away from it, once an alternative is identified).

However it is worth mentioning some of the areas of investigation:

### Bridge-based

A few bridge-based designs have been investigated: leveraging traditional bridges (e.g. [Arbitrum](https://developer.offchainlabs.com/docs/l1_l2_messages), [Rainbow](https://near.org/bridge/)), or optimistic bridges (e.g. [Nomad protocol](https://docs.nomad.xyz/)). Cross-chain bridging is still [nascent](https://www.theverge.com/2022/2/3/22916111/wormhole-hack-github-error-325-million-theft-ethereum-solana), with a lot of design space to explore before we felt comfortable proposing a solution for the network to rely on.

### Remote allocation closure

Indexers could manage allocations on the "indexed" chain, removing the need for cross-chain communication to close allocations. This could be combined with an optimistic bridging approach, for disputing POIs. This would also make it possible to validate data freshness within a given chain.

This significantly increases the complexity of the network, requiring compatible deployments on all the chains we want to support on the network, so this is not feasible in the immediate term.

### Sync chain

A "sync" chain has been discussed, as it is relevant to other areas where synchronising time between chains is required. The central idea is to have a blockchain which tracks and validates corresponding blocks on different chains.

The future of the Block Epoch Oracle could indeed be a sync chain, as validators of the sync chain propose updates, with an elected validator updating the Oracle.

In the immediate term, establishing a new blockchain is a larger undertaking than creating an oracle

### P2P Consensus

The Indexers of The Graph Network have many reasons that they might want to interact, via "gossip" or otherwise. Given that they all have an on-chain stake commitment, it would also be possible to reach some consensus, across indexers. Indexers could similarly become the source of the Block Epoch Oracle's updates, increasing decentralisation and robustness.

### Submitting recent blocks for POIs

This has been [proposed](https://forum.thegraph.com/t/require-that-more-recent-pois-are-submitted-in-order-to-collect-indexing-rewards-multi-blockhain-pois/2500) as a means of achieving data freshness on the network. While this is possible for Ethereum Mainnet with the current implementation of Allocations, that becomes unviable as soon as we want to support other networks. If we chose to proceed with "Remove allocation closure" this would be possible, but in the short term this change would not have broad benefit, and would "special case" Ethereum Mainnet, which is not desirable. Ideally Query Rewards will also grow in importance for indexers, which will be another means to enforce data freshness (as indexers with fresher data will be preferred by consumers).

### On-chain Epoch Oracle

In addition, there was a design for an oracle solution where data was stored on-chain, instead of simply as calldata (with execution in the subgraph). This is described [here](https://hackmd.io/KHLuc4haR5uCnmpMu0Ce7Q) for completeness, but is significantly less gas efficient & flexible.

# Copyright Waiver

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

