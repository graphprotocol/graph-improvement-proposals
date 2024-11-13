---
GIP: "00XX"
Title: A New Proposal for Indexing Payments
Authors: Matias <matias@edgeandnode.com>, Pablo <pablo@edgeandnode.com>
Created: 2024-10-21
Updated: 2024-10-21
Stage: Proposal
Discussions-To: <TODO>
Category: Protocol Logic
Depends-On: GIP-0066, GIP-0068
Replaces: GIP-0058
---

## Abstract

This GIP describes **Indexing Payments**, a mechanism by which a network participant (commonly a gateway) can incentivize a candidate indexer to service a subgraph of interest. It details a target trust-minimized mechanism that relies on Graph Horizon, while also introducing a minimum viable product based on the current protocol and off-chain components (gateway + indexer stacks). This GIP is an alternative, simplified proposal to [GIP-0058](todo-link).

## Motivation

Curation has generally been perceived as a source of friction for Developers that may only want to have subgraphs indexed to power their decentralized applications. It requires locking up certain amounts of GRT, and the necessary amounts might vary depending on curation on other subgraphs. While the quality of service provided on the network is generally high, Indexers collect rewards for posting Proofs of Indexing independently to the query service they provide, so Developers are not guaranteed a certain level of service for the amount of curation they put into the system.

For Indexers, there is no clear relationship between the Indexing Rewards on a subgraph and the amount of work required to index it. Attempts at defining better curation and rewards models to pay for indexing have so far been unsuccessful, which presents a need for a simpler mechanism where parties interested in indexing a subgraph can simply pay for it through the protocol. However, when building such a mechanism, it is imperative to preserve the protocol's trust-minimization and decentralization principles.

[TODO]: # "I believe I can improve on this section"

## Prior Art

GIP-0058 presents a previous version of a very similar proposal. This GIP re-introduces the same concepts, but incorporates the latest approach in Graph Horizon, and simplifies the proposal with an off-chain MVP while clarifying that the actor engaging in **Indexing Payments** is generally a Gateway running automated indexer selection.

[TODO]: # "filecoin deals?"

## High-Level Description

We are proposing a new coordination mechanism to get subgraphs serviced to a sufficiently performant level. Such a coordination mechanism is necessary to jump-start, and sometimes maintain, the desired level of subgraph servicing. One such case is when a subgraph is being upgraded.

[TODO]: # "move to motivation?"
[TODO]: # "Add more cases in which the coordination mechanism is required?"
[TODO]: # "describe what subgraph servicing and QoS is?"

Attaining a target subgraph service level has two main concerns that we need to address:

1. The desired service level can only be defined and assessed by the data consumer.
2. The cost of said service level is unknown a-priori, and it depends on the query performance level as well as the cost of indexing the subgraph.

To solve for the former, we propose that the gateway is the one coordinating with the indexers. Gateways can define and assess QoS on an ongoing basis and decide which indexers continue to work with based on that.

To solve for the latter, we propose that indexers are paid proportionally to the amount of work they do to index the subgraph. This amount of work is reported by indexers and thus needs to be verifiable.

[TODO]: # "explain why we need to slash for incorrect work done."

In particular we propose that the coordination mechanism is underpinned by an **Indexing Agreement** to be drafted by the gateway, shared with a subset of indexers of its choosing and subsequently approved by some of those indexers. These agreements are designed to allow for indexers to work and collect for their work without having to go back to the payer. They are self-contained in that regard. Furthermore, they specify the following:

1. the subgraph to be indexed
1. the data service that will initiate payment collection
1. a deadline by which the agreement must be accepted
1. an end date to the agreement
1. a maximum amount payable for the initial indexing.
1. a maximum amount payable for the ongoing indexing. The ongoing indexing is the indexing that happens between the last collection and the present.
1. the maximum amount of time that can elapse between collections.
1. a price per unit of work done.

This agreement is to be upheld by protocol smart contracts, leveraging horizon primitives. This means that payers will have to escrow funds similarly to what's done for query fees, but will use a collector appropriate for this type of arrangement. Between the smart contracts and the escrow, indexers are guaranteed that if they accept an indexing agreement, they'll be compensated fairly, without the need to trust the agreement's counterpart.

At the same time, agreements are cancellable by either party at any time, with some limitations put in place to allow indexers to collect for all work done up to that point.

Putting all of this together the solution consists of the following:

- Indexers disclose publicly their price per unit of work.
- Gateways send indexing agreements to indexers they want to work with.
- Indexers accept (on-chain) agreements that meet their criteria and get to work indexing.
- On a regular basis:
  - Indexers collect for their work, by posting on-chain the original agreement, a POI and the amount of work done for that period.
  - Gateways monitor the QoS per open agreement and assess if they need to cancel (on-chain) and/or replace any of them.

This solution aims to:

- compensate indexers for the exact amount of work they do.
- allow indexers to gracefully exit the agreement if it can't be fully fulfilled.
- cap the amount payers will spend on indexing. Though they might spend but get no service back if the cap is too low.
- incentivize sufficient QoS by awarding agreements to indexers that provide it.

[TODO]: # "probably missing more desired properties"

### Indexing Indexer Selection Algorithm (IISA)

As mentioned above, gateways get to choose who they work with. As a way to do this we propose an IISA is used. It considers Indexer Quality-of-Service, available stake (with delegation), and other parameters to find the optimal balance between QoS, load-balancing, economic security and price.

[TODO]: # "detail the IISA?"

### Arbitration and disputes

Since collections are tied to "amount of work done" and this is reported by the indexers themselves, there needs to be a mechanism to punish malicious reporting. We propose here to use slashing. For this to be fair we need the "amount of work done" to be verifiable by an arbitrator in the case of a dispute.

### Amount of work done

As an initial iteration of the "amount of work done" proxy we propose to use indexed blocks and subgraph entities. In particular the amount to be collected by indexers shall be equal to:

- A price per amount of blocks indexed (with a different price for each chain). These are charged incrementally, i.e. every time a POI is posted, the indexer collects a payment for the new blocks indexed since the last POI.
- A price per amount of entities stored by the subgraph (same price for each chain). These are charged on a recurring basis per unit of time, for the total number of entities on the subgraph at the time the POI is posted times the time since the last POI.

This approach is vulnerable to some forms of abuse and may also lead to denial-of-service scenarios (e.g. malicious users deploying subgraphs that are super expensive to index but get a very small entities count).

We anticipate needing to perform research into alternative approaches in the near future. We theorize that "amount of work done" may be modeled as a **subgraph gas** unit down the line, similar to ethereum gas.

### Minimum Viable Product

While this proposal mostly focuses on the full Graph Horizon backed implementation, we are also developing an MVP that does not require smart contracts.

In this MVP, we will roll out a new gateway component (the “dipper”) that will be controlled with a CLI. E&N users (e.g. the BD team) can be onboarded to this CLI and use it to add subgraphs to the list of supported subgraphs. Once a subgraph is in the list, the dipper will take care of finding Indexers and setting up indexing agreements with them.

The Indexer stack will be updated to be able to accept indexing agreements from the dipper when the price is above the price set by the Indexer, and up to a maximum amount of subgraphs also set by the Indexer.

Pricing between Indexer and gateway in the MVP will be based on the "amount of work done" proxy detailed above.

The dipper will initially set up agreements with 3 Indexers for each desired subgraph, as we estimate this is a reasonable number to achieve 99.9% uptime in most subgraphs.

Once the smart contracts proposed in this GIP are deployed, we anticipate adapting the indexer and gateway components accordingly.

This MVP has different trust implications than what this proposal has described so far. In particular these are:

- Indexers do the work upfront and trust that the gateway will pay them accordingly after the fact, since the gateway issues payment vouchers upon receiving POIs and not before.
- Gateways trust that indexers are reporting the correct amount of work done, since this information is not posted on-chain and thus it's non-slashable. This is partially mitigated by the fact that the gateway can (theoretically) compare the reported work done among the pool of indexers selected and thus can weed out bad actors, but it's nevertheless a weaker mechanism than slashing.

### Generalized collector

[TODO]: # "describe new generalized collector"

## Detailed Specification

_(Optional) Include specific APIs, semantics, data structures, code snippets, etc. related to this proposal. This type of info is required for a proposal to reach the "Draft" stage._

## Backward Compatibility

_(Optional) How does this protocol impact backward compatibility? What breaking changes, if any, are included? Does the proposal have at least N-1 compatibility, or must it be deployed in a knife-edge rollout?_

## Dependencies

(_Optional) How does this proposal depend on other GIPs or other engineering work?_

## Risks and Security Considerations

_(Optional) What technical or security risks are there with implementing this proposal? How might they be mitigated or addressed?_

## Validation

_(Optional) How should this proposal be validated before being included in the protocol? Examples might include security audits, economic modeling, user research, running in testnet, etc._

## Rationale and Alternatives

_(Optional) What, if any, alternate designs or interfaces were considered while writing this proposal? Why was the current proposal chosen over any alternate designs? Justify the design decisions you made in writing this proposal._

## Copyright Waiver

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
