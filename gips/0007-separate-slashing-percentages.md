---
GIP: 0007
Title: Separate slashing percentages for query and indexing disputes
Authors: Ariel Barmat <ariel@edgeandnode.com>
Created: 2021-05-21
Updated: 2021-06-03
Stage: Candidate
Discussions-To: https://forum.thegraph.com/t/different-slashing-percentages-for-query-and-indexing-disputes/2020
Implementations: https://github.com/graphprotocol/contracts/pull/458
Audits: https://blog.openzeppelin.com/thegraph-slashing-upgrade-audit/
---

# Abstract

This proposal introduces two different slashing percentages, one for indexing disputes and another for query disputes. This change allows slashing percentages to be set in such a way that accounts for the different risks of producing slashable faults while indexing and querying.

# Motivation

One of the security mechanisms in The Graph relies on filing disputes whenever an Indexer presents a wrong Proof of Indexing (PoI) or if it returns a query with inaccurate data.

After any participant files a dispute, the Arbitrators will decide if the dispute is valid according to norms established via protocol governance. If the Arbitrator accepts the dispute, the offending Indexer's stake is slashed according to a governance parameter called `slashingPercentage`.

The issue with having a single `slashingPercentage` parameter for both indexing and query disputes is that an Indexer, in the regular day-to-day operation, will return a disproportionately higher amount of queries than PoIs. As a result, servicing queries is a higher-risk activity than indexing.

This proposal allows two different slashing percentages to balance the risk, one for indexing disputes and another for query disputes.

# Detailed Specification

All the changes required to implement this proposal are related to the **DisputeManager** contract.

## Current Behavior

By default, every time disputes are accepted, indexers get slashed on the their total stake using the single `slashingPercentage` protocol parameter, both for indexing dispute and query disputes.

## Proposed Changes

#### 1) Introduce separate protocol parameters for query and indexing slashing

Change set slashing percentage signature to accept two different parameters.

```
function setSlashingPercentage(uint32 _qryPercentage, uint32 _idxPercentage)
```

Add two state variables in the contract storage:

```
uint32 public qrySlashingPercentage;
uint32 public idxSlashingPercentage;
```

#### 2) Add dispute type when created

Whenever a new dispute is created we store the dispute type. This way the contract can resolve using the appropiate slashing percentage.

```
enum DisputeType { Null, IndexingDispute, QueryDispute }

// Disputes contain info necessary for the Arbitrator to verify and resolve
struct Dispute {
    address indexer;
    address fisherman;
    uint256 deposit;
    bytes32 relatedDisputeID;
    DisputeType disputeType;
}
```

#### 3) Dispute resolution

Update the `acceptDispute` function to slash based on the dispute type.

# Backwards Compatibility

The current solution changes the getter of the previous `slashingPercentage` variable. Anyone doing an integration with the contracts need to take special care to call the proper `idxSlashingPercentage` or `qrySlashingPercentage` according to their needs.

Additionally, the function `setSlashingPercentage(uint32 _percentage)` changed to `setSlashingPercentage(uint32 _qryPercentage, uint32 _idxPercentage)`.

# Validation

## Audits

An [audit](https://blog.openzeppelin.com/thegraph-slashing-upgrade-audit/) of the changes described in this document has been performed by OpenZeppelin.

## Testnet

The implementation is yet to be deployed to testnet and the source code verified in Etherscan.

# Copyright Waiver

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
