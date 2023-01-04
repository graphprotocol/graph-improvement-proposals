---
GIP: "0010"
Title: Update rewards snapshot when close allocation with empty POI
Authors: Ariel Barmat <ariel@edgeandnode.com>
Created: 2021-06-15
Stage: Candidate
Implementations: https://github.com/graphprotocol/contracts/pull/459
---

# Abstract

Indexers get rewards for indexing subgraphs. These rewards are minted whenever an indexer close an allocation with a non-zero POI. A function called distributeRewards() within the Staking contract is used for that purpose. This function calls `RewardsManager.takeRewards()` that updates the snapshots used for rewards calculations and then mint the tokens. In some cases the snapshot calculation is not called.

# Motivation

When an indexer calls `closeAllocation()` with a POI=0x0 the function `distributeRewards()` is not called, as a consequence, `takeRewards()` is not called and in turn the snapshots are not updated before unallocating tokens. This creates a situation where the rewards manager considers there are a different amount of allocated tokens and distort calculations until it is called next time. Hopefully the distortion self corrects when another action is performed that updates the snapshot.

# Specification

Update the Staking contract `closeAllocation()` function to call `_updateRewards()` when the POI presented is 0x0 before unallocating the tokens.

# Implementation

See [@graphprotocol/contracts#459](https://github.com/graphprotocol/contracts/pull/459)

# Backwards Compatibility

The proposal is fully backwards compatible.

# Validation

### Audits

The implementation was audited by OpenZeppelin. Full public report: https://blog.openzeppelin.com/thegraph-staking-bugfix-audit-2/

### Testnet

The implementation has not yet been deployed to Testnet.

# Copyright Waiver

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
