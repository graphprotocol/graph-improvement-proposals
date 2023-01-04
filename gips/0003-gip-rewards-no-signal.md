---
GIP: 0003
Title: Fix accumulated subgraph rewards being reset to zero
Authors: Ariel Barmat <ariel@edgeandnode.com>
Created: 2021-03-01
Stage: Candidate
Discussions-To: https://forum.thegraph.com/t/accumulated-subgraph-rewards-reset-to-zero-on-edge-case/1537
Implementations: https://github.com/graphprotocol/contracts/pull/452
---

# Abstract

This proposal resolves an issue that prevents indexing rewards from being collected in certain cases. The RewardsManager contract's main responsibility is to calculate rewards accrued for allocations based on the total token supply, tokens signaled on subgraphs, accumulated rewards on a subgraph and allocations. In a specific edge case, calculation could fail and block closing an allocation in a way that collects rewards.

# Motivation

There is a particular edge case that happens when there are running allocations on a subgraph deployment ID, the total amount of curation signal is removed back to zero and `closeAllocation()` is called by an Indexer. If this happens, the Indexer won't be able to close that allocation unless it is opting for no rewards by sending POI=0x0. The reason for this is that `getAccRewardsForSubgraph()` is reseting the accumulated rewards for a subgraph when the curation pool is empty. The consequence is that `subgraph.accRewardsForSubgraphSnapshot` can be larger than `subgraph.accRewardsForSubgraph` leading to a revert when calculating new accrued rewards.

# Detailed Specification

## Proposed Changes

By removing the early check within `getAccRewardsForSubgraph()` that returns zero for the subgraph accumulated rewards when the signal is zero we will always return the accumulated amount, as it should be an ever increasing number.

#### 1) Update the calculation for accumulated rewards of a subgraph.

`function getAccRewardsForSubgraph(bytes32 _subgraphDeploymentID)`

**Remove early exit returning zero when no signalled tokens:**
```
uint256 subgraphSignalledTokens = curation().getCurationPoolTokens(_subgraphDeploymentID);
if (subgraphSignalledTokens == 0) {
  return 0;
}
uint256 newRewards = newRewardsPerSignal.mul(subgraphSignalledTokens).div(TOKEN_DECIMALS);
return subgraph.accRewardsForSubgraph.add(newRewards);
```

Removing that check will result in always keeping the accumulated rewards for the subgraph. In the cases when subgraphSignalledTokens = 0, no new rewards will be accrued per the below calculations.

**New implementation:**
```
uint256 subgraphSignalledTokens = curation().getCurationPoolTokens(_subgraphDeploymentID);
uint256 newRewards = newRewardsPerSignal.mul(subgraphSignalledTokens).div(TOKEN_DECIMALS);
return subgraph.accRewardsForSubgraph.add(newRewards);
```

# Validation

## Tests

Added a test to reproduce the condition that fails to pass with the older implementation and works with the new one.

https://github.com/graphprotocol/contracts/pull/452/files#diff-9a2435f8c93e0efee7cee8b77fdc92daa626cc3058366c9e021d1dce33dcc389R687


## Audits

An audit for this change is pending.

# Copyright Waiver

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
