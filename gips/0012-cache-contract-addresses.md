---
GIP: "0012"
Title: Reduced gas usage by caching address resolution
Authors: Ariel Barmat <ariel@edgeandnode.com>
Created: 2021-06-14
Stage: Candidate
Implementations: https://github.com/graphprotocol/contracts/pull/430
---

# Abstract

Improve the gas cost of fetching network contract addresses by caching them locally in each contract.

# Motivation

The Controller contract works as a single registry for all the network contracts addresses. Whenever a contract needs to call another one it will fetch the address from the Controller.

Having a Controller contract is great in that we can sync addresses across the protocol from a single place, but it involves a roundtrip of CALL opcodes to get the addresses each time they are used.

An improvement to this is to have a state within each contract to cache the list of contract addresses and refresh it when the Controller is changed.

Gas saving are approximately 5% to 15% gas depending on the transaction.

# Specification

Any contract that uses the Controller inherits the **Managed** contract. This contract expose a number of function to fetch other contracts addresses.

- Use the reserved `addressCache` state variable to store the local contract addresses.
- Add a `_resolveContract()` function to test if the local variable is set, if not, use the default CALL opcode and fetch it from the remote Controller contract.
- Change the contract getters to always use `_resolveContract()`.
- Add a public `syncAllContracts()` that will refresh the locally-cached addresses using the latest addresses registered in the Controller.
- Emit the `ContractSynced(bytes32 indexed nameHash, address contractAddress)` event whenever a contract is synced.

# Implementation

See [@graphprotocol/contracts#430](https://github.com/graphprotocol/contracts/pull/430)

# Backwards Compatibility

The feature as implementated in this proposal is fully backwards compatible. By default, once upgraded, the contracts will work with no changes. To enable the feature, an Ethereum account will need to call at least once the `syncAllContracts()` function on each contract.

# Validation

### Audits

The implementation was audited by OpenZeppelin. Full public report: https://blog.openzeppelin.com/thegraph-addresses-cache-audit/

### Testnet

The implementation has not yet been deployed to Testnet.

# Copyright Waiver

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
