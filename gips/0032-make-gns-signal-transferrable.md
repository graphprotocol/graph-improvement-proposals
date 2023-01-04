---
GIP: 0032
Title: Make GNS signal transferrable
Authors: Ariel Barmat <ariel@edgeandnode.com>
Created: 2022-05-15
Updated: 2022-05-15
Stage: Candidate
Discussions-To: https://forum.thegraph.com/t/gip-0032-make-gns-signal-transferrable/3338
Implementations: TBD
---

# Abstract

The GNS contract allows anyone to publish a subgraph with associated metadata and a target subgraph deployment. Subsequently, curators can mint and receive shares (signal) of the subgraph by depositing GRT tokens. The issue lies in that the curators cannot transfer those shares to a different user, blocking several use cases. This proposal intends to solve those issues by introducing a transfer shares function to the GNS.

# Motivation

Curators can mint curation shares from a Subgraph Deployment directly on the Curation contract and get fully transferrable ERC20 shares. However, when they curate a Subgraph at the GNS level, the GNS contract is the one keeping all the signal shares under its control. The subgraph curators cannot transfer those shares because the GNS is not exposing any function to do that. There are a few use cases that are affected by this behavior:

- A developer that curated the initial signal and then wants to move it to a different address like a multisig.
- A concierge application that setups a subgraph and mints some signal, which eventually needs to delegate the ownership to a different beneficiary.
- A curator that needs to reorganize wallets.

The GNS currently supports delegating ownership of a subgraph by transferring the Subgraph-NFT, but it does not allow doing the same with the signal.

We consider this is a great addition to the core functionality of the GNS that will enable new use cases.

# Specification

Add a function to transfer the subgraph shares with the following signature: `transfer(uint256 _subgraphID, address _recipient, uint256 _amount) returns (bool)`

This function will reduce the caller's share balance from `mapping(address => uint256) curatorNSignal` and add it to the `_recipient`. Additionally, it must ensure that all relevant balance check are present in the same fashion as an ERC20 transfer.

This implementation relies on updating the internal balances in the GNS and it doesn't create an ERC20 token for each subgraph as we consider it would add a significant gas overhead. The proposal keeps ERC20 tokens to represent shares of a subgraph deployment at the Curation contract level.

In the future, an additional proposal could turn the current shares into ERC20. This feature might be more suitable on an L2.

# Implementation

This proposal doesn't have an implementation yet.

# Backwards Compatibility

This proposal is fully backward compatible.

# Validation

### Audits

The implementation has not yet been audited.

### Testnet

The implementation has not yet been deployed to Testnet.

# Copyright Waiver

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
