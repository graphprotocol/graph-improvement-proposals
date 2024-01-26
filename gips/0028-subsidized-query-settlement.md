---
GIP: "0028"
Title: Subsidized Query Settlement
Authors: Ariel Barmat <ariel@edgeandnode.com>
Created: 2022-03-24
Stage: Candidate
Category: Protocol Logic
Implementations: https://github.com/graphprotocol/contracts/pull/557
---

# [GIP-0028] Subsidized Query Settlement

# Abstract

This proposal extends the indexer reward subsidy to take into account both indexing subgraphs as well as query fee collection to ensure we close the loop of the lifecycle of an allocation.

# Motivation

The network provides incentives to bootstrap the operation of indexers for new subgraphs which may not have pre-existing demand to attract indexers. The way it works is that each subgraph in the network is allotted a portion of the total network token issuance, based on the proportional amount of total curation signal that subgraph has. That amount, in turn, is divided between all the Indexers staked on that subgraph proportional to their amount of allocated stake.

Today, indexers collect indexing rewards when they close the allocation to a subgraph. Consequently, the subsidy only covers half of the allocation lifecycle, and indexers might choose never to collect query fees due to changing network conditions or any other strategy. By placing all rewards distribution at the end of the allocation lifecycle, we align the incentives to ensure that the network subsidy includes all the actions that indexers need to perform to end up with a lively network.

# Specification

### Allocation Lifecycle

The lifecycle of an Allocation starts by setting aside tokens from the global indexer stake into a subgraph and finishes when claiming the query fees from the rebate pool.

**Current:**

1. Create Allocation
2. Close Allocation -> **_distribute indexing rewards_**
3. Collect query fees through payment vouchers
4. Claim Allocation Rebate -> **_distribute query rewards_**

**Proposed:**

1. Create Allocation
2. Close Allocation
3. Collect query fees through payment vouchers
4. Claim Allocation Rebate -> **_distribute both indexing and query rewards_**

### Store Indexing Rewards

The main change is that indexing rewards are no longer distributed when indexers call `closeAllocation()` but it will be delayed to a later stage in the Allocation lifecycle. Rewards will still accrue until closeAllocation is called, but we will set aside those rewards when the indexer presents a non-zero POI.

To support this, we will add an `indexingRewards` state variable in the **Allocation** struct and initialize it to zero when indexers create the allocation in `allocateFrom()`.

Then, after at least one epoch has passed, the indexer will close the allocation, the protocol will use the `RewardsManager.takeRewards()` function to mint the accrued tokens to the Staking contract and get the amount of tokens created. We will store those tokens in the `Allocation.indexingRewards` state variable instead of sending them directly to the indexers & delegators.

### Distributing Indexing Rewards

The lifecycle of an Allocation ends when the `claim()` function is called. Claiming is a public function that anyone can call passing a valid AllocationID.

Any token distribution, both for query (from rebate pool) and indexing rewards will happen when claiming the Allocation.

Changes:

- After claiming from the rebate pool, calculate the totalRewards by checking if **_query rewards_** received are greater than zero, if that is the case we will let the indexer get their **indexing rewards**. If **_query rewards_** are zero the indexer will forfeit indexing rewards. The formula is `totalRewards = queryRewards + indexingRewards`

- Collect delegation rewards for both stored indexing rewards and queries coming out from the rebate pool. The delegation rewards are then sent into the delegation pool for the indexer that created the allocation.

- Reset the `indexingRewards` state variable in the Allocation struct.

- Send the indexer part of rewards to the indexer (both indexing + query rewards). The formula is `indexerRewards = totalRewards - delegationRewards`.

### Additional interface changes

The function to claim an allocation currently has an extra parameter to decide whether to restake the rewards or not. However, this is not necessary and in fact can be troublesome as anyone can call the public `claim()` function.

We propose to use the internal `_sendRewards()` function along with the existing **Rewards Destination** feature to decide whether to restake or not.

```
# old interface
function claim(address _allocationID, bool _restake) external override;

# new interface
function claim(address _allocationID) external override;
```

### Gas Considerations

- We are using 20,000 more gas to store the new `indexingRewards` variable in the Allocation struct during `closeAllocation()`, some of that is recovered when the variable is cleaned up during `claim()`.
- Doing all token transfers during `claim()` instead of having it split into two different calls is more efficient, saving around 10,000 gas.

### Special Considerations

- Indexers that haven't collected any query fees are not elegible for indexing rewards. Those rewards stored during close allocation will be burned after claiming.
- Claiming rebates requires waiting a seven-day period to ensure that all funds are collected properly from payment channnels, this has the side effect of delaying indexing rewards to when all the conditions are met.

# Implementation

See [@graphprotocol/contracts#557](https://github.com/graphprotocol/contracts/pull/557)

# Backwards Compatibility

If we remove the unused `_stake` from the `claim()` function that would require any agent using the contracts to update.

Once the upgrade to the new mechanism happens, all of the Allocation structs for created allocations will have the `indexingRewards` variable set to zero, which is convenient as we don't need to initialize them. Whenever indexers close any of those allocations the protocol will calculate the accrued rewards and fill that variable for use during `claim()`.

# Validation

### Audits

The implementation is not yet audited.

### Testnet

The implementation has not yet been deployed to Testnet.

# Copyright Waiver

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
