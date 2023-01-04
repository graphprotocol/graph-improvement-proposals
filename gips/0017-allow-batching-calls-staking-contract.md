---
GIP: "0017"
Title: Allow batching calls in Staking contract
Authors: Ariel Barmat <ariel@edgeandnode.com>
Created: 2021-09-25
Stage: Candidate
Implementations: https://github.com/graphprotocol/contracts/pull/475
---

# Abstract

Expose a way to batch multiple calls into a single transaction. It provides great flexibility for indexer agents to combine multiple functions in different ways. It also reduce the gas cost by saving the initial gas, and in some cases, accessing a "used" slot by the other bundled transactions.

# Specification

A new contract called MultiCall is introduced, inspired by the one used by Uniswap. The payable keyword was removed from the `multicall()` as the protocol does not deal with ETH. Additionally, it is insecure in some instances if the contract relies on msg.value.

The Staking contract inherits from MultiCall that expose a public `multicall(bytes[] calldata data)` function that receives an array of payloads to send to the contract itself. This allows to batch ANY publicly callable contract function.

By using a multicall a user can batch an arbitrary number of operations into a single transaction. The indexer agent can combine close, open, claim transactions in any way it sees more effective.

# Implementation

See [@graphprotocol/contracts#475](https://github.com/graphprotocol/contracts/pull/475)

# Backwards Compatibility

The proposal introduces some breaking changes to save bytecode.

The following functions are removed as they can be constructed using a
multicall:

- closeAllocationMany()
- closeAndAllocate()
- claimMany()

# Validation

### Audits

The implementation was audited by Consensys Diligence.

### Testnet

The implementation has not yet been deployed to Testnet.

# Copyright Waiver

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
