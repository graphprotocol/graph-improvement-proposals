---
GIP: "0019"
Title: Save gas when initializing a subgraph deployment
Authors: Ariel Barmat <ariel@edgeandnode.com>
Created: 2021-09-25
Stage: Candidate
Implementations: https://github.com/graphprotocol/contracts/pull/505
---

# Abstract

A subgraph-specific ERC20 token represents the signal on a subgraph deployment. When a curator mints signal for the first time, the Curation contract, among other things, deploys a new ERC20 token as part of the initialization process. This process is quite costly at about 1.2M gas. This proposal describes an update to make this protocol action much more efficient.

# Motivation

Reduce the gas cost for anyone initializing a subgraph deployment. This update is particularly beneficial for subgraph/app developers who do the first signal.

# Specification

Use a Minimal Proxy to clone the Graph Curation Token (ERC20) based on an implementation contract deployed just once. The implementation bytecode is also called master copy.

The benefit of this solution is that the token interface remain the same, making it backwards compatible.

**This solution involves the following changes:**

- Modify the initializer of Curation to accept the master copy contract address to use when cloning.
- Expose a function in Curation contract so that governance can change the implementation in the future.
- Leverage "@openzeppelin/contracts/proxy/Clones.sol" to spawn the Minimal Proxies.
- Update the current Graph Curation Token contract code and make it upgradeable so it can be used from the context of a Minimal Proxy.

**Additional improvements:**

- Avoid re-deployment of the subgraph ERC20 token when all the signal is burned back to zero.

**Important notes:**

- Previously created ERC20 tokens for subgraph deployments will not be changed.
- The implementation used as template for already deployed clones cannot be changed, it can only change for future ones.

Based on gas consumption reports, using a Minimal Proxy reduces the gas used on first mint from ~1,230,000 gas to ~432,000

## Operational Considerations

Performing this upgrade involves:

- Deploying the Graph Curation Token master copy and getting the address before doing the upgrade.
- When doing the upgrade of Curation contract we need to pass the master copy address to use as implementation for every clone.

# Implementation

See [@graphprotocol/contracts#505](https://github.com/graphprotocol/contracts/pull/505)

# Backwards Compatibility

The proposal is fully backward compatible.

# Validation

### Audits

The implementation has not yet been audited.

### Testnet

The implementation has not yet been deployed to Testnet.

# Copyright Waiver

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
