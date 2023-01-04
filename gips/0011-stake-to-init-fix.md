---
GIP: "0011"
Title: Delegation parameters initialization when used from stakeTo
Authors: Ariel Barmat <ariel@edgeandnode.com>
Created: 2021-06-14
Stage: Candidate
Implementations: https://github.com/graphprotocol/contracts/pull/457
---

# Abstract

The Staking contract has a function that allow indexers to set delegation parameters. Under certain circumstances this function will initialize the wrong address. The changes in this proposal fix the issue.

# Motivation

The function `setDelegationParameters()` in the Staking contract can be used to set the delegation parameters for when rewards are distributed. The Staking contract uses that same function to initialize the delegation parameters when an indexer stakes for the same time to 100% indexer cut.

Initialization can happen within the execution of two public functions: `stake()` and `stakeTo()`. The first will only be called by the indexer, while the second can also be called by third parties addresses depositing stake to the indexer.

The issue lies in that `setDelegationParameters()` always use _msg.sender_, this means that when using `stakeTo()` to initialize a delegation pool gives rise to the two-fold incorrect behavior:

The delegation parameters for indexer remain unset, while we expected them to be set. The delegation parameters for _msg.sender_ are unintentionally set.

# Specification

The solution is to have an internal `_setDelegationParameters()` that accepts the indexer address for which we are setting the configuration and use it from `stake()`, `stakeTo()`, and the public `setDelegationParameters()` functions.

# Implementation

See [@graphprotocol/contracts#457](https://github.com/graphprotocol/contracts/pull/457)

# Backwards Compatibility

The proposal is fully backwards compatible.

# Validation

### Audits

The implementation was audited by OpenZeppelin. Full public report: https://blog.openzeppelin.com/thegraph-staking-bugfix-audit-1/

### Testnet

The implementation has not yet been deployed to Testnet.

# Copyright Waiver

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
