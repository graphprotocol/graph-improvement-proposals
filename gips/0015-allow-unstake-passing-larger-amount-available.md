---
GIP: "0015"
Title: Allow unstake passing a larger amount than available
Authors: Ariel Barmat <ariel@edgeandnode.com>
Created: 2021-09-25
Stage: Candidate
Implementations: https://github.com/graphprotocol/contracts/pull/487
---

# Abstract

The Staking contract verifies that an indexer never unstakes under the minimum indexer stake, if the indexer is going to do so, it must unstake _fully_ back to zero tokens. This condition combined with the use of the `stakeTo` function can be used to frontrun an indexer that is unstaking fully and make this action to revert.

# Motivation

Whenever an indexer unstakes, the Staking contract verifies that the stake is over `minimumIndexerStake`. Because of this, a partial unstake that is under the `minimumIndexerStake` will revert. There is a particular edge case where the unstake transaction can be frontrun leading to the indexer to fail to fully unstake.

The condition happens during following example sequence:

1. Indexer has _200,000_ staked tokens, the `minimumIndexerStake` is _100,000_
2. Indexer send a `unstake(200,000)` transaction to unstake fully.
3. A malicious actor sends a `stakeTo(indexerAddress, 1)` to stake just one token on the indexer address **right before transaction from item #2** gets mined.
4. The transaction from step **#2** gets mined and will **revert**. Even if the attacker is gifting tokens to the indexer it will make the unstake transaction to revert, because the contract will find the indexer has 200,001 tokens and by unstaking 200,000 it will be under the minimum stake.

# Specification

Allow the `unstake()` function to receive any amount of tokens to unstake, even larger than the current stake (like MAX_UINT256). Then use `unstakeAmount = min(currentStake, unstakeAmount)` to get the actual unstake amount. This way we cap the unstake amount to the max staked tokens when the transaction gets processed and it doesn't revert based on the passed amount.

# Implementation

See [@graphprotocol/contracts#487](https://github.com/graphprotocol/contracts/pull/487)

# Backwards Compatibility

The proposal is fully backwards compatible.

# Validation

### Audits

The implementation was audited by Consensys Diligence.

### Testnet

The implementation has not yet been deployed to Testnet.

# Copyright Waiver

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
