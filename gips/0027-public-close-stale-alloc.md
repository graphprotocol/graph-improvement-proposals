---
GIP: "0027"
Title: Create altruistic allocations and close stale allocations
Authors: Ariel Barmat <ariel@thegraph.com>, Sam Green <sam@semiotic.ai>
Created: 2022-03-24
Updated: 2022-03-24
Stage: Proposal
Implementations: https://github.com/graphprotocol/contracts/pull/549
---

# Abstract
Provide a mechanism so that indexers can opt-out of indexing rewards by allocating zero tokens to a subgraph. This would be useful for an indexer who wants to altruistically serve queries. In addition, this implementation allows the public to close stale allocations after the max allocation period, however, nobody can close an altruistic allocation.

This GIP arose from the least contentious ideas in this forum discussion: https://forum.thegraph.com/t/rewarded-force-close-mechanism-to-eliminate-stale-allocations/2951/8.

# Background
An indexer's allocation toward a particular subgraph has a maximum allocation period of 28 epochs (i.e., `maxAllocationEpoch`), after which the indexer should close the allocation. If an indexer leaves an allocation open beyond 28 epochs, we call this allocation *stale*.

> The mechanism for closing state allocations after the `maxAllocationEpoch` was introduced to prevent indexers from withholding rewards and never distributing them to delegators. Additionally, it promotes liveness in the protocol state changes that affect other stakeholders' decisions.

An indexer's stale allocation is still eligible for indexing rewards. If an indexer were to close their stale allocation (at some point beyond epoch 28) with a non-zero Proof of Indexing (non-zero PoI indicates to the protocol that the Indexer did valid work). However, any delegator can close their indexer's stale allocation with a zero PoI which indicates to the protocol that it should not distribute indexing rewards for that allocation.

When an indexer's allocation is closed with a non-zero PoI, then the protocol will calculate the amount of indexing rewards owed to the indexer. The amount of indexing rewards owed to an indexer is proportional to the indexer's allocated stake relative to the total of all allocated stake on the same subgraph.

The two paragraphs above imply that if an "active" indexer closes an allocation that is less than 28 epochs old (i.e. it is not stale), but there is another indexer with a stale allocation currently open on the same subgraph, then the active indexer will receive less indexing rewards than they would have had the stale indexer not been allocated.

# Motivation

1. Stale allocations affect the stability of the network when an indexer that has a big proportional allocated stake on a subgraph does not keep it active, as other indexers won't want to allocate if it already has a big allocation. Stale allocations also reduce the speed of how value flows through the protocol, reducing the frequency at which curators and delegators get their share of rewards and fees. Currently, delegators of an indexer are the only ones who can close stale allocations, but this is an unnecessary validation considering that anyone could turn into a delegator, even with the smallest amount of tokens, and then close it. For a summary of the current stale allocation statistics, see the "Allocation Age" histogram here: https://dune.xyz/abarmat/The-Graph.

2. Today, indexers must allocate more than zero tokens. However, indexers might want to index and serve queries on a particular subgraph altruistically by opting out of rewards. Today indexers can serve queries altruistically by presenting an empty PoI on `closeAllocation`, however, this is not efficient because they must still pay Ethereum transaction fees to "refresh" (i.e., close and re-open) their allocations. By allowing altruistic indexers to allocate zero tokens, we can skip all the code paths for updating indexing rewards and provide a huge gas improvement.

# High-Level Description

This GIP proposes to make the following changes:

* Indexers can allocate zero-tokens to a subgraph to advertise they are indexing and serving queries but have opted out of indexing rewards. We call these *altruistic allocations*.
* Even if indexers allocate zero tokens, they are still required to have the minimum stake (currently 100,000 GRT).
* Altruistic allocations cannot be closed after the `maxAllocationEpoch` by the public - only by the indexer or the indexer's operator.
* Anyone can now close an indexer's allocation after the `maxAllocationEpoch`.

# Implementation
https://github.com/graphprotocol/contracts/pull/549

# Copyright Waiver
Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
