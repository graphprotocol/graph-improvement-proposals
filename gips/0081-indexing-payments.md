---
GIP: 0081
Title: A New Proposal for Indexing Payments
Authors: Matias <matias@edgeandnode.com>, Pablo <pablo@edgeandnode.com>
Created: 2024-10-21
Updated: 2024-10-21
Stage: Draft
Discussions-To: <TODO>
Category: Protocol Logic
Depends-On: GIP-0066, GIP-0068
Replaces: GIP-0058
---

## Abstract

This GIP describes **Indexing Payments**, a mechanism by which a protocol participant (commonly a gateway) can incentivize a candidate indexer to service a subgraph. It details a target, trust-minimized end-state that relies on Graph Horizon, while also introducing a semi-trusted Minimum Viable Product (MVP) based on the current protocol and off-chain components (gateway + indexer stacks). This GIP is an alternative, simplified proposal to [GIP-0058](https://github.com/graphprotocol/graph-improvement-proposals/blob/main/gips/0058-Replacing-Bonding-Curves-with-Indexing-Fees.md).

## Motivation

Curation has generally been perceived as a source of friction for Developers that may only want to have subgraphs indexed to power their decentralized applications. It requires locking up certain amounts of GRT, and the necessary amounts might vary depending on curation on other subgraphs. While the quality of service provided on the network is generally high, Indexers collect rewards for posting Proofs of Indexing independently to the query service they provide, so Developers are not guaranteed a certain level of service for the amount of curation they put into the system.

For Indexers, there is no clear relationship between the Indexing Rewards on a subgraph and the amount of work required to index it. Attempts at defining better curation and rewards models to pay for indexing have so far been unsuccessful, which presents a need for a simpler mechanism where parties interested in indexing a subgraph can simply pay for it through the protocol. However, when building such a mechanism, it is imperative to preserve the protocol's trust-minimization and decentralization principles.

## Prior Art

GIP-0058 presents a previous version of a very similar proposal. This GIP re-introduces the same concepts, but incorporates the latest approach in Graph Horizon, and simplifies the proposal with an off-chain MVP while clarifying that the actor engaging in **Indexing Payments** is generally a Gateway running automated indexer selection.

## High-Level Description

We are proposing a new coordination mechanism to incentivize subgraph serving to a performant level. Such a coordination mechanism is necessary to jump-start, and sometimes maintain, performant subgraph serving. One such time is when a subgraph is being upgraded. In early conversations this has been referred to as **Direct Indexing Payments** (DIPs).

In particular, the coordination mechanism has been designed with the following in mind:

- The objective is to get subgraphs served through the protocol to a sufficiently performant level
- The price to get a subgraph served at said level should ideally be related to the amount of work required.

While the proposal is meant to fully support this use case, it does not exclude other use cases that may organically arise. Several components are general and can be re-used.

Looking further into the above, we assert that as described, it has some concerns that we need to address:

1. Desired performance should be defined by the payer.
1. Performance can only be assessed after the service has been consumed.
1. The cost of serving a subgraph is unknown a-priori, and it depends on the desired query performance as well as the indexing cost.

To solve for the first two, we propose that the gateway is the one coordinating with the indexers. Gateways can define and assess Quality of Service (QoS) on an ongoing basis and decide which indexers to continue to work with based on that.

To solve for the last concern, we propose that indexers are paid proportionally to the amount of work they do to index the subgraph. This amount of work is reported by indexers and thus needs to be sufficiently verifiable for gateways, fishermen or other indexers to dispute how much work was actually done.

In particular we propose that the coordination mechanism is underpinned by an **Indexing Agreement** to be drafted by the gateway, shared with a subset of indexers of its choosing and subsequently approved by some of those indexers. These agreements are designed to allow for indexers to work and collect for their work without having to go back to the payer. They are self-contained in that regard. Furthermore, they specify the following:

1. the subgraph to be indexed
1. the data service that will initiate payment collection
1. a deadline by which the agreement must be accepted
1. an end date to the agreement
1. a maximum amount payable for the initial indexing.
1. a maximum amount payable for the ongoing indexing. The ongoing indexing is the indexing that happens between the last collection and the present.
1. the maximum amount of time that can elapse between collections.
1. a price per unit of work done.

This agreement is to be upheld by protocol smart contracts, leveraging horizon primitives. This means that payers will have to escrow funds similarly to what's done for query fees. Between the smart contracts and the escrow, indexers are guaranteed that if they accept an indexing agreement, they'll be compensated as agreed, without the need to trust the agreement's counterpart.

At the same time, agreements are cancellable by either party at any time, with some limitations put in place to allow indexers to collect for all work done up to that point.

Putting all of this together the solution consists of the following:

- Indexers disclose publicly their price per unit of work.
- Gateways send indexing agreements to indexers they want to work with.
- Indexers accept (on-chain) agreements that meet their criteria and get to work indexing.
- On a regular basis:
  - Indexers collect for their work, by posting on-chain the original agreement, a Proof of Indexing (PoI) and the amount of work done for that period.
  - Gateways monitor the QoS per open agreement and assess if they need to cancel (on-chain) and/or replace any of them.

This solution aims to:

- compensate indexers for the exact amount of work they do.
- allow indexers to gracefully exit the agreement if it can't be fully fulfilled.
- cap the amount payers will spend on indexing. Though they might spend but get no service back if the cap is too low.
- incentivize sufficient QoS by awarding agreements to indexers that provide it.
- provide sufficient transparency and flexibility for gateways, so that they can abstract the complexity and provide simpler pricing to subgraph developers

### Indexing Indexer Selection Algorithm (IISA)

As mentioned above, gateways get to choose who they work with. In Horizon, gateways can be run be any protocol participant and they get to decide how to implement the indexer selection. Nevertheless we propose that a good IISA needs to consider QoS, available stake (with delegation), and other parameters to find the optimal balance between QoS, load-balancing, economic security and price.

### Arbitration and disputes

Since collections are tied to "amount of work done" and this is reported by the indexers themselves, there needs to be a mechanism to punish malicious reporting. We propose here to use slashing. For this to be fair we need the "amount of work done" to be verifiable by an arbitrator in the case of a dispute.

### Amount of work done

As an initial iteration of the "amount of work done" proxy we propose to use indexed blocks and subgraph entities. In particular the amount to be collected by indexers shall be equal to:

- A price per block indexed (with a different price for each chain). These are charged incrementally, i.e. every time a PoI is posted, the indexer collects a payment for the new blocks indexed since the last PoI.
- A price per extracted entity of the subgraph (same price for each chain). These are charged on a recurring basis per unit of time, for the total number of entities on the subgraph at the time the PoI is posted times the time since the last PoI.

We understand that this approach is vulnerable to some forms of abuse. For example, malicious users can deploy subgraphs that are expensive to index but amount to a comparatively small "amount of work done" proxy fee. We suggest that indexers stop serving such subgraphs as soon as possible, to minimize the damage done.

We anticipate needing to perform research into alternative approaches in the near future. We theorize that "amount of work done" may be modeled as a **subgraph gas** unit down the line, similar to ethereum gas.

### Minimum Viable Product

While this proposal defines a smart contracts based solution using the Subgraph Service on Graph Horizon, we also propose an MVP that can be built without any smart contracts changes using off-chain interfaces between Indexers and Gateways. This will allow us to implement parts of the solution that will be useful for the full proposal implementation, will also allow us to learn from seeing the mechanism in operation, and will help define the user interfaces and pricing approaches for subgraph developers that can be built on top of the protocol. As the MVP does not require changes to the on-chain protocol, it technically does not require a formal GIP approval, but we present it here alongside the full proposal to gather consensus towards this approach and to invite Indexers to participate in this initial stage and provide feedback.

In this MVP, we will roll out a new gateway component (the “dipper”) that will be controlled with a CLI. Gateway operators can be onboarded to this CLI and use it to add subgraphs to the list of supported subgraphs. Once a subgraph is in the list, the dipper will take care of finding Indexers and setting up indexing agreements with them.

The Indexer stack will be updated to be able to accept indexing agreements from the dipper when the price is above the price set by the Indexer, and up to a maximum amount of subgraphs also set by the Indexer.

Pricing between Indexer and gateway in the MVP will be based on the "amount of work done" proxy detailed above.

The dipper will initially set up agreements with 3 Indexers for each desired subgraph, as we estimate this is a reasonable number to achieve 99.9% uptime in most subgraphs.

Once the smart contracts proposed in this GIP are deployed, we anticipate adapting the indexer and gateway components accordingly.

The MVP has different trust implications than what this proposal has described so far. In particular these are:

- Indexers do the work upfront and trust that the gateway will pay them accordingly after the fact, since the gateway issues payment vouchers upon receiving PoIs and not before.
- Gateways trust that indexers are reporting the correct amount of work done, since this information is not posted on-chain and thus it's non-slashable. This is partially mitigated by the fact that the gateway can (theoretically) compare the reported work done among the pool of indexers selected and thus can weed out bad actors, but it's nevertheless a weaker mechanism than slashing.

## Detailed Specification

TODO

## Dependencies

This GIP depends on Horizon being fully functional.

## Rationale and Alternatives

When presenting this proposal, we often hear suggestions for alternative, and possibly simpler, approaches to payments, for instance:

- Could the Gateway just pre-pay a fixed amount per subgraph?
- Could Indexers just charge a specific (Indexer-determined) amount for each subgraph in arrears?

However, the design of The Graph protocol is, and must be, adversarial. When designing incentives, participants are generally considered rational, but the protocol must be safe against byzantine participants that may derive profit from malicious actions in ways that cannot be predicted within the protocol. The typical cryptoeconomic approach for this is to establish a cost-of-corruption, so that malicious actors are punished and participants can quantify the risk in each interaction.

In the simple examples mentioned above, there are a few potential attack vectors to consider: the Gateway could refuse payment after the fact, or the Indexer could take the money and run, or the Indexer could try to charge a ridiculous amount after the Gateway has promised payment.

Considering this, the protocol is designed to minimize the need for trust, in two directions:

- Gateways should not need to trust that Indexers will do their job or provide correct data. Otherwise, Gateways would only query trusted Indexers (e.g. Indexers with a reputation or direct relationship with the Gateway, which could lead to centralization or a winner-take-all Indexer.)
- Indexers should not need to trust that Gateways will provide payments after completing their job. Otherwise, Indexers would only serve trusted Gateways to ensure payment. For example, Indexer would only serve reputable Gateways, which could lead to Gateway centralization and force consumers to access the network through the set of trusted Gateways.

The proposed payment system minimizes trust in both directions.

When combined with the cryptoeconomic security from stake and appropriate fault-detection, the protocol can establish a cost-of-corruption, or a cost to faults in general. With this in place, a new participant entering the Network, for example as a Gateway or even as a Consumer running their own Gateway, can use the Network to its full potential. Any trusted relationship enshrined at the protocol level would directly jeopardize the Network's censorship-resistance and resilience, as it could require participants to interact with only a handful of trusted participants.

These principles are evident in the original protocol design and in the ongoing efforts to roll out the Timeline Aggregation Protocol (see GIP-0054). The design of the payments mechanism in this GIP is designed to preserve these principles, while allowing maximum flexibility for Gateways to provide alternative payment systems for data consumers or subgraph developers.

Moreover, the payment mechanism for this GIP is designed in a way that allows extension to arbitrary data services. The initial implementation is subgraphs-specific, but the Escrow and voucher system can allow arbitrary subscription-like payments, for any verifiable use case where an initial agreement leads to periodic payments of up to a certain amount for work that can be represented by an onchain proof or commitment.

It works as follows:

- **Gateways** will add funds to an **Escrow** account specific to the Indexer they want to interact with. This prevents double spending (the Gateway could otherwise pay itself from those same funds before the Indexer collects payment). Funds are locked with a thawing period, so the Indexer can know that funds will be available for at least that amount of time.
- When the Gateway wants an Indexer to perform indexing work, it sends a signed **Voucher**. The Voucher specifies the max cost per period, the agreed price per gas/unit of work, and the details of the work to do (i.e. which subgraph).
- The Indexer can post that Voucher onchain, which results in accepting the **Agreement**, the terms for which are encoded in the Voucher. The Voucher specifies that the Subgraph Service contract is authorized to release the payment; the Subgraph Service contract also specifies the stake-to-fees ratio. From the moment the voucher is posted onchain, the Indexer is guaranteed to be able to collect payment if they post the PoI (indexing up to the current Epoch) within the specified time and meeting all necessary conditions (e.g. provisioned stake).
- The Gateway doesn’t need to sign anything else to release further payments, they just need to keep the Escrow topped up. The Indexer can stop working if they don’t see sufficient funds in the Escrow. This ensures that Gateways cannot avoid payment after the Indexer has done the work.
- The Subgraph Service releases payments conditional on Indexers posting PoIs - this ensures that Indexers cannot collect payment without doing the work (and if they present an invalid PoI, they can be slashed). The voucher release contract can ensure that only up to the maximum amount is released, but the Subgraph Service can release the exact amount based on the agreed pricing (e.g. price per gas times the reported indexing gas used).

**This voucher release mechanism is generic and can be used for other data services:** it validates that the caller is the approved data service contract, that the token amount is within the maximum value specified by the voucher per period, and that the right amount of time has passed. The data service specific aspects e.g. PoI validity, gas/pricing, etc. are defined at the data service contract level. Moreover, this contract can be immutable, so payers are protected even if the data service contract is upgraded.

It is important to note that we cannot expect Indexers to take on risk, e.g. by agreeing to a fixed price for a specific subgraph (instead of per unit of work/gas). The cost of indexing each subgraph is unknown until the subgraph has been indexed, so if the price was fixed a rational Indexer would simply stop indexing when the subgraph becomes unprofitable, forcing the payer to start over. Tying payments as much as possible to the verifiable cost of performing the work is the best way to prevent Denial of Service scenarios, just like with gas in Ethereum.

## Copyright Waiver

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
