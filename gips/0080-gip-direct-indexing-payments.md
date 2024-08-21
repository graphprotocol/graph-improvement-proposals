---
GIP: "0080"
Title: <The title of this GIP>
Authors: Tim Fong <tim@edgeandnode.com>, Pablo <pablo@edgeandnode.com>
Created: <The date the GIP was created.>
Updated: <The date the GIP was last updated.>
Stage: <The current stage of the proposal. Specified by the author and confirmed by editors by virtue of a GIP being accepted into an editor's view of the repo.>
Discussions-To: <The forum post where this proposal is discussed.>
Category: <(Optional) The type of GIP or GRP. This field should be left blank for GRCs. Valid types are "Protocol Logic", "Protocol Interfaces", "Subgraph API", "Process", "Economic Parameters", and "Protocol Charters".>
Depends-On: <(Optional) A list of GIPs that this GIP depends on. If some other type of dependency exists, include a reference link here and an explanation in the body of the GIP.>
Superseded-By: <(Optional) A GIP that supersedes the current proposal. If this field is specified, the stage of the GIP should be "Withdrawn".>
Replaces: <(Optional) A GIP that this proposal is intended to supersede.>
Resolves: <(Optional) If this GIP was written in response to an RFP, include it here.>
Community-Polls: <(Optional) A list of URLs to community polls relating to this GIP.>
Governance Proposals: <(Optional) A list of URLs to governance proposals related to this GIP.>
Implementations: <(Optional) A list of URLs to reference implementations for this proposal.>
Audits: <(Optional) A list of URLs to audits related to this proposal.>
---

## Abstract
This supercedes GIP-0058, which introduced a protocol mechanism for Gateways to pay Indexers directly for indexing subgraphs.  This updated variation doesn’t introduce new major concepts, but does the following: describes the payment mechanism with more detail, provides a more detailed taxonomy of the participants, and provides a comparison of the Protocol versus the Product.  Enabling Indexing Fees provides an alternative alongside the current Curation model.

This GIP isn’t specifying execution timing, but introduces two high-level phases: a) an off-chain MVP; b) a post-MVP contracts-based implementation. 

## Motivation
Currently, subgraph developers use the Curation mechanism to attract Indexers to their subgraph. Curation adds friction by locking a potentially high amount of GRT, and delivers a low predictability for the required amount of GRT needed to achieve a certain Quality of Service (QoS). The friction from Curation may make it harder for developers to select The Graph over alternatives.

Moreover, the current Indexing Rewards mechanism does not reward Indexers  for serving queries with a high QoS, and does not provide Indexers with predictability on the revenue they can make for indexing and serving subgraphs.

## High-Level Description
There are two primary phases: off-chain MVP and Post-MVP contracts-based implementation.  Here are the high-level components of each phase.

### Off-Chain MVP
The MVP implementation can be done without any changes to the protocol smart contracts. Interactions between Gateways and Indexers happen through HTTP APIs, and payments are sent as a lump sum as if it were a single query fee payment, to be collected using rebates together with the query fees for the allocation.

This allows the development of the simplest possible implementation with the existing stack, though at this stage Indexers need to trust that Gateways will pay after the POI is posted. For simplicity, the MVP will use an indexing price as a function of entities and indexed blocks.

The following is a sequence diagram

![image](https://github.com/user-attachments/assets/1182cc24-9adf-4aea-969a-954b9364389f)

### Post-MVP Contracts-Based Implementation
The Post-MVP phase depends on the Subgraph Service implementation from Graph Horizon, and uses a trust-minimized payment mechanism for Indexers to collect payment from an Escrow (that is the same Escrow used by TAP for query fee payments). 

The Gateway commits to paying the specified price with a voucher, that is posted on-chain by the Indexer, and that the contracts will use as a pre-authorization. 

This allows the Indexer to collect payment directly when posting the POI, and having a guarantee that the payment will be available when the job is done (as long as the Escrow is sufficiently funded). 

At this phase, we will likely also iterate on the pricing approach and introduce a better "indexing gas" metric as an evolution to the MVP entities-and-blocks-based approach. 

### Product vs Protocol
The GIP is primarily enabling Consumers, who most likely will be Gateways, to interact via the protocol with the Indexer Network.

However, the actual Products which are the interface for Developers and End Customers, need a way to successfully build their business ontop of the Protocol.

This illustration shows the relationship between the two:

![image](https://github.com/user-attachments/assets/0415f01d-4b8f-4bf0-9579-72acbe037a51)

### What is a Product?

The Product Experience **defines the UX for the End User**, in most cases, a dapp developer who wants data that has been indexed.

A good Product experience includes as much **predictability and transparency upfront for potential costs to the paying Developer.**

However, there could be instances the Gateway has to take on risks for the sake of simplicity.

**This decision is a Product decision** made by the Studio operator.

### What is the role of the Gateway in the Product experience?

The Studio operator will often be running the Gateway, as well (although if a trusted experience is established with a third-party Studio, this could also be possible).

The Gateway, then, is like the payment gateway **and** the Indexing Indexer Selection Algorithm (IISA) for the Studio operator.  As a payment gateway, the Gateway operates more like Stripe: a utility facilitating the payment flow via escrow.  However, the IISA **could** be considered part of the product experience, a means for the Studio to potentially differentiate in the marketplace.

Could another Studio use the same Gateway?

If the Studio used an existing Studio via an API, such as a Studio-as-a-Service”, then another Studio would inherit the trust assumptions of the Gateway operator (since the Studio and Gateway are the same operator). 

Could a Studio choose to work with multiple Gateways?

Yes, there would need to be a trusted relationship. But a Studio which wanted to offer multiple data services and not operate the corresponding Gateway for each data service, they could. 

In this case, the Studio pays the Gateway (or the Gateway bill the Studio), such that a flow of funds goes via the Escrow to the selected Indexing Indexers to fulfill the Agreement obligations for Indexing Network Payment.

## Detailed Specification
### The payments mechanism

When presenting this indexing fees proposal, we often hear suggestions for alternative, and possibly simpler, approaches to payments, for instance:

- Could the Gateway just pre-pay a fixed amount per subgraph?
- Could Indexers just charge a specific (Indexer-determined) amount for each subgraph in arrears?

However, the design of The Graph protocol is, and must be, adversarial. When designing incentives, participants are generally considered rational, but the protocol must be safe against Byzantine participants that may derive profit from malicious actions in ways that cannot be predicted within the protocol. The typical cryptoeconomic approach for this is to establish a cost-of-corruption, so that malicious actors are punished and participants can quantify the risk in each interaction.

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
- The Indexer can post that Voucher onchain, which results in accepting the **Agreement**, the terms for which are encoded in the Voucher. The Voucher specifies that the Subgraph Service contract is authorized to release the payment; the Subgraph Service contract also specifies the stake-to-fees ratio. From the moment the voucher is posted onchain, the Indexer is guaranteed to be able to collect payment if they post the POI (indexing up to the current Epoch) within the specified time and meeting all necessary conditions (e.g. provisioned stake).
- The Gateway doesn’t need to sign anything else to release further payments, they just need to keep the Escrow topped up. The Indexer can stop working if they don’t see sufficient funds in the Escrow. This ensures that Gateways cannot avoid payment after the Indexer has done the work.
- The Subgraph Service releases payments conditional on Indexers posting POIs - this ensures that Indexers cannot collect payment without doing the work (and if they present an invalid POI, they can be slashed). The voucher release contract can ensure that only up to the maximum amount is released, but the Subgraph Service can release the exact amount based on the agreed pricing (e.g. price per gas times the reported indexing gas used).

**This voucher release mechanism is generic and can be used for other data services:** it validates that the caller is the approved data service contract, that the token amount is within the maximum value specified by the voucher per period, and that the right amount of time has passed. The data service specific aspects e.g. POI validity, gas/pricing, etc. are defined at the data service contract level. Moreover, this contract can be immutable, so payers are protected even if the data service contract is upgraded.

It is important to note that we cannot expect Indexers to take on risk, e.g. by agreeing to a fixed price for a specific subgraph (instead of per unit of work/gas). The cost of indexing each subgraph is unknown until the subgraph has been indexed, so if the price was fixed a rational Indexer would simply stop indexing when the subgraph becomes unprofitable, forcing the payer to start over. Tying payments as much as possible to the verifiable cost of performing the work is the best way to prevent Denial of Service scenarios, just like with gas in Ethereum.

### Terms and Definitions

**Curation**: Curation requires Consumers to stake a certain amount of GRT per subgraph to signal Indexers to provide indexing services.  However, because the strength of the signal depended upon the strength of other signals using GRT, which also had variability, this became a hard mechanism for developers with many subgraphs.

**Indexers:** Indexers are node operators in The Graph Network that stake Graph Tokens (GRT) in order to provide indexing and query processing services. Indexers earn query fees and indexing rewards for their services.

**Indexing Network Payments:**  this represents the payments to Indexers upon verifying successful indexing.  Typically will occur between a Gateway and Indexers selected by their IISA.

**End Consumers:** The core users of data services via Products built onto the Protocol are referred to as Consumers. Specifically, the consumer is the one that receives a benefit (and thus is willing to pay) for data services. In some cases, this may be the developer. In other cases, a consumer may want to leverage the subgraph of another developer. In general, the consumer is the party that interacts with Gateways/Indexers for the indexing services.

**Consumer**: This is referring to one who interacts directly with the Protocol for indexing services.  Often a **Gateway** (see below).

**Gateway:** a permissionless entity which can select an Indexer and pay for Indexing; often associated with a **Studio**

**Studio:** the primary "Product" that End Customers or developers interact with to pay for Indexing and to query Indexed Subgraphs

**Indexing Gas or Unit of Work:** a concept still being defined that will be used to measure the work required to deliver the indexing.  This is likely a combination of compute, storage, and network related “work” needed to be performed by Indexers providing the indexing service.  This could potentially vary by chain.

**Indexing Gas:** this will be the posted “price” by an Indexer in order to perform indexing services based on the unit of work..

**Horizon Escrow:** to trust-minimize the system, as part of the Horizon roll-out, the Gateway will place payments from the Consumer into Horizon Escrow.  The Indexer will be paid from Horizon.  However, there will be a “trusted” version released without Horizon Escrow.

**Agreement**: An agreement is used to identify the terms of service between an Indexer and the Gateway.  The negotiations typically take place off-chain, and formalized on-chain.  This Agreement can occur between a Gateway and multiple Indexers, and has some concept of cost.  Typically encoded as a **Voucher**.

**Indexing User Payments:** This is the amount that the End User pays for Indexing.  For example, a Developer using a Studio would make this payment, in either credit card or via GRT  to the Gateway.  This mechanism is off-chain, and is considered part of the **Product** experience which we will describe below.
