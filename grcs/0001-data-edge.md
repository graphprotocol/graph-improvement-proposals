---
GRC: 0001
Title: DataEdge
Authors: Zac Burns <zac@edgeandnode.com>
Created: 2022-03-08
Updated: 2022-03-18
Stage: Proposal
Discussions-To: https://forum.thegraph.com/t/gip-0025-dataedge/3161
---

# Abstract

This GIP introduces DataEdge: a gas-efficient method to bridge data into subgraphs.

As L1s become increasingly costly, developers seek to minimize those costs. Rising costs are not just a hypothetical concern but an existential threat to the feasibility of blockchain for many use-cases. DataEdge solves this problem by reducing L1 gas costs to their theoretical lower limit.

We will first introduce DataEdge as a general concept and then propose a specific instantiation of the DataEdge to be used by the protocol. Flagship use cases within The Graph Protocol will include the Cross-Chain Epoch Oracle and the Query Version Registry.

# High Level Description

The DataEdge smart contract has a minimal interface containing only an empty `fallback()` method. A message comprises a selector and payload bytes from a call to the contract and can take arbitrary meaning defined by a subgraph that decodes the message.

A DataEdge contract will be deployed and designated by The Graph Council as the DataEdge for all protocol interfaces for use in The Graph Protocol.

# Detailed Specification

In terms of specification, there is no more to add to the DataEdge. An empty contract that sends payloads into the void is self-explanatory, even if befuddling. It may be helpful in the remainder of this GIP to establish norms and design patterns to inspire users of DataEdge on how to make the best use of such a vague tool.

## Subgraph ABI

Technically, the 4 bytes in the selector and any payload bytes can be taken as a whole message for maximum efficiency. However, if the selector identifies a method signature, graph-node can have a descriptive ABI in the subgraph manifest, and smart contract authors can call methods with descriptive names. The price of the descriptive manifest and developer ease is less than 100 gas per transaction.

The Graph Protocol's instantiation of DataEdge will use the selector as a namespace. For example, the `crossChainEpochOracle(bytes _payload)` and `queryVersionRegistry(bytes _payload)` methods would each correspond to selectors with their own payload encoding. This namespacing simplifies the development of the protocol by allowing individual subsystem's encodings to evolve independently.

## Compression

DataEdge message decoder implementations can rely on the fact that transactions cannot be re-ordered on a per-account basis. Even in a long reorg, the transaction nonce prevents such a re-ordering. Therefore the encoding can use stateful compression techniques, so long as the only data dependencies are the previous transactions from a given account.

Consider an example from the Cross-Chain Epoch Oracle. This oracle's job is to provide a list of block numbers from foreign chains on a recurring interval. Regular block numbers are a classic example of time-series data and lend themselves to stateful compression. Taking the previous block numbers from each foreign chain as state (derived only from previous transactions from the oracle's account), delta-of-delta encoding reduces the amortized block number size down to less than 2 bytes per entry.

## Batching

For most cases, most of the transaction cost will be the base fee and other overhead, rather than the payload itself. DataEdge implementations should support batching by concatenating multiple payloads to lower the amortized cost.
# Copyright Waiver

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).