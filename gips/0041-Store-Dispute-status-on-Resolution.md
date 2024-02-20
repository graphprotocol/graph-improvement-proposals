---
GIP: "0041"
Title: Store Dispute status on Resolution
Authors: jordan@soulbound.xyz goose@soulbound.xyz
Created: 2023-11-22
Updated:
Stage: Draft
Category:
---

### Abstract

The Dispute Manager contract allows fisherman to police Indexer activity by submitting disputes for invalid indexing or query responses. Right now, disputes are deleted on resolution instead of being marked as accepted, rejected, or drawn. This prevents other smart contracts from tracking the status of disputes and past Indexer slashing events. Rather than delete disputes, this proposal suggests that the Dispute Manager contract should permanently store disputes alongside their status.

### Motivation

Soulbound Labs is building a Subgraph Bridge designed to offload prohibitively expensive computations from the EVM to The Graph.

Our goal is to prevent query responses from Indexers from being certified on-chain if that response is tied to either an active or valid dispute.

We consider this is a great addition to the core functionality of the Dispute Manager that will help transform subgraphs into read-oriented rollups.

### [](https://forum.thegraph.com/t/gip-0041-store-dispute-status-on-resolution/3759#specification-3)Specification

Modify the Dispute struct in [IDisputeManager.sol](https://github.com/graphprotocol/contracts/blob/dev/contracts/disputes/IDisputeManager.sol) to include the current status of the dispute.

```
   struct Dispute {
        address indexer;
        address fisherman;
        uint256 deposit;
        bytes32 relatedDisputeID;
        DisputeType disputeType;
        DisputeStatus disputeStatus; /* New disputeStatus enum */
    }

    enum DisputeStatus {
        Null,
        Accepted,
        Rejected,
        Drawn,
        Pending
    }

```

Add an `onlyPendingDisputes` modifier.

Modify the `_resolveDispute` and `_resolveDisputeInConflict` methods in [DisputeManager.sol](https://github.com/graphprotocol/contracts/blob/dev/contracts/disputes/DisputeManager.sol) to accept and set a `DisputeStatus` value on a `Dispute`.

Modify `acceptDispute`, `rejectDispute`, and `drawDispute` methods to call the updated `_resolveDispute` and `_resolveDisputeInConflict` calls with the appropriate status. Note that `acceptDispute` will set the valid dispute's status to `Accepted` and the invalid dispute's status to `Rejected`.

Modify `_createIndexingDisputeWithAllocation` and `_createQueryDisputeWithAttestation` methods to create new disputes with a `Pending` status.

Modify the `isDisputeCreated` function to use the new dispute status.

### Implementation

A pull request is open [here](https://github.com/graphprotocol/contracts/pull/766).

### Backwards Compatibility

This proposal is fully backward compatible.

### Validation

Audits
The implementation has not yet been audited.

### Testnet

The implementation has not yet been deployed to Testnet.

### Copyright Waiver

Copyright and related rights waived via CC0.
