---
GIP: "0044"
Title: Enforce minimum indexer stake across all indexer actions
Authors: Pablo Carranza Vélez <pablo@edgeandnode.com>, Ariel Barmat <ariel@edgeandnode.com>, Tomás Migone <tomas@edgeandnode.com>
Created: 2023-02-03
Updated: 2023-02-03
Stage: Candidate
Category: "Protocol Logic", "Protocol Interfaces", "Economic Parameters"
---

# Abstract

The minimum indexer stake is the GRT that indexers deposit as a warranty to participate in the protocol. This stake is subject to slashing if the indexer misbehaves, and the goal is to ensure the system's overall security. The protocol enforces this amount whenever the indexer stakes or un-stakes, but it can get below the minimum under certain conditions. In this GIP, we propose ever validating the minimum stake whenever an indexer accesses the staked amounts.

# Motivation

The primary motivation for this change is to ensure the correctness and consistency of how the protocol manages the indexer stake. Certain edge cases can lead to the minimum stake being under the minimum while still allowing the indexer to operate in the protocol. One such is when a dispute slashes an indexer for an amount that puts it under the minimum.

# High-Level Description

The protocol must consistently enforce the minimum indexer stake whenever an indexer accesses the staked amount.

These actions are ones such as:

- Creating an allocation
- Staking
- Unstaking

The protocol is already checking that the stake is over the minimum whenever an indexer stakes. It is also checking that an indexer never unstakes under the minimum, and if doing so it will only work as long as the indexer is unstaking in full. The only remaining condition that is left to check is when an indexer creates an allocation.

An indexer under the minimum stake will be considered INACTIVE and only able to allocate once they top-up their stake. An indexer is ACTIVE as long as its commited stake, the amount of GRT not thawing for withdrawal, is above the minimum stake.

# Validation

This proposal has not yet been reviewed nor audited.

# Copyright Waiver

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
