---
GIP: "0018"
Title: Subgraph ownership transfer
Authors: Ariel Barmat <ariel@edgeandnode.com>
Created: 2021-09-25
Stage: Candidate
Implementations: https://github.com/graphprotocol/contracts/pull/497
---

# Abstract

The GNS contract allows anyone to publish a subgraph with associated metadata and a target subgraph deployment. The new Subgraph will then be tied forever to the account that created it. The impossibility of transferring the Subgraph ownership makes some use cases very inconvenient, such as moving your control to a multisig or a community member creating it on behalf of a DAO, etc. This proposal intend to solve those issues.

# Motivation

App developers create subgraphs to index blockchain data. Then, they want indexers to run their subgraphs in the decentralized network. To achieve that, they publish a Subgraph in the GNS that targets a Subgraph Deployment. Once they publish a subgraph, they work to attract curators that delegate signal to the Subgraph so that the app developer can properly incentivize indexers.

For many reasons, the app developer might want to transfer ownership of the subgraph to a different account, a valid use case, not possible with the current implementation.

# Specification

The proposed implementation involves a number of changes to make it easier to manage a subgraph. In addition to that, it simplifies many of the contract interfaces.

### Subgraph Primary Key

The current primary key that defines a subgraph is a combination of `graphAccount` and `subgraphNumber` that gets passed to every function used to interact with the GNS.

To make it easier to store and reference, we will change that for a single `subgraphID`. A `subgraphID` will be determined at the moment of publication, and calculated like `KECCAK(creatorAccount, seqID)`.

- `creatorAccount` is the `msg.sender` that calls `publishNewSubgraph`
- `seqID` is a sequential ID that gets incremented for that creator to avoid collisions

### NFT-based Ownership

Whenever an app developer publishes a new subgraph, the GNS will mint an NFT. Whoever owns the NFT controls the subgraph. The NFT is based on a standard ERC721, so it can be easily transferred to different accounts.

All access control for the following functions will be NFT-owner based:

- updateSubgraphMetadata()
- publishNewVersion()
- deprecateSubgraph()

### Interface Simplification

All functions and events where `graphAccount` and `subgraphNumber` were passed will be changed for just a single `subgraphID`. This is a breaking change for clients that will need to adapt to the new interface. The fact that the new subgraphID is the `keccak(graphAccount, subgraphNumber)` makes it easier to translate old IDs into new ones.

### Remove Unused Functionality

The GNS included a feature to transfer the account identity that is not currently used and should be removed. For that it used the EthereumDIDRegistry contract.

### Migration Facilities

The new implementation will expose a function that owners of old-type subgraphs can call to mint their NFTs. This function must ensure that it is only called once per old-type subgraph.

Additionally, the contract will keep track a mapping of `subgraphID => (graphAccount, subgraphNumber)` for old subgraphs to make them backward compatible.

## Operational Considerations

Performing this upgrade involves:

- Migration of old subgraph types and minting of NFTs by calling a function exposed by the contract.
- Any frontend that integrates GNS functionality needs to start using the single subgraphID.
- Update the Core Network Subgraph to read the new events emitted by the contract.

# Implementation

See [@graphprotocol/contracts#497](https://github.com/graphprotocol/contracts/pull/497)

# Backwards Compatibility

The proposal has a number of breaking changes.

### Function Interfaces

Functions that change from taking `(graphAccount,subgraphNumber)` to `subgraphID`:

```
- updateSubgraphMetadata
- publishNewSubgraph
- deprecateSubgraph
- mintNSignal -> mintSignal
- burnNSignal -> burnSignal
- withdraw
- tokensToNSignal
- nSignalToTokens
- vSignalToNSignal
- nSignalToVSignal
- getCuratorNSignal -> getCuratorSignal
- isPublished
```

### Events

#### Updated Events

- SubgraphMetadataUpdated
- SubgraphDeprecated
- GRTWithdrawn

#### Deprecated Events

- SubgraphPublished
- NameSignalEnabled
- NSignalMinted
- NSignalBurned
- NameSignalUpgrade
- NameSignalDisabled

#### New Events

- SubgraphCreated
- SubgraphUpgraded
- SubgraphVersionUpdated
- LegacySubgraphClaimed
- SignalMinted
- SignalBurned

# Validation

### Audits

The implementation has not yet been audited.

### Testnet

The implementation has not yet been deployed to Testnet.

# Copyright Waiver

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
