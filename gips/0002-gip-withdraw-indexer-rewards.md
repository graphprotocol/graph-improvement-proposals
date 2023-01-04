---
GIP: 0002
Title: Separate destination address for Indexer indexing rewards.
Authors: Ariel Barmat <ariel@edgeandnode.com>
Created: 2021-03-01
Stage: Candidate
Discussions-To: https://forum.thegraph.com/t/rewards-destination-for-indexers-indexing-rewards/1279
Implementations: https://github.com/graphprotocol/contracts/pull/450
---

# Abstract

This proposal describes a mechanism for Indexers to withdraw rewards to a designated address instead of automatically re-staking these rewards in the protocol. The effect is that Indexers that interact with the protocol via the widely used TokenLock vesting contracts will be able to withdraw indexing rewards to cover operational costs, without fully unstaking from the protocol.

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

## Proposed Changes

#### 1) Introduce a function to allow any Indexer to set the rewards destination.

`function setRewardsDestination(address _destination) external`

It can be set to any of these options:

- **0x0:** When the address is set to empty any rewards will be staked.
- **Destination address:** The address where to send the minted indexing rewards.

This function will emit an event with signature `event SetRewardsDestination(address indexed indexer, address indexed destination);` whenever it is set.

Note: An Indexer can only set the rewards destination using the Ethereum account they initially staked from and not using any operator keys they may have registered in the protocol.

#### 2) Update rewards distribution function

`_distributeRewards(address _allocationID, address _indexer) private`

This function currently stakes *indexerRewards* by default if any is present.

**Current implementation:**
```
// Add the rest of the rewards to the Indexer stake
if (indexerRewards > 0) {
    // TODO: << check if rewards destination is set to stake or send to address >>
    _stake(_indexer, indexerRewards);
}
```

In the proposed implementation, check the rewards destination for the Indexer to whether stake or transfer funds to destination address.

**New implementation:**
```
// Send the indexing rewards
_sendRewards(
    graphToken(),
    indexerRewards,
    _indexer,
    rewardsDestination[_indexer] == address(0)
);

```
The `sendRewards()` private function encapsulates the staking or sending of rewards to the destination configured by the indexer.

### Main Benefits

- By setting an Indexer's defined address to send rewards, we can avoid staking and later withdrawal to TokenLock contract that would otherwise lock the funds.
- By setting the rewards destination once using the `setRewardsDestination()` function we do not need to change the close allocation function signature.
- The feature will work for rewards collected after its implementation, past rewards remain staked.
- Claimed rewards from the rebate pool will use the same destination address when chose not to restake.

# Backwards Compatibility

This solution is backward compatible. This means that any Indexer not setting rewards destination will default to the current behavior to re-stake any rewards. Additionally, there is no breaking change on current interfaces that will result in clients needing to adapt.

# Validation

## Audits

An audit of the changes described in this document were performed. This is the link to the preliminary audit report: https://gist.github.com/xaler5/6e31e0521f1cdd58c17b229de30954f8

## Testnet

The implementation was deployed to The Graph Rinkeby Testnet and the source code verified in Etherscan.

**Implementation deployment:**
https://rinkeby.etherscan.io/tx/0x691f49ce9777e23e6d3dcaf9c7c6c1886eacee32a7e9ba1574c2b8c23e316454

**GraphProxyAdmin updating implementation of Staking:**
https://rinkeby.etherscan.io/tx/0x9525373a4968265ff83ac365bc37195495c232962f01a0e8072f477a41a74786

A number of Indexers already tested the implementation in Rinkeby.

**Close allocation sending rewards to destination:**
https://rinkeby.etherscan.io/tx/0x536a83254c1c78dbcb72bbe967cf528dfe86877d489548bd1c1696def0f3cf88

# Rationale and Alternatives

## Alternative Implementations

### Indexing rewards deposited to a rewards pool

An alternative implementation was tested to benchmark gas efficiency. The idea is to use a rewards pool to collect rewards from multiple allocations and the Indexer then decides when to withdraw them.

**Implementation (#2):** https://github.com/graphprotocol/contracts/pull/451

**Benchmark (gas usage):**
```
closeAllocation()

- #1 direct destination-address-no-balance: 271898
- #1 direct destination-address-with-balance: 256886
- #2 rewardsPool no-balance: 265438
- #2 rewardsPool with-balance: 250438
```

The *no-balance* and *with-balance* distinction is analyzed as in Ethereum the SSTORE opcode takes more gas when a state variable is zeroed, this is why the "no-balance" condition is more expensive as it is setting the state variable to a non-zero value.

Reasons to disregard this implementation:
- The main implementation is simpler, we add less state, avoid new getter functions and function calls which require UX changes.
- The gas gains are initially negligible and require more study in real-life conditions about how Indexers will withdraw their rewards to evaluate properly.
- A rewards pool can be added later if it is really needed and there is community support.

## Additional Considerations

The current proposal provides a mechanism to avoid default re-staking of the Indexer share of indexing rewards while delegation commissions are still sent to a shared delegation pool.

All income generated in the network for Curators and Delegators, when query fees are collected or indexing rewards minted, are deposited into a shared pool. This allows for gas efficiency, both delegators and curators claim their share of the bigger pool (what they deposited + rewards) when unsignaling or undelegating.

A change to how this mechanism works could be explored in a new GIP, including research on how to do a viable implementation, gas consumption analysis, security considerations, etc.

## Role of thawing periods in indexing and delegation

The feature was designed to avoid exploits in the protocol economics, not to lock participants for the purpose of restricting liquidity.

#### Indexer thawing period

This period prevents Indexers from easily avoiding a dispute that would result in slashing them. When an Indexer unstakes they have to wait for a period of time, during which funds are still slashable, before they can fully withdraw their funds.

#### Delegator unbonding period

It was introduced to prevent the same delegated stake being used simultaneously across several Indexer allocations, which may last a period of time equal to Delegator's thawing period. If a Delegator could re-delegate multiple times within the span of a single allocation, different Indexers could be creating allocations using the same capital.

# Copyright Waiver

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
