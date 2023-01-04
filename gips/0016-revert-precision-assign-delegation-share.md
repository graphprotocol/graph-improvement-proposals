---
GIP: "0016"
Title: Revert when not enough precision to assign a delegation share
Authors: Ariel Barmat <ariel@edgeandnode.com>
Created: 2021-09-25
Stage: Candidate
Implementations: https://github.com/graphprotocol/contracts/pull/491
---

# Abstract

A delegation pool holds tokens deposited by delegators as well as rewards. When rewards start accruing, delegating a _very-very_ small amount of new tokens by a delegator could lead to zero shares being assigned even if the tokens deposited were accepted.

# Specification

Revert in any condition when no shares are assigned by adding a check in the `delegate()` function right after the amount of shares is calculated for the amount of tokens received.

# Implementation

See [@graphprotocol/contracts#491](https://github.com/graphprotocol/contracts/pull/491)

# Backwards Compatibility

The proposal is fully backwards compatible.

# Validation

### Audits

The implementation was audited by Consensys Diligence.

### Testnet

The implementation has not yet been deployed to Testnet.

# Copyright Waiver

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
