---
GIP: 0006
Title: Withdraw helper contract to support collecting funds from Vector channels.
Authors: Ariel Barmat <ariel@edgeandnode.com>
Created: 2021-05-21
Stage: Candidate
Implementations: https://github.com/graphprotocol/contracts/pull/454
---

# Abstract

Consumers in the Graph Network set up state channels for sending query fees to Indexers using Vector. [Vector](https://github.com/connext/vector) is an ultra-simple, flexible state channel protocol implementation. This proposal introduces a contract required to encode custom logic in the process of withdrawing query fees from Vector state channels. The **GRTWithdrawHelper** contract receives query fees from a Vector state channel, then it forwards them to the **Staking** contract where they are distributed according to the protocol economics.

# Motivation

Vector state channels are contracts that hold funds used as collateral for consumer-to-indexer payments. When an Indexer closes an allocation, both parties sign a withdrawal commitment to transfer the query fees accumulated for the allocation back to the network. Vector, by default, supports plain transfers from the channel to any destination defined in the withdrawal commitment. In the case of the Graph Network, all funds should be collected by calling the collect() function in the Staking contract. For the purpose of calling custom logic when transferring from channels, Vector supports using a **WithdrawHelper** contract. In this proposal, we describe the implementation of such a contract for connecting a Vector withdrawal transfer to The Graph smart contracts.

# Detailed Specification

We created a contract called **GRTWithdrawHelper** that extends the base **WithdrawHelper** provided by the Vector framework. This contract has a function `execute(WithdrawData _wd, uint256 _actualAmount) external` that is called in the same transaction when the indexer withdraw funds from the channel. The execute function evaluates the validity of the `WithdrawData` struct, and then approve and transfer the received funds to the **Staking** contract by calling `collect()`.

### Transaction Flow

```sequence
Channel->GRTWithdrawHelper: transfer(_actualAmount)
Channel->GRTWithdrawHelper: execute(WithdrawData _wd, uint256 _actualAmount)
GRTWithdrawHelper->Staking: approve(_actualAmount)
GRTWithdrawHelper->Staking: collect(_actualAmount, allocationID)
Staking->GRTWithdrawHelper: transferFrom(_actualAmount)
```

## Features

- The **GRTWithdrawHelper** is deployed with the token address that will be used for transfers, in our case it is the GRT token address. It can't be changed at later stage but this can easily be solved deploying a new contract if needed.
- The contract expects to have available tokens in balance when `execute(WithdrawData calldata wd, uint256 actualAmount)` is called by the Channel. For that to happen the Channel needs to transfer funds to the **GRTWithdrawHelper** before execute is called.
- On execute, the **GRTWithdrawHelper** will approve the funds to send to the Staking contract and then execute `collect()` passing the proper allocationID and amount. The Staking contract will then pull the tokens.
- If the call to `collect()` fails, the funds are returned to a return address. This is important for Vector accounting of funds.

## Requirements

- Set the **GRTWithdrawHelper** contract as **AssetHolder** in the **Staking** contract. This way we allow the network contracts to accept funds from this contract.
- The cooperative withdrawal commitments must have the following data (indicated with arrows):

```
struct WithdrawData {
    address channelAddress;
    address assetId;            // -> GRT token address
    address payable recipient;  // -> GRTWithdrawHelper contract address
    uint256 amount;
    uint256 nonce;
    address callTo;             // -> GRTWithdrawHelper contract address
    bytes callData;             // -> CollectData struct defined below
}
```

```
struct CollectData {
    address staking;        // -> Staking contract address
    address allocationID;   // -> AllocationID to send collected funds
    address returnAddress;  // -> Address where to return funds in case of errors
}
```

The parties must agree and sign the WithdrawalCommitment with that information to achieve proper execution of the withdrawals.

## Exceptions

The following conditions will result in reverts of the execution:

- `assetId` in `WithdrawData` not matching the token address configured in the **GRTWithdrawHelper**.
- **GRTWithdrawHelper** not having enough balance to transfer the **Staking** contract as stated in `actualAmount`.
- Setting an invalid staking contract address.

Note: To avoid unexpected conditions in the Vector off-chain logic, it is encouraged that the parties verify the WithdrawData does not revert by doing a call/estimateGas.

## Handled Exceptions

These reverts are handled and will make the contract to return the funds to the return address.

- Doing a withdrawal for a non-existing `allocationID` as defined in `CollectData` when calling `collect()`.

## Assumptions

- This contract does not know if a `WithdrawalCommitment` is correct, it just accepts a transfer and `WithdrawData` that defines a proper destination.

- A properly formatted `WithdrawData` can be crafted to send outside funds to the **Staking** contract. In that case, the person will get those funds distributed with Curators + Delegators + taxed a protocol percentage. There is no rational economic incentive to do so.

# Backwards Compatibility

The contract in this proposal is new and peripheral to the rest of the network contracts. No breaking change is introduced. The Graph Council will need to set the **GRTWithdrawHelper** as an **AssetHolder** in the Staking contract.

# Validation

## Audits

An audit of the changes described in this document was performed by ChainSafe (see [`assets/gip-0006/TheGraph-GRTWithdrawHelper-Audit.pdf`](../assets/gip-0006/TheGraph-GRTWithdrawHelper-Audit.pdf)).

## Testnet

The implementation was deployed and tested on Rinkeby testnet. https://rinkeby.etherscan.io/address/0xe5fa88135c992a385aaa1c65a0c1b8ff3fde1fd4

# Copyright Waiver

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
