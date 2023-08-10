---
GIP: 0058
Title: Indexing Fees
Authors: Justin Grana, Howard Heaton
Created: 2023-08-08
Updated: 2023-08-08
Stage: Draft
Discussions-To: <The forum post where discussion for this proposal is taking place.>
Category: curation
Implementations: NA
---

# Abstract
 A  key step to creating unstoppable dapps is  creating a system whereby Indexers and data consumers can coordinate to get subgraphs indexed. The current coordination mechanism, curation with bonding curves, is rife with inefficiencies.  This document provides a detailed description and rationale for indexing fees via a new mechanism that would replace curation to enable efficient coordination among Indexers and developers. With indexing fees, Indexers publicly post their price *per unit of subgraph gas* and consumers (or someone acting on behalf of the consumer), choose Indexers to perform indexing at the posted price.  Security of the payments is ensured through on-chain collateralization contracts.  Importantly, while individual consumers can directly choose Indexers, this mechanism enables general automated Indexer selection algorithms that are essential for scalability and can be abstracted from end consumers. 

# Executive Summary

**Problem:** The Graph’s current curation mechanism is inefficient for several reasons.

1. Subgraphs are heterogenous in their cost to index, leading to uncertainty in Indexer profits.
2. Indexer payments are uncertain and volatile due to staking decisions in disparate parts of the network.
3. There is a noisy relationship between curation signal and the quality of indexing service for a subgraph.

**Proposed Solution:** Indexing fees are collected via a new mechanism that proceeds as follows.
1. Indexers publicly post a price per unit of work to index a subgraph.
2. A consumer chooses which Indexers it would like to contract to index the subgraph.
(More likely, an algorithm acting on behalf of the consumer can choose for them.)
3. The selected Indexers index the subgraph and then submit a POI that verifiably and deterministically states
how many units of work it took to index the subgraph.
4. The consumer pays the Indexer according to the posted price and the units of work.

**Benefits**: In addition to its simplicity, indexing fees yield the following favorable properties.
1. Indexer compensation depends directly on how resource intensive it is to index a subgraph.
2. Indexer revenue per unit of work is perfectly predictable and is not volatile.
3. The relation between consumer payment (per unit of work) and number of Indexers is perfectly predictable. This makes the relationship between consumer payment and quality of service more transparent.

Although indexing fees yield a radical departure from curation, it adds clarity and predictabilty to the protocol by limiting
uncertainty. This will enable an efficient and scalable marketplace for indexing services.

# Motivation

Decentralized and permission-less data services provide the robustness and censorship resistance required to build unstoppable apps. However, the pseudo-anonymous nature of a permission-less decentralized environment presents new challenges, especially in regard to novel attacks such as sybil attacks (i.e. where participants are able to improve their outcomes by splitting their identities) and ensuring parties are able to establish sufficient trust. To address these concerns, decentralized protocols (including The Graph), aim to implement mechanisms to deter behavior that is not in the spirit of the protocol while maintaining an efficient market. The Graph’s current curation system is meant to serve that purpose.  

Today, coordination for indexing uses bonding curves. Each subgraph has a bonding curve on which curation shares are minted when a user deposits GRT. Each subgraph’s bonding curve is unique. Indexers are incentivized to index subgraphs with high signal (i.e. much GRT) so that they can collect indexing rewards. Under this mechanism, the rewards Indexers get on a particular subgraph fluctuate depending on the signal of other subgraphs and the staking decisions of other Indexers; this leads to volatility and uncertainty for Indexers and unpredictable quality of service for consumers. 

# Proposed Solution

To address the shortcomings of curation, we propose using indexing fees whereby Indexers publicly post a price per unit of work for which they are willing to index subgraphs. Consumers are able to choose Indexers to contract to index their subgraphs; often, we expect an algorithm with automate this process on behalf of consumers. The selected Indexers index the subgraph and submit a POI that verifiably and deterministically states how many units of work it took to index the subgraph. The consumer pays the Indexer according to the posted price and the units of work. 

A key departure from curation is that, with indexing fees, the incentive to index does not come from indexing rewards, but rather from a direct transfer of GRT from a consumer to an Indexer. Today, Indexers are primarily compensated for indexing activity via indexing rewards. With Horizon, indexing rewards will be used to subsidize protocol security in the form of payments to collateral. Therefore, indexing rewards will no longer be used to incentivize indexing activities. Instead, the incentive to index is provided through payments that flow directly from the consumer to the Indexer. 

To fully document and explain the proposed mechanism, the remainder of this document proceeds as follows. First we identify key terms and then desired properties for a mechanism. The main details of indexing fees follows with an illustration on how they meets the desiderata. The next section  is dedicated to an example automated Indexer selection algorithm. Frequently asked questions are addressed in following section, which are followed by the conclusion. An appendix (to be linked) includes an important analysis of subgraph gas and how it relates to computational resources required to index a subgraph.

## Terms and Background
**Indexers:** Indexers are node operators in The Graph Network that stake Graph Tokens (GRT) in order to provide indexing and query processing services. Indexers earn query fees and indexing rewards for their services. 

**Consumer.:* The core users of data services are referred to as consumers. Specifically, the consumer is the one that receives a benefit (and thus is willing to pay) for data services. In some cases, this may be the developer. In other cases, a consumer may want to leverage the subgraph of another developer. In general, the consumer is the party that interacts with Gateways/Indexers for the indexing services.

**Subgraph Gas.** There are two notions of subgraph gas: sync gas and storage gas. Storage gas is simply the size of entities stored in the database, which correlates with the size of a subgraph on disk. Sync gas captures the amount of compute and disk operations required to sync a subgraph, write it to disk, and perform network operations (e.g. JSON RPC). Both storage gas and sync gas are verifiable quantities.  A detailed relationship between subgraph gas and compute resources is given in Appendix A.

**Horizon**. Graph Horizon introduces a general purpose scheme for securing bilateral transactions in The Graph ecosystem. The core aim is to establish trust via on-chain means; to do this, Horizon utilizes an escrow-like service with the capability of penalizing bad behavior (via burning tokens of the party at fault). Horizon “establishes trust” via economic incentives from smart contracts and allows “freely evolving services” due to a fundamental shift in how governance operates. With respect to the scope of this document, the only essential information to know about Horizon is that it enables on-chain agreements to be formed for coordinating procurement of services and arbitration. 

**Agreement**: An agreement is used to identify the terms of service between an Indexer and a consumer. For example, a caricature agreement might be "Indexer will index a subgraph within 2 days for 1 GRT per unit of subgraph gas." Agreements are often formalized via smart contracts; however, an off-chain agreement may also be formed between a consumer and a Gateway acting on the consumer’s behalf.


## Desiderata
The desiderata — desired properties of a new mechanism— reflect addressing the issues with the current curation mechanism. Specifically, once agreements are reached, the Indexer should have low uncertainty in regard to its profit and revenue, and that certainty should not wane with time. On the consumer side, once agreements are reached, their price paid and received quality of service should both be predictable and steady over time. Reducing these uncertainties yields the desirable higher level properties simplicity and efficiency. Of course, this must also be done in a sybil resistant manner to preserve the robustness and censorship resistance benefits of a decentralized platform. The remainder of this section details each of the desiderata.

**Low Uncertainty – Indexer Revenue**
Indexers should be able to forecast how much revenue they will achieve by indexing a subgraph. Uncertainty may arise when indexing revenue is influenced by actions of other participants (e.g. curators adjusting signal with bonding curves, Indexers unexpectedly allocating a large amount of stake on a subgraph).

**Low Uncertainty – Indexer Profitability**
Indexers cannot know the total cost to index subgraphs beforehand. To ensure Indexers are profitable, it follows that there must be a way to compensate them proportional to their incurred indexing costs.

**Low Price Volatility**
Even if an Indexer knows the price today, how much payments fluctuate over time for indexing services can be complicated to reason about. This similarly affects developers when making plans to use data services for their dapps. Cognitive load and pricing efficiency can typically be improved by reducing volatility.

**Low QoS Uncertainty** 
The consumer can easily see how much it needs to pay to achieve a given quality of service. Furthermore, the consumer can have the flexibility of obtaining a high quality of service by selecting several competent Indexers or selecting fewer but more effective Indexers.

**Sybil Resistant**
A key property of blockchains is identities can be easily duplicated across several wallet addresses. To ensure developers have their dapps serviced by multiple Indexers (e.g. for robustness to failure of any single Indexer), developers must be able to ensure, with reasonable confidence, they are able to attract
distinct Indexers to index their subgraphs.

**Efficiency**
The mechanism (approximately) maximizes the number of mutually beneficial transactions.

**Simple**
We aim to create a simple market. We consider this along two dimensions: actions and consequences are simple to understand (low cognitive overhead) and tasks are simple to perform (efficient workflows).

# Proposal

### Sequence of Events and Details
Under the indexing fees mechanism, the order of events can be summarized as follows. We then provide a detailed description of each step.

**Step 1:** Indexers post prices per unit of subgraph gas with a sync-speed warranty per unit of subgraph gas.

**Step 2:** Consumer (or agent acting on behalf of consumer) selects Indexers.

**Step 3:** Consumer and Indexers both enter an on-chain agreement that specifies:
1. Required Indexer collateral.
2. Terms of agreement (price per unit of subgraph gas, correctness warranties, slashable events, arbitrating party, collateral thawing period, etc.). The contract is enforced when the consumer executes a trasnaction to accept the terms and deposits funds to pay for the indexing services and the Indexer deposits the time-locked collateral.

**Step 4:** Indexer indexes the subgraph.

**Step 5:** Indexer posts POI.

**Step 6:** Dispute period begins.
1. Within the thawing period defined in the agreement the consumer can choose to raise a dispute that, if the arbitrator determined to be valid, would result in the Indexer being slashed according to the terms of agreement.
2. The Indexer cannot withdraw its payment nor its collateral until the dispute period ends.

**Step 7:** Indexer retrieves any funds (collateral and payment) after the dispute period.

**Step 8:** Indexer periodically reports ongoing sync gas and ongoing storage gas and withdraws funds from the contract accordingly.

**Step 9:** When the contract is over, the Indexer can withdraw collateral for the sync speed warranty.

### Step 1 — Indexers Post Prices
Simply stated, Indexers will submit prices to the Gateway that it charges to index a subgraph. This is similar to the current process for the Indexer selection algorithm for choosing queries. Importantly, these prices are in terms of metered computational units of work and also contain a sync speed warranty. The motivation for metered pricing is to account for subgraph heterogeneity and to incentivize dapp developers to optimize their subgraphs. One of the main challenges with previous versions of curation was that payment to Indexers did not depend on the amount of work (compute, storage, etc) required to index a subgraph. With indexing fees, Indexer pricing is a function of computational resources required to index a subgraph. As an initial protocol design choice, prices will be in terms of “subgraph gas,” an approximation of computational resources needed to index a subgraph. See section A for more details on the relationship between subgraph gas compute costs.
The sync-speed warranty provides a channel for Indexers to credibly differentiate the quality of their hardware and thus earn more revenue by deploying better hardware.  By posting sync-speed warranties per unit of compute (subgraph gas), Indexers with better hardware can post faster sync-speed warranties and thus charge a higher price. For more details on how different hardware configurations impact sync-time, see Appendix A. Furthermore, Indexers can post multiple prices with multiple sync-speed warranties if they want to offer
different levels of service at different prices.

### Step 2 — Indexer Selection
Indexer selection can happen in two ways 1) directly by the consumer or 2) through an automated Indexer selection algorithm that optimizes for consumers preferences (e.g. quality of service). An automatic Indexer selection algorithm is discussed in detail below. This section focuses on the case where consumers choose their own Indexers.  Under manual Indexer selection, a consumer is presented with a menu of Indexers and relevant features (price per unit of gas, geographical location, quality of service statistics, etc.). Note quality of service stats are trusted data (in the case of Gateway usage) and untrusted if manually collected. The consumer can then select one or more Indexers based on its preference to index its subgraph. Importantly, at this point neither the consumer nor the Indexer have entered into an agreement through smart contract. The selection step notifies Indexers that there is a smart contract that if they choose to enter would make them eligible for a payment upon successfully indexing of a subgraph.

### Step 3 — Consumer and Indexer Enter Agreement
After the Indexer is notified, the consumer and the Indexer must both opt into the agreement. The consumer opts into the agreement by depositing GRT into a smart contract that would then be transferred to the Indexer upon successful indexing. The Indexer deposits collateral into the smart contract that would be slashed if indexing did not happen according to the agreements parameters.  The smart contract specifies several parameters of the agreement including the price per unit of subgraph gas, the required amount of Indexer collateral, the maximum amount of gas the Indexer should spend indexing the subgraph, slashable events and parameters (how much an Indexer should be slashed for not meeting their sync-speed warranty, for example), arbitrating entity and collateral thawing period. The contract is uses the collateralization of the Horizon Core contract and can contain general terms. 

### Step 4 — Indexing Occurs
This step is identical to how the Graph functions today.

### Step 5 — Indexer Posts POI
This step is identical to how the Graph functions today

### Step 6 — Enter Dispute Period
After the Indexer submits a POI the dispute period begins. Within this period, the consumer has the option to submit a dispute to the arbitrator (specified in step 3). If the arbitrator determines that the terms in step 3 were violated by the Indexer, the Indexer is slashed according to the agreement. The portion of slashed GRT that is returned to the consumer versus the portion that is burned is specified in step 3. During the dispute period, both the consumer’s funds and the Indexer’s funds are frozen in the contract.

### Step 7 — Indexer Retrieves Funds
After the dispute period, the Indexer retrieves its entitled funds (both collateral and payment) that remain after any slashing events. Of course, because the collateral and slashing provides adequate incentive for Indexers to meet the terms in step 3, step 7 will often consist of the Indexer withdrawing its full collateral and the consumer’s payment for indexing services.

### Step 8 — Continual Syncing and Payment
Since subgraphs incur additional syncing and continual , the Indexers post the incremental sync gas and disk space required to fully sync the subgraph. After a dispute period and assuming no slashable events, the Indexer can withdraw additional funds as compensation for the additional syncing resources.

### Step 9 - Indexer Retrieves Remaining Funds
Although a correctness warranty can be reclaimed at the end of a dispute period (e.g. whenever a PoI is submitted + fixed time,) the sync speed warranty can only be reclaimed at the end of the agreement. A PoI may be required periodically to checkpoint syncing, but submitting a PoI and collecting payment for the subgraph synced so far does not obviate the need for the sync speed warranty to continue for the subgraph yet to sync past that checkpoint.

## Desiderata Revisited
Given the full description of indexing fees in the previous section, it is now possible to show how the new mechanism meets the desiderata:

**Low Uncertainty – Indexer Revenue**
The Indexer’s price (per unit of “work”) is perfectly predictable. It is simply the posted price. This is not affected by the behavior of other Indexers.

**Low Uncertainty – Indexer Profitability**
Because Indexers post prices per unit of gas, they can charge more for more expensive subgraphs, thus reducing profitability uncertainty.

**Low Price Volatility**
The price an Indexer receives is fixed and does not change over time as a result of other Indexers' staking decisions.

**Low QoS Uncertainty**
Each Indexer’s price and sync-speed warranty are publicly posted. Therefore, if a consumer knows the size of its subgraph, it can compute exactly how much it will cost for each additional Indexer.

**Sybil Resistant**
Because consumers select Indexers, they will select Indexers that they know to be distinct. Therefore, Indexers have no incentive to split their identity.

**Efficiency**
It is documented that in a general class of large markets, posted prices are the most efficient pricing mechanism.

**Simple**
The new indexing fees mechanism mirrors standard market mechanisms: Sellers post a price and consumers select what to buy based on product quality and cost.

## Automated Indexer Selection

Although consumers may manually choose their own Indexers, those seeking a simpler experience could use an automatic Indexer selection algorithm. Such an algorithm may abstract away the need for consumers to screen individual Indexers. Instead, after a consumer provides their desired a) number of Indexers and b) sync speed preferences (high, medium, low), an automated Indexer selection algorithm can report the price and select the Indexers on behalf of the consumer.

An example algorithm works as follows. First, Indexers are sorted based on their sync speed warranties and then categorized into high, medium or low sync speed (based on pre-defined thresholds). Indexer’s that are dominated, i.e. have a higher price for a lower sync speed warranty than some other Indexer — are removed from selection. The number of requested Indexers in the tier are selected sequentially.

The first Indexer within the chosen sync speed tier is selected at uniform random. To provide robustness and guard against sybil attacks, the subsequent Indexers are selected at uniform random as long as their quality of service is not sufficiently correlated with previously selected Indexers. In other words, robustness and redundancy relies on Indexer performance being uncorrelated; this ensures, if one Indexer fails, there is no loss in service. By selecting Indexers that are uncorrelated, the automated algorithm ensures robustness. Furthermore, sybil
identities are also likely to have a correlated quality of service, and so selecting Indexers with low correlation protects against sybil attacks.

This example algorithm is admittedly crude and imprecise. However, it already gives consumers greater control over their quality of service than they have today and, thus, makes the protocol more efficient. Nevertheless, there is nothing preventing other agents from developing more sophisticated Indexer selection algorithms, incorporating both on-chain and off-chain data to improve consumer experience.

## Frequently Asked Questions
In this section, we address some of the frequently asked question that arise when replacing curation with indexing fees.

**How do Indexers know which subgraphs to index?**
They can be notified that they have been selected to index a subgraph at their posted price.

**What is an Indexer’s incentive to index the subgraph if they are selected?**
They will receive a payment equal to their posted price multiplied by the subgraph gas used to index.

**Once selected, what actions must an Indexer take before making the indexing agreement valid?**
An Indexer must opt-in to the agreement. This gives an Indexer the opportunity to verify the subgraph is
available before accepting the indexing contract.

**Can an Indexer post different prices for different subgraphs?**
No, the price per unit of gas is network-wide. This is subject to change in the future.

**Can an Indexer post a price based on a quantity other than subgraph gas?**
Initially no, though this is subject to change.

**Does this impact query fees and how Indexers are selected to serve queries?**
No. Indexing fees only pertain to indexing and not serving queries.

**Can an Indexer that is not selected to index a subgraph still index the subgraph and compete for queries?**
Yes! Indexing fees does not preclude any Indexer from indexing a subgraph, it only provides extra incentive.

**How much gas will a subgraph consume?**
This is impossible to know exactly before indexing. You can find some statistics for common subgraphs that relate subgraph properties to gas costs in Appendix A.

# Summary
Indexing fees are an improvement over the current curation mechanism. These add transparency and predictability which begets simplicity and efficiency for both consumers and Indexers. Together with Graph Horizon, indexing fees are general enough to allow for future services on the network and to scale, independent of issuance.  While it may seem like a radical departure from current curation, indexing fees simply use a posted prices mechanism that is ubiquitous for large markets.

# Detailed Specification
The detailed specification of the smart contracts for this GIP will be added at a later time.

# Copyright Waiver

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
