---
GIP: 0052
Title: Timeline and requirements to increase rewards in L2
Authors: Pablo Carranza Velez <pablo@edgeandnode.com>, Ariel Barmat <ariel@edgeandnode.com>, Tomas Migone <tomas@edgeandnode.com>
Created: 2023-04-20
Updated: 2023-04-20
Stage: Draft
Discussions-To: TODO
Category: Economic Parameters
Depends-On: GIP-0043, GIP-0046
---

# Abstract

This proposal presents a potential timeline to migrate 100% of indexing rewards to the L2 instance of The Graph. It proposes four steps with L2 rewards going from 5% to 25%, 50%, staying for 95% for some time, and finally 100%. It presents requirements and an estimated time for executing each step.

# Motivation

In several GIPs, we’ve presented various steps to migrate The Graph Network to Arbitrum One. Now that the L2 network has been deployed, and other important milestones have been met, it would be desirable to have a clear timeline and exit criteria for completing the migration by switching 100% of indexing rewards to L2.

# Prior Art

This GIP builds on top of GIP-0031, GIP-0037, GIP-0040, GIP-0043 and GIP-0046.

# High Level Description

In GIP-0043 and the corresponding GGP-0021, the Council approved sending 5% of indexing rewards to L2. Rewards now follow a linear issuance formula, with an `issuancePerBlock` parameter specifying the amount of GRT that are assigned on each layer as total rewards per block. These are then split across subgraphs and allocations.

We propose four steps to take rewards from the current status to 100% minted on L2:

1) Bump to 25% on L2 after releasing all migration helpers as described in GIP-0046
2) Bump to 50% on L2 after at least two weeks of migration helpers working as expected, and some increase in L2 participation.
3) Bump to 95% on L2 two months after the previous step, as long as everything is working as expected and there is a considerable increase in L2 participation.
4) Bump to 100% on L2 a month after the previous step, as long as a large percentage of subgraphs have migrated to L2.

Note this proposal is more about the steps than the actual percentages, if the community or the Council prefer different issuance values for each step, the proposal can still be implemented with different values.

# Detailed Specification

The four steps would have some requirements or “exit criteria” that would need to be met before moving from one step to the next and performing the L2 rewards increase. This might push timelines back a bit, so the ETA provided for each step is to be taken as an optimistic estimate, and meeting the exit criteria should be the main driver for executing each step. Updates about meeting these criteria should be shared in the Forum when they happen.

Any issues that require making changes in the smart contracts should restart the timeline, to make sure there is enough time to ensure safety before another increase.

## 1) Increase to 25% on L2

ETA: week of May 8th, 2023

Criteria:

- L2 with 5% rewards has been running for at least 2 weeks
- GRT issuance has followed the expected formula throughout this time
- Smart contracts are working properly:
    - No smart contract issues have been reported
    - Indexers have successfully opened and closed allocations on L2
    - Indexers have successfully served queries on L2, and collected query fees
- Indexer Agent support for running on L1 and L2 with a single instance has been released
- GNS (Subgraphs / Curation) migration helpers have been released as per GIP-0046
- Stake and Delegation migration helpers have been released as per GIP-0046
- Vesting contract migration helpers have been released as per GIP-0046

## 2) Increase to 50% on L2

ETA: week of May 22nd, 2023

Criteria:

- L2 with 25% rewards has been running for at least 2 weeks
- GRT issuance has followed the expected formula throughout this time
- Smart contracts are working properly:
    - No smart contract issues have been reported
    - Indexers have successfully opened and closed allocations on L2
    - Indexers have successfully served queries on L2, and collected query fees
- Some (> 1) subgraphs have been successfully migrated to L2 using the migration helpers
- Some (> 1) indexers have used the stake migration helpers

## 3) Increase to 95% on L2

ETA: week of July 24th, 2023

Criteria:

- L2 with 50% rewards has been running for at least 2 months
- GRT issuance has followed the expected formula throughout this time
- Smart contracts are working properly:
    - No smart contract issues have been reported
    - Indexers have successfully opened and closed allocations on L2
    - Indexers have successfully served queries on L2, and collected query fees
- The number of indexers on L2 is at least 50% the number of indexers on L1
- Gateway QoS metrics on L2 are comparable to metrics on L1

## 4) Increase to 100% on L2

ETA: week of August 28th, 2023

This last bump will essentially disable the L1 network, as indexers will have little incentive to keep indexing subgraphs there. For this reason, we should wait until most subgraph owners have migrated to L2 to avoid outages, which is reflected in the last criterion.

Criteria:

- L2 with 95% rewards has been running for at least 1 month
- GRT issuance has followed the expected formula throughout this time
- Smart contracts are working properly:
    - No smart contract issues have been reported
    - Indexers have successfully opened and closed allocations on L2
    - Indexers have successfully served queries on L2, and collected query fees
- The number of indexers on L2 is (still) at least 50% the number of indexers on L1
- Gateway QoS metrics on L2 are comparable to metrics on L1
- At least 90% of L1 subgraphs with more than the minimum curation signal have migrated to L2

# Backwards Compatibility

This proposal is backwards compatible, though the L1 instance of The Graph would have zero rewards after this, which could be considered a “breaking change”.

# Dependencies

Depends on GIP-0043 / GIP-0037 / GGP-0021 (already executed), and GIP-0046.

# Risks and Security Considerations

This proposal would make The Graph more strongly dependent on Arbitrum One. Recent steps taken by Arbitrum (in particular, moving to DAO governance), are a good indicator that Arbitrum is moving in the right direction towards full decentralization, but Arbitrum is still not as decentralized and therefore censorship-resistant as Ethereum.

The risk of Arbitrum having a catastrophic failure has already been discussed in GIP-0031; if this were to happen, the community would have to quickly react to either 1) move back to L1 or 2) deploy to a new L2 (which could be a Graph-specific chain or another public network). Both have pros and cons and would require some development. In the meantime, Arbitrum should still continue to work in a read-only state, as it should be possible to set up RPC nodes that read from the existing state that has been posted to L2; this should allow indexing operations to continue working for a short time, while the solution is implemented.

# Validation

Validation is described in the Detailed Specification above.

# Rationale and Alternatives

An alternative proposal would be to keep rewards flowing on both layers for a longer time (or indefinitely). We believe moving 100% to L2 sooner is better, as it allows the next generation of improvements to the network to be developed as L2-native proposals. It also shortens the time in which all participants have to support the dual-network setup, that adds maintenance costs, friction and cognitive load. Keeping The Graph running on a single chain simplifies things considerably.

# Copyright Waiver

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
