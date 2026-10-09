---
GIP: "0090"
Title: Community Allocation of 0.1% of issuance to The Night's Watch
Authors: Petko Pavlovski <petkopavlovski@gmail.com>, The Night's Watch
Created: 2026-09-23
Stage: Draft
Discussions-To: <https://forum.thegraph.com/t/gip-0090-community-allocation-of-0-1-of-issuance-to-the-nights-watch/7083>
Category: Protocol Logic
Depends-On: GIP-0076, GIP-0088, GIP-0089
Implementations: https://github.com/nuthatch-org
---

## Abstract

This proposes allocating 0.1% of protocol issuance, 0.12073 GRT per block, to The Night's Watch through the same mechanism GIP-0089 used for the Innovation Allocation: a new instance of the audited DirectAllocation contract, added as an allocator-minting target on the Issuance Allocator. It funds work that is already running on zero budget and will keep running either way: Lodestar, the community's research and engineering against the Catalyst roadmap, daily indexer and subgraph developer support through graph-support and the Discord, and the nuthatch and Graph integration. The allocation is time-boxed to twelve months, reported quarterly on this forum, and removable by governance in one transaction. At 2025's block count it is about 315,000 GRT a year. At today's price that is under eight thousand dollars. It is a symbolic amount, and the point of it is the symbol: the network paying, for the first time, for community infrastructure it already depends on.

## Motivation

GIP-0089 went live on 1 September and was explicit about its policy: the Innovation Allocation "replaces broad, open-ended grants to external teams with directed funding tied to the protocol's strategic priorities." We agree with the direction. The difficulty is that the community work most aligned with those priorities is now outside every funding path the protocol has. It is not a core dev, it is not on the Foundation's payroll, and grants are no longer the model.

Concretely, over the last six weeks:

- graph-support took 43 issues from indexers and subgraph developers, found the root cause on 36 of them, and turned three into upstream graph-node and docs pull requests and three more into upstream issues. The 16 September outage, where one CRLF manifest stopped every version of the network subgraph and took the gateway down for three hours, was diagnosed and written up there before any official post existed.
- Lodestar stopped reading any Graph subgraph on 7 September and now serves the whole dashboard from self-hosted nuthatch nests and one Rust process. An indexer asked for an omni search at 11:42 and it was live at 13:42 the same day.
- The Catalyst tracker maps all eight roadmap items to public repos with a measured score against each, currently between 42% and 74%, and records what only the Foundation can move: the external audits and the second gateway operator.

All of that is public, permissively licensed, and paid for out of one person's pocket. The 28 August post "On the Catalyst roadmap" offered it to the incoming team and asked for a conversation, not money. It has 52 views and no replies. This proposal asks the network directly instead.

## Prior Art

GIP-0076 introduced the Issuance Allocator and DirectAllocation. GIP-0088 deployed the allocator and reserved a 5% target for DIPS. GIP-0089 added the Innovation Allocation at 20%, operated by a Foundation multisig, with no new contract code. This proposal is the same shape at one two-hundredth of the size, with two things GIP-0089 does not have: a sunset and a quarterly report.

## High-Level Description

No new smart contract code. One new deployment of the existing, audited DirectAllocation contract, with a Night's Watch multisig as operator, added as an allocator-minting target at 0.12073 GRT per block. The Rewards Manager's allocation is reduced by the same amount. Total issuance stays at 120.73 GRT per block.

## Detailed Specification

### 1. Community Allocation management

A DirectAllocation instance is deployed with a 2-of-3 multisig on Arbitrum One as operator. Signers are published in this thread before any Council vote. The Issuance Allocator mints to the contract; the operator withdraws as needed. Governance retains the allocator's target list and can remove the target at any time.

### 2. Allocation configuration

Read from the Issuance Allocator on Arbitrum One on 23 September 2026, and the proposed change:

| Target | Type | Now (GRT per block) | Proposed | Share |
| :--- | :--- | ---: | ---: | ---: |
| Rewards Manager | Self-minting | 96.584 | 96.46327 | 79.9% |
| Innovation DirectAllocation | Allocator-minting | 24.146 | 24.146 | 20% |
| Community DirectAllocation (this proposal) | Allocator-minting | 0 | 0.12073 | 0.1% |
| **Total** | | **120.73** | **120.73** | **100%** |

When the GIP-0088 Phase 3 split to the Recurring Agreement Manager activates, the 6 GRT per block comes from the Rewards Manager as already approved and this allocation is unaffected.

For reference, there were 2,610,162 blocks in 2025. At 0.12073 GRT per block the allocation is approximately 315,000 GRT a year.

### 3. What it funds

In priority order, and all of it already running:

1. **Lodestar.** Kept live, free, and independent of any Graph subgraph or gateway. Hosting, RPC and data bills are the first call on the funds.
2. **Catalyst research and engineering.** The tracker stays public and is updated at least quarterly. The workstreams we can still move alone (gateway operator tooling, the multi-product Studio surface, the audit layer) keep moving. Where an item is gated on the Foundation, the tracker says so rather than pretending.
3. **Support.** graph-support keeps its 48 hour human reply and its rule that no issue closes without a stated root cause. The Discord stays open with no application and no fee. Fixes that belong upstream go upstream as pull requests.
4. **nuthatch and The Graph.** The public Network Subgraph endpoint, the Graph protocol nests, hosted nests for hackathon teams, and the subgraphs Studio cannot yet take.
5. **New data services.** Reference implementations any indexer can run, in the open, with the traps documented.

At this size nothing goes to salaries. If a quarter's spend is below the quarter's allocation, the remainder is carried and reported, not distributed.

### 4. Governance and monitoring

- The Graph Council approves the allocation.
- The Night's Watch posts a report on this forum every quarter: what was delivered, what was spent, every withdrawal by transaction hash.
- The allocation expires twelve months after activation unless the Council renews it. A missed quarterly report is grounds for immediate removal, and removal is one governance transaction on the allocator.

## Implementation

1. Deploy a DirectAllocation instance with the Night's Watch multisig as operator.
2. Governance adds it as an allocator-minting target and sets the split: Rewards Manager 96.46327, Community Allocation 0.12073, Innovation Allocation unchanged at 24.146.
3. Verify that the targets sum to 120.73 GRT per block and that the first distribution mints to the new contract.

## Backward Compatibility

No changes to existing contracts. Indexing rewards fall by 0.125% of their current level. A Council that preferred to leave indexers untouched could source the 0.12073 GRT per block from the Innovation Allocation instead, which would reduce it by half of one percent; the allocator arithmetic is identical and we would accept either.

## Risks and Security Considerations

1. **Smart contract risk.** None new. Same audited DirectAllocation, new instance.
2. **Economic impact.** Negligible by construction. The amount is one two-hundredth of the allocation approved three weeks ago.
3. **Governance risk.** The allocation is to a small community group with a single active maintainer. That is mitigated by the multisig, the sunset, the quarterly report with hashes, and the fact that governance can stop it in one transaction. It is also, frankly, the risk the proposal exists to test: whether the protocol can fund community infrastructure at all without a grant programme.

## Copyright Waiver

Copyright and related rights waived via CC0.
