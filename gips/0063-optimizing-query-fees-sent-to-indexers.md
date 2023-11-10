---
GIP: 0063
Title: Optimizing Query Fees sent to Indexers
Authors: Howard <howard@edgeandnode.com>
Created: 2023-11-08
Updated: 2023-11-08
Stage: Draft
Category: Economic Parameters
---

## Abstract

Today, at least 11% of query fees paid by data consumers do not ever make it to either the Gateway or the Indexers that provide the service to the data consumers. This GIP aims to reduce these transactional costs. This can be done without changing any smart contract code; instead, only two parameters need to be tweaked: removing the fees sent to curators and setting the protocol tax to zero. This will simplify how the protocol operates and potentially save GRT both for developers and Indexers.

## Introduction and Motivation

Curation in The Graph is mostly used by dapp developers to incentivize Indexers to index subgraphs. This is a necessary step to take before Indexers serve queries for a subgraph. Today, much of the curation on The Graph Network running on Arbitrum is from dapp developers providing their GRT to buy curation shares, signaling to Indexers that they want a subgraph indexed. Once the developer begins paying for queries, 11% of the fees are directed to parties other than the Gateway or relevant Indexer. The protocol can be simplified by removing this 11% cut being redirected ***and*** save GRT for both developers and Indexers.

## Prior Art

The existing smart contracts for The Graph route 10% of query fees to the curation pool and burn 1% of query fees. The proposal does not require any changes to the code/logic of the contracts; instead, it just requires changing the values of two parameters. Two related GIPs also aim to increase the efficiency of operations for participants in The Graph ecosystem. These include [GIP-0051](https://forum.thegraph.com/t/gip-0051-exponential-query-fee-rebates-for-Indexers/4162) that resulted in Indexers keeping a notably larger portion of query fees (by replacing Cobb-Douglas) and [GIP-0059](https://forum.thegraph.com/t/gip-0059-disable-subgraph-owner-tax-when-publishing-a-new-version/4460) about removing a tax on subgraph owners (to reduce the costs born when upgrading). 

## High Level Description

Of the total query fees generated, 1% are burnt, 10% are sent to curators on the relevant subgraph, and the remainder is sent to the Indexer (after going through the Exponential Rebate mechanism). This proposal is to set 0% of query fees to be burnt and 0% to be sent to curators on the relevant subgraph, with all of the remaining portion to be sent to Indexers via the Exponential Rebate mechanism. _This change would only be applied to the protocol on Arbitrum_.

## Detailed Specification

To deploy this GIP, the Council would have to:

- Set the curation percentage to zero by calling `L2Staking.setCurationPercentage(0)`
- Set the protocol tax to zero by calling `L2Staking.setProtocolPercentage(0)`

These two can be executed in a single transaction batch.

## Rationale

The rationale for this proposal is to reduce unnecessary frictional costs and simplify how the protocol operates. Below we expand upon two aspects of reasoning for this change, along with a note about the evolution of the curator role.

***Elimination of Circular Flow of GRT for Developers***

Consider a dapp developer that aims to have their subgraph indexed. On Arbitrum today, the developer can buy curation shares with GRT to provide “signal” to Indexers. Even after an Indexer decides to index the subgraph, the developer typically keeps their GRT deposited in the curation shares. When the developer then pays for queries, 10% of these are routed back to the subgraph’s holdings. If the developer is the only curator on the subgraph, this means 10% of the GRT they paid toward query fees goes back to themself, minus the transaction fees involved (****e.g.**** in selling the curation shares). This circulating of query fees back to the developer can be completely avoided by implementing this proposal.  This would save the developer GRT and remove an unnecessary but cognitively burdensome mechanism from the protocol.  

***Revenue to Indexers***

Today Indexers get a portion of query fees. Of the total, 1% are burnt, 10% are sent to curators on the relevant subgraph, and the remainder is sent to the Indexer (after going through the Exponential Rebate mechanism). Thus, even with the improvements in the proportion of query fees collected by Indexers as a result of Exponential Rebates, Indexers can collect at most 89% of query fees. We can increase this by 11%. 

***What about Curators?***

Curation was originally designed as a way to incentivize independent third parties to identify “valuable” subgraphs.  Unfortunately, this has not come to fruition for one main reason: the party that is often best informed about the value of a subgraph is the dapp developer.  As a result, a large amount of curation has simply been “self-signaling” and is functionally a mechanism for developers to pay (in terms of cost of capital) to have their subgraphs indexed. While providing incentives to identify useful information is a valid goal for a protocol (that is indeed the business model of journalism and news enterprises, among others), the current curation mechanism does not achieve this.  So while core dev teams in The Graph will consider and research alternative curation mechanisms in the future, this proposal will remove the currently ineffective incentive for third-party curators.  

## Risks and Security Considerations

The removal of 10% of query fees going to curators eliminates direct financial gain as an incentive for curators. Consequently, curators that are not dapp developers (or another role in The Graph ecosystem) may be expected to sell their curation shares. Since this proposal will only affect query fees on Arbitrum, curators can sell their shares without any loss from their originally deposited GRT.

Since the proposed update is merely a parameter change and does not require any changes to the code in smart contracts, minimal risk is expected with respect to introducing smart contract exploits. No external audits are planned; however, we recommend that the two transactions required for implementing this proposal are tested using simulation and on Testnet before executing in production.

## Future Directions

Since this proposal removes _direct_ incentives for third-party curators, future directions will include researching alternative curation incentives and consider whether such a mechanism is in scope for The Graph protocol.  

## Copyright Waiver

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
