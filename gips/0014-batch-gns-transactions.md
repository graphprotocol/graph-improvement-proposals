---
GIP: "0014"
Title: Batch GNS transactions
Authors: Ariel Barmat <ariel@edgeandnode.com>
Created: 2021-07-19
Stage: Candidate
Implementations: https://github.com/graphprotocol/contracts/pull/485
---

# Abstract

The GNS is a contract the allows creating entities called Subgraphs that allows an app developer to get signal delegated by curators and incentive their subgraph deployments. The process of publishing a subgraph and minting signal means the app developer needs to send two different transactions at the risk of frontrunning. This proposal adds the multicall pattern to the GNS so that an app developer can combine publishing with the first minting of signal. An additional benefit of this implementation is that it allows any combination of batching calls.

# Motivation

One of the issues brought by the community is that sometimes a subgraph publisher would want to publish a new subgraph and deposit the initial tokens. Today, that's only possible by using a Multisig or any other contract to batch those transactions.

This proposal allows batching transactions on the GNS, based on the Multicall pattern seen in Uniswap (https://github.com/Uniswap/uniswap-v3-periphery/blob/main/contracts/base/Multicall.sol) and recently implemented in OpenZeppelin (https://docs.openzeppelin.com/contracts/4.x/api/utils#Multicall)

# Specification

A new contract called **MultiCall** is introduced, inspired by the one used by Uniswap. The `payable` keyword was removed from the `multicall()` as the protocol does not deal with ETH. Additionally, it is insecure in some instances if the contract relies on `msg.value`.

The **GNS** inherits from MultiCall that expose a public `multicall(bytes[] calldata data)` function that receives an array of payloads to send to the contract itself. This allows to batch ANY publicly callable contract function.
Client-side one can build such payloads like:

```
// Build payloads
const tx1 = await gns.populateTransaction.publishNewSubgraph(
    <subgraphAccount>,
    <subgraphDeploymentID>,
    <versionMetadata>,
    <subgraphMetadata>,
)
const tx2 = await gns.populateTransaction.mintNSignal(
    <subgraphAccount>,
    <subgraphNumber>,
    <amount>,
    <signalOut>,
)

// Send batch
await gns.multicall([tx1.data, tx2.data])
```

# Implementation

The changes are implemented in the following PR: https://github.com/graphprotocol/contracts/pull/485

# Validation

### Audits

The implementation was audited by OpenZeppelin. Full report: [Batch GNS Transactions Audit](https://github.com/graphprotocol/contracts/blob/ec7f51a9726d15f6ff2e9e64204bdf825d045de6/audits/OpenZeppelin/2021-08-graph-gns-audit.pdf)

### Testnet

The implementation has been deployed to Testnet.

# Copyright Waiver

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
