---
GIP: 0004
Title: Withdraw indexing rewards to beneficiary address with thawing period.
Authors: Ariel Barmat <ariel@edgeandnode.com>
Created: 2021-03-23
Stage: Candidate
Discussions-To: https://forum.thegraph.com/t/rewards-destination-for-indexers-indexing-rewards/1279
Implementations: https://github.com/graphprotocol/contracts/pull/456
Replaces: GIP-0002
---

# Abstract

This proposal describes a mechanism for Indexers to withdraw rewards to a designated address instead of automatically re-staking these rewards in the protocol. The effect is that Indexers that interact with the protocol via the widely used TokenLock vesting contracts will be able to withdraw indexing rewards to cover operational costs, without fully unstaking from the protocol.

This proposal replaces a previous proposal that addresses the same root problem but had the side effect of removing a thawing period that Indexers currently have to wait for in order to withdraw indexing rewards.


# Motivation

As part of The Graph's "Mission Control" testnet, Indexers participated in learning, testing and improving of the protocol, for nearly half a year at their own expense. Participating effectively in the testnet involved a significant investment DevOps labor and infrastructure resources for the duration of the testnet.

At the conclusion of the testnet Indexers were given an allocation of GRT for the purpose of participating in the protocol. This GRT is held in a TokenLock vesting contract that allows for participation in the protocol, but only allows withdrawing tokens that are in excess of the remaining locked amount.

Therefore, any income earned from indexing rewards and query fees may only be fully withdrawn by completely exiting the protocol into the TokenLock contract so that it recognizes the excess tokens. This is undesirable for a couple reasons:
- Indexers suffer a significant opportunity cost for the 28 day unbonding period required to unstake as an Indexer.
- An Indexer exiting the protocol deprives their Delegators of rewards and thus hurts their reputation with Delegators.

This problem is particularly salient for smaller, independent Indexers who are not as well-capitalized as larger professional node operating services. These independent Indexers are vital for maintaining a diverse and decentralized network, and represent some of the most active participants in the community. They have been making significant contributions to the community for over half a year at this point and need to be able to access their query fees and indexing rewards to cover their expenses and to continue contributing at a high level.

This proposal allows Indexers to withdraw indexing rewards to a separate address, rather than having those rewards automatically re-staked in the protocol.

# Detailed Specification

## Current Behavior

By default, every time `closeAllocation()` is called by an Indexer presenting a Proof of Indexing (POI), the `getRewards()` function of the **RewardsManager** is used to calculate the amount of tokens to distribute for that allocation. Those rewards are minted, re-staked and the Delegators' commission deposited into the Indexer's delegation pool.

## Proposed Mechanism

An Indexer can avoid staking rewards by default and send them to an indexer-dedicated rewards pool by passing *restake=false*, a parameter included in both `closeAllocation()` and `claim()`. Any new tokens deposited in the rewards pool will be subject to a thawing period for withdrawal. Indexers can only withdraw them after the locking period passed by calling `withdrawRewards(address _beneficiary)`. The beneficiary parameter allows them to send the tokens a different address than the one that staked. Additionally, when depositing multiple times into a rewards pool, the new locking period for all the accumulated tokens will be adjusted based on the stake-weighted average of the combined thawing periods, reusing the same function as for `unstake()`.

The locking period for rewards will use the same governance parameter than for indexer stake called `thawingPeriod`.

## Proposed Changes

#### 1) Introduce a RewardsPool state variable to collect  indexing rewards and query fees from the rebate pool

```
    struct Pool {
        uint256 tokensLocked;
        uint256 tokensLockedUntil; // Block when locked tokens can be withdrawn
    }

    mapping(address => Rewards.Pool) public rewardsPools;
```

The rewards pool allows collecting both indexing rewards and query fees while using a thawing period for withdrawals.

#### 2) Introduce a parameter to closeAllocation() to decide whether to stake rewards or send them to the rewards pool


`function closeAllocation(address _allocationID, bytes32 _poi, bool _restake)`

- If *_restake* is set to false, the accrued rewards will be deposited into rewards pool under thawing period.
- If *_restake* is set to true, rewards will be staked an be available to use for allocations.

#### 3) Update rewards distribution function

`function _distributeRewards(address _allocationID, address _indexer, bool _restake)`

**Current implementation:**

This function currently stakes *indexerRewards* by default if any is present.

```
// Add the rest of the rewards to the Indexer stake
if (indexerRewards > 0) {
    // TODO: << check if rewards destination is set to stake or send to address >>
    _stake(_indexer, indexerRewards);
}
```

**New implementation:**

In the proposed implementation, check the passed *_restake* parameter to stake or deposit the tokens into the rewards pool.

```
// Send the indexing rewards to the indexer
_sendRewards(indexerRewards, _indexer, _restake);
```
The `_sendRewards()` private function encapsulates the staking or sending of rewards to the rewards pool.

This function is used both in `closeAllocation()` and `claim()`.

#### 4) Similar behaviour for indexing rewards and claiming from the rebate pool.

`function claim(address _allocationID, bool _restake)`

Query fees that are collected from the rebate pool are staked if *_restake = true* and sent to the rewards pool if *_restake = false*.

#### 5) Introduce a function to withdraw rewards from the rewards pool

`function withdrawRewards(address _beneficiary)`

- This function will use a thawing period to deliver accumulated rewards.
- This function accepts a beneficiary address to uses as destination of rewards.

### Main Benefits

- By allowing an Indexer to call `withdrawRewards()`  passing a beneficiary address, we can avoid withdrawals to a TokenLock contract that would otherwise lock the funds.
- Setting whether to stake rewards or send them to the rewards pool on the `closeAllocation()` and `claim()` functions, provides extreme flexibility to Indexers to update to their best strategy at no gas cost.
- Adding a thawing period to the rewards pool removes any short term speculation that could lead Indexers to close multiple allocations in a short period of time, and that could end up affecting the network health.
- All tokens leaving the Staking contract, for indexer stake, delegation and rewards go through thawing periods.
- The feature will only work for rewards collected after its implementation, past rewards remain staked.
- Both closeAllocation (for indexing rewards) and claim (for rebate rewards) have the same interface to decide if staking or not.

# Backwards Compatibility

The current solution changes the interface to `closeAllocation()` adding a new boolean parameter. The indexer agent software will need to be updated to use the new interface and pass the right value depending on the indexer's selected strategy.

# Validation

## Audits

An audit of the changes described in this document is yet to be performed.

## Testnet

The implementation is yet to be deployed to The Graph Rinkeby Testnet and the source code verified in Etherscan.

# Rationale and Alternatives

## Alternative Implementations

### Direct transfers sent to a rewards destination (GIP-0002)

An alternative implementation was proposed and described in GIP-0002. Please refer to that document for a detailed explanation.

We consider that this GIP supersedes GIP-0002 as it introduces a number of improvements, described in [Main Benefits](#main-benefits), collected from community feedback.

## Additional Considerations

The current proposal provides a mechanism to avoid default re-staking of the Indexer share of indexing rewards while delegation commissions are still sent to a shared delegation pool.

All income generated in the network for Curators and Delegators, when query fees are collected or indexing rewards minted, are deposited into a shared pool. This allows for gas efficiency, both delegators and curators claim their share of the bigger pool (what they deposited + rewards) when unsignaling or undelegating.

A change to how this mechanism works could be explored in a new GIP, including research on how to do a viable implementation, gas consumption analysis, security considerations, etc.

## Role of thawing periods in indexing and delegation

The feature was designed to avoid exploits in the protocol economics, not to lock participants for the purpose of restricting liquidity.

#### Indexer rewards pool

Adding a thawing period to the rewards pool removes any short term speculation that could lead Indexers to close multiple allocations in a short period of time, and that could end up affecting the liveness of the network.

Keeping a thawing period for withdrawing indexing rewards also preserves the balance of power between Delegators and Indexers, and removes any incentive for Indexers and Delegators to form extra-protocol arrangements for reward sharing.

#### Indexer stake thawing period

This period prevents Indexers from easily avoiding a dispute that would result in slashing them. When an Indexer unstakes they have to wait for a period of time, during which funds are still slashable, before they can fully withdraw their funds.

#### Delegator unbonding period

It was introduced to prevent the same delegated stake being used simultaneously across several Indexer allocations, which may last a period of time equal to Delegator's thawing period. If a Delegator could re-delegate multiple times within the span of a single allocation, different Indexers could be creating allocations using the same capital.

# Copyright Waiver

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
