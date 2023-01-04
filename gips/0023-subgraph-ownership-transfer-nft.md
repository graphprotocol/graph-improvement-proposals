---
GIP: "0023"
Title: Subgraph ownership transfer using external NFT contract
Authors: Ariel Barmat <ariel@edgeandnode.com>
Created: 2022-01-16
Stage: Candidate
Discussions-To: https://forum.thegraph.com/t/gip-0018-subgraph-ownership-transfer/2589
Replaces: "0018"
Implementations: https://github.com/graphprotocol/contracts/pull/527
Audits: https://github.com/graphprotocol/contracts/blob/dev/audits/ConsenSysDiligence/2022-01-graph-pr527-audit.pdf
---

# Abstract

The GNS contract allows anyone to publish a subgraph with associated metadata and a target subgraph deployment. The new Subgraph will then be tied forever to the account that created it. The impossibility of transferring the Subgraph ownership makes some use cases very inconvenient, such as moving your control to a multisig or a community member creating it on behalf of a DAO, etc. This proposal intend to solve those issues.

This GIP supersedes GIP-0018 maintaining the same goal but refactoring the implementation. All details about the changes are under the Specification section. Entire sections are copied over from the original GIP and updated to avoid misinterpretation.

# Motivation

App developers create subgraphs to index blockchain data. Then, they want indexers to run their subgraphs in the decentralized network. To achieve that, they publish a Subgraph in the GNS that targets a Subgraph Deployment. Once they publish a subgraph, they work to attract curators that delegate signal to the Subgraph so that the app developer can properly incentivize indexers.

For many reasons, the app developer might want to transfer ownership of the subgraph to a different account which is not possible with the current implementation.

# Specification

The proposed implementation involves a number of changes to make it easier to manage a subgraph. In addition to that, it simplifies many of the contract interfaces.

### Subgraph Primary Key

The current primary key that defines a subgraph is a combination of `graphAccount` and `subgraphNumber` that gets passed to every function used to interact with the GNS.

To make it easier to store and reference, we will change that for a single `subgraphID`. A `subgraphID` will be determined at the moment of publication, and calculated like `KECCAK(creatorAccount, seqID)`.

- `creatorAccount` is the `msg.sender` that calls `publishNewSubgraph`
- `seqID` is a sequential ID that gets incremented for that creator to avoid collisions

### NFT-based Ownership

Whenever an app developer publishes a new subgraph, the GNS will mint an NFT. Whoever owns the NFT controls the subgraph. The NFT is based on a standard ERC721, so it can be easily transferred to different accounts with no limitation. Additionally, the NFT is burned when the owner deprecates the subgraph.

All access control for the following functions will be NFT-owner based:

- updateSubgraphMetadata()
- publishNewVersion()
- deprecateSubgraph()

#### Differences with GIP-0018

The first implementation of the NFT-subgraph inherited the ERC721 behaviour from the GNS, and as a result, we could use the GNS contract as the registry. This brought a number of issues, the main one was that OpenSea, Etherscan and other apps would not detect the upgraded GNS as a valid ERC-721 NFT. The reason for that is that they scan for newly deployed contracts and "mark" them as non-fungible tokens. In addition to that, inheriting the full ERC721 code into the GNS ended up in bloated code in a single contract, getting close to the contract bytecode limit.

The new implementation proposed in this GIP uses a different NFT contract deployed separately from the GNS and make them work through composability.

#### Architecture

`GNS ->(minter)->SubgraphNFT->(tokenURI)->SubgraphNFTDescriptor`

To support this functionality we introduce two contracts:

- **SubgraphNFT**: This is a standard ERC721 contract based from OpenZeppelin implementation. This contract uses a _TokenDescriptor_ to render the tokenURI. The TokenDescriptor can be swapped by governance to provide future-proofing. The SubgraphNFT allows to set a special role called the `minter` that is the only one that can mint, burn or set the NFT metadata. The minter, in our setup, is the GNS.

- **SubgraphNFTDescriptor**: This is a contract that implements the TokenDescriptor interface and whose only purpose is to render the tokenURI.

And we do the following changes to the **GNS**:

- The GNS has an additional state variable that stores the SubgraphNFT address, so that whenever an app developer interacts with a subgraph, the GNS can mint, burn or check ownership of the subgraph through the NFT.
- We leave the door open for Governance to swap the SubgraphNFT used by the GNS, but this should only be used on an emergency case and requires state migration.

#### Metadata

The subgraph metadata is an IPFS-hash that contains a JSON file that encode relevant information about the subgraph, like an image, display name, category, etc.

The subgraph metadata was originally just emitted on the `SubgraphMetadataUpdated` event whenever a subgraph was published or the app developer decided to update it.

This GIP propose to store the subgraph metadata (IPFS-hash) into a state variable in the SubgraphNFT. This way the NFT can then render the proper tokenURI out of it and be visible on wallets and any other NFT Marketplace.

A TokenDescriptor contract will be provided to convert the IPFS-hash format stored as a bytes32 into a compatible base58 string that IPFS uses in client URIs.

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
- It is recommended to warn any known dapp integrated with the contracts to ensure they update the interfaces.

Upgrade activities:

1. On the day of the upgrade the team needs to perform the following steps:

- Deploy the new GNS implementation
- Deploy the SubgraphNFTDescriptor
- Deploy the SubgraphNFT
- Verify all contracts on Etherscan
- SubgraphNFT.setMinter(GNSProxy.address)
- SubgraphNFT.setTokenDescriptor(SubgraphNFTDescriptor.address)
- SubgraphNFT.transferOwnership(Council)

2. The Council will then perform the following tasks if they agree on the upgrade:

- Prepare the correct initializer parameters
- Upgrade GNS setting the SubgraphNFT.address in the initializer when sending the `acceptProxyAndCall` transaction

3. The team will then call `migrateLegacy()` function for each subgraph to mint the NFT on behalf of every user.

# Implementation

See [@graphprotocol/contracts#497](https://github.com/graphprotocol/contracts/pull/497)
See [@graphprotocol/contracts#527](https://github.com/graphprotocol/contracts/pull/527)

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

The original implementation under GIP-0018 has been audited by OpenZeppelin ([Report](https://github.com/graphprotocol/contracts/blob/4ec06e9357c948829e72c3a4c9ed8bfc1d0d04ba/audits/OpenZeppelin/2021-11-graph-gns-transferrable-owner.pdf)) and Consensys Diligence.

The refactored implementation related to this GIP was audited by Consensys Dilligence (PR-527 : [Report](https://github.com/graphprotocol/contracts/blob/cdac3a2d155522d04fc28a1f696128a426c40fd9/audits/ConsenSysDiligence/2022-01-graph-pr527-audit.pdf))

All the audit reports can be found under the `/audit` folder in https://github.com/graphprotocol/contracts.

Findings were fixed in the following commits:

- (3.1) -> `0a3888d572cd2faab0f9b32477fe237f0cd5856e`
- (3.2) -> `ceab72985b923c44dc2aceaede7523861142790f`

### Testnet

The implementation has been deployed to Rinkeby multiple times. The team has practiced the upgrade and migration process using the following NPM packages.

**Deployments of the newer version of the protocol:**

```
- @graphprotocol/contracts/v/1.10.2-testnet-nft
- @graphprotocol/contracts/v/1.10.1-testnet-nft
- @graphprotocol/contracts/v/1.10.0-testnet-nft
```

**Deployments of mainnet version of contracts then upgraded into the new ones:**

```
- @graphprotocol/contracts/v/1.8.3-testnet-nft
- @graphprotocol/contracts/v/1.8.2-testnet-nft
- @graphprotocol/contracts/v/1.8.1-testnet-nft
- @graphprotocol/contracts/v/1.8.0-testnet-nft
```

# Copyright Waiver

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
