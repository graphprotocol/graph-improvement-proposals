---
GIP: 0068
Title: Subgraph Service - A subgraph data service in Graph Horizon
Authors: Tomás Migone <tomas@edgeandnode.com>, Pablo Carranza Vélez <pablo@edgeandnode.com>, Miguel de Elias <miguel@edgeandnode.com>
Created: 2024-06-02
Updated:
Stage: Draft
Discussions-To: https://forum.thegraph.com/t/<xxxx>
Category: Protocol Logic, Protocol Interfaces, Economic Parameters
Depends-On: GIP-0066
Implementations: https://github.com/graphprotocol/contracts/pull/944
Audits:
---

# Abstract

This GIP describes the Subgraph Service, a data service designed to work with Graph Horizon that supports indexing subgraphs and serving queries to consumers. We also propose a safe path for indexers to transition from managing their subgraph operation on the current staking contract to the new Subgraph Service contract without service disruption. Note that this GIP depends on the changes proposed by [GIP-0066](https://github.com/graphprotocol/graph-improvement-proposals/blob/main/gips/0066-graph-horizon).

# Motivation

With the introduction of Graph Horizon we proposed transforming the protocol layer of The Graph Network into a data services protocol, expanding it’s capabilities horizontally to support more use cases and extracting subgraph functionality from it’s core. This GIP introduces how subgraphs would work within this data services protocol.

# Prior Art

The design of the Subgraph Service is based on the original implementation of subgraphs functionality in The Graph. The Subgraph Service presents a new implementation of the ideas that have been battle tested for almost 4 years, adapted to work with Graph Horizon and following the Data Service framework specification. We suggest reading GIP-0066 and becoming familiar with the changes proposed there before continuing. 

# High-Level Description

The main purpose of the Subgraph Service contract is to allow indexers to collect payment for the service they provide which is indexing subgraphs and serving subgraph queries. In order to do so indexers need to provide stake as collateral in the form of a Graph Horizon provision; funds in this provision are subject to being slashed by a dispute mechanism in the event the work they provided is found to be incorrect or fraudulent. A quick note on terminology, the term “indexer” has historically been used to refer to the entity providing subgraph indexing services. In the context of the Subgraph Service this term is used interchangeably with “service provider” which is the generic term used by Graph Horizon.

The Subgraph Service’s main purpose is to facilitate on chain coordination of subgraph indexing and serving operations in accordance with the flows described by the Data Service framework. These can be summarized as:

- Allow indexers to register in the Subgraph Service as service providers
- Allow indexers to signal their intent of starting (or stopping) indexing a subgraph
- Allow indexers to collect payment for the service they provided. There are two types of work an indexer can perform:
    - Indexing a specific subgraph deployment, for which the indexers receive ***indexing rewards***.
    - Serving queries for an indexed subgraph, for which the indexers receive ***query fees***.
    - In the future indexers will also receive ***indexing fees*** for indexing work performed for specific agreements. A separate GIP will focus on those changes.
- Allow arbitrators to slash an indexer if required. This could be eventually replaced for a more automated verifiability mechanism.

For the most part these flows behave in a very similar manner than the current ones on the staking contract. There are a few key differences however so we’ll next provide an overview of the changes.

### Service registration

Before an indexer starts indexing subgraphs they need to register on the Subgraph Service, before Horizon they would register on a special contract, the Service Registry, which will now become deprecated. When registering on the Subgraph Service, the indexer needs to provide metadata associated to their operation that helps with discoverability (like the URL where they can be reached at) and they must have previously provisioned stake to the Subgraph Service using Graph Horizon’s primitives, i.e. created a provision. The Subgraph Service will validate the provision meets its requirements and register the indexer which will now be ready to start indexing subgraphs.

The following are the requirements for the provision:

- The provisioned amount must be over a minimum amount established by the Subgraph Service, that will be 100k GRT as in the current protocol.
- The provision’s verifier must be the address of the Subgraph Service contract. This allows the Subgraph Service to slash the provision if required (the decision will actually be delegated to a dispute management contract).
- The provision’s thawing period must be equal to the dispute resolution period established by this dispute management contract. This allows enough time to raise and resolve disputes before the indexer can remove the stake from the protocol.

### Indexing a subgraph

Next, an indexer would want to signal the network they are indexing a subgraph. For that they need to “allocate” a portion of their available stake to the subgraph the intend to index (thawing stake does not count as available). The allocated amount is used to calculate indexing rewards. However, same as with the current version of the protocol, the entire provisioned stake will act as economic security for the indexing work (it will be slashable).

**Allocation management**

The Subgraph Service takes on the concept of allocations from the original protocol with a few tweaks. Allocations are still a one-to-one mapping of stake to subgraph deployments but their lifecycle is now vastly simplified. Indexers create allocations when they start indexing a subgraph but they only close them when they decide to stop indexing the subgraph, essentially allocations are now long-lived. To fully support indexer operations, new interfaces are introduced:

- To allow indexers to collect indexing rewards any time an allocation is open by presenting a valid Proof of Indexing (POI).
- To “resize” allocations (change the amount of allocated tokens), which can be useful to balance allocated stake across allocations without closing them.

**Indexing rewards**

Indexers can collect indexing rewards for their indexing work by presenting a POI. This must be done on an open allocation and will not affect its status, rewards can be collected from the same allocation an indefinite amount of times. Note that POIs must be valid for the epoch they are being presented, based on the block number reported by the Epoch Block Oracle. Since allocations can now last for a long time and in order to ensure indexers keep up with the indexing work, the Subgraph Service will require them to periodically post POIs on-chain. This will be enforced by denying rewards for POIs that are not “fresh” enough. Rewards will not be issued if the POI being presented is considered stale even if it’s valid, that is, if the interval between the last and current POIs is greater than a threshold. We propose setting this threshold to 14 days, which is half of the original 28 days lifecycle that was designed for the high gas fees environment on L1.

The following diagram depicts a scenario where the indexer was late when presenting the second POI and so the rewards corresponding to that period were forfeited:

[poi](../assets/gip-0068/poi.png)

Indexers are advised to periodically post POIs to prevent being locked out of future rewards, if needed a zero POI can be presented to reset the “freshness counter”. This could be useful if there is an infrastructure or indexing problem that leads to downtime and the last POI becomes stale, by submitting a zero POI the indexer forfeits previous rewards but ensures upcoming ones can be collected:

[poi2](../assets/gip-0068/poi2.png)

One consequence of long lived allocations is that the allocated amount will accrue rewards continuously, even if no POIs are presented and the indexer gets no rewards they would be diluting everyone else’s rewards by “burning” theirs (actually they are never minted). To prevent this from happening allocations with stale POIs can be closed by anyone (without rewards).

Finally, we note that indexing rewards are calculated following the same issuance formula on the current iteration of the protocol: each subgraph gets a portion of the rewards based on the curation signal the subgraph has, then that amount is divided between indexers proportional to their allocated stake. This means that as long as there are indexing rewards the Subgraph Service will rely on the curation mechanism. Further details can be found on this blog post: [https://thegraph.com/blog/the-graph-network-in-depth-part-2/](https://thegraph.com/blog/the-graph-network-in-depth-part-2/). There is ongoing work to introduce Indexing Fees in parallel to Indexing Rewards, and also to iterate on the issuance mechanism, but these will be addressed in future GIPs and are orthogonal to this one.

### Serving queries

A key change introduced by the Subgraph Service is that any form of payment that the indexer collects needs to be collateralized and secured by stake deposited in a Graph Horizon provision, including payments for query fees and (in the future) indexing fees resulting from direct indexer payments. Note that future indexing fees are mentioned in this section as they will function very much like query fees however they are not a part of this GIP.

**Query Fees**

Once queries start flowing the indexer starts generating query fees. The Subgraph Service uses Graph Horizon’s payment protocol, in particular the TAP Collector, to process query fee payments. This ensures interactions between Indexers and payers (gateways) are trust-minimized, which is essential to preserving the decentralized nature of the protocol in the long term. Indexers receive TAP vouchers (RAVs) from a gateway and present them on the Subgraph Service to collect payment. The collected payment is then distributed to the following parties, essentially in the same way as in the current protocol:

- Protocol tax: a small percentage is burnt by Graph Horizon’s payment protocol (1% as in the current protocol).
- Delegator cut: delegators receive a share of the query fees for providing stake to the indexer. This is set by the indexer.
- Data service cut (curators cut): This is the share of the query fees for curators and it’s set by Subgraph Service governance (The Graph council).
- Indexer cut: remainder is sent to the indexer.

Note that there is no rebate mechanism in the query fee collection process. Instead, as indexers collect fees they lock a portion of their available stake for a period of time, these locks are called “stake claims”. Stake claims are similar to allocations in the sense that they both represent stake that is temporarily locked to provide economic security. It’s worth noting that, same as allocations, stake claims do not interfere with the thawing mechanism from the staking contract, funds being withdrawn are still subject to the thawing period established by the provision. The locking period for stake claims only affect how the provision’s stake is internally accounted for by the Subgraph Service. They have the following characteristics: 

- Locked stake cannot be used for further query fee collection until released thus ensuring at all times that there is stake backing every collection. A corollary of this is that the amount of available tokens in the provision dictate how much the indexer can collect.
- Stake claims have a duration equal to the Subgraph Service’s dispute resolution window to allow enough time for disputes to be raised and settled before the indexer can remove stake from the protocol. This is designed so that if an indexer collects fees and starts thawing at the same time, the funds are guaranteed to be available for slashing for the time the claim is locked. As a reminder the Subgraph Service requires the thawing period on its provisions to be equal to the dispute resolution window.
- The amount of stake that gets locked for a given amount of fees being collected is determined by the “stake-to-fees” ratio, a parameter which can be set by governance. For example, collecting 10 GRT with a stake-to-fees ratio of 5 will result in 50 GRT being locked. The higher the ratio the higher the economic security behind the query is as locked stake risks being slashed. However the maximum stake-to-fees ratio is limited by the locked stake cost of capital and the length of the dispute period:
    - This limitation is quite aggressive. e.g. with a dispute period of 14 days and assuming a ~1% monthly cost of capital (~12% annual), the maximum stake to fees ratio is 200 (see Appendix A), and this means the fees are just enough to cover the cost of capital - so in practice, a realistic max stake-to-fees ratio should be lower, e.g. 100.
    - We propose setting an initial value based on the current total stake and delegation, and the amount of indexing rewards that are distributed, which would produce an approximate value of 188 (see Appendix B). More economic research could allow proposing a different value in future GIPs.
- Whenever query fees are collected, previously expired stake claims will be automatically released.

### Provisioned stake usage

In previous sections we mentioned both types of work performed by indexers make use of “available tokens” from the provisioned stake. For indexing rewards, the indexer uses stake to create allocations which will earn them indexing rewards when POIs are presented. For serving queries, the indexer locks stake on stake claims each time they collect payment for their queries. Future indexing fee payments would also use the same stake claims mechanism.

It’s important to consider how the provisioned stake is used for economic security. Both indexing rewards and query fees share the same pool of provisioned stake however they each have separate accounting. Essentially this means that there is some leveraging at play as provisioned stake is being used to collateralize two different types of work at the same time. This is more capital-efficient and closer to the current protocol, where the allocations used to collect rewards and post POIs are the same that are then used to calculate rebates. There is a tradeoff between this capital efficiency and economic security, as using the same stake for multiple purposes hypothetically makes it more profitable to carry out multiple simultaneous attacks, as they would all share the same cost-of-corruption. We think allowing this single shared use, and not more, strikes the right balance between capital efficiency and security. The following diagram shows how the stake in a provision can be reused for the different types of work the indexer performs:

[allocation](../assets/gip-0068/allocation.png)

It’s expected that the size of the provision will be the a priori measure of economic security to be used by the gateway’s ISA - ideally gateways will correlate this to collected and pending payments and stop sending queries to an indexer that is under-staked for the amount of query fees that the indexer is owed (at least as far as each gateway knows).

### **Slashing**

The Subgraph Service provides a way to permissionlessly create disputes for incorrect behavior in the Subgraph Service, potentially slashing indexer’s stake. Dispute resolution is delegated to a special contract, the Dispute Manager. This is for the most part the same contract the current protocol uses for dispute resolution, however there are some a differences. Also note that the existing Dispute Manager will not be upgraded, this will be a new contract to be deployed. We’ll provide a brief overview of the functionality and cover those differences.

The Dispute Manager allows creating two types of disputes: 

- Query disputes: Indexers receive queries and return responses with signed receipts called attestations. An attestation can be disputed if the consumer (or other party) thinks the query response was invalid. Indexers use the derived private key for an allocation to sign attestations for each query, which means they can be identified.
- Indexing disputes: Indexers will periodically present a Proof of Indexing (POI) to prove they were indexing a subgraph and claim indexing rewards. This POI is optimistically accepted when posted however anyone can challenge it and dispute it’s validity.

Disputes are submitted by Fisherman, a permissionless role in the protocol designed to bolster the economic security by ensuring there’s always an incentivezed party focused on maintaining data integrity. In order to create a dispute, a fisherman needs to provide a bond to back their claim. This bond will be returned to them unless the dispute is deemed invalid. A high enough bond ensures disputes cannot be spammed with hopes of doing a denial of service type of attack. The Dispute Manger defines a special role, the arbitrator, which can resolve disputes in the following ways:

- accept a dispute: the indexer gets slashed, the dispute submitter gets back the bond plus a reward.
- draw a dispute: the indexer does not get slashed, the dispute submitter gets back the bond.
- reject a dispute: the indexer does not get slashed, the dispute submitter’s bond is burnt.

Arbitrators should investigate and resolve disputes within an established dispute period. The Subgraph Service enforces a thawing period equal to this dispute period, this ensures that indexers cannot withdraw their stake from the protocol before disputes are resolved. One deviation from the current iteration of the Dispute Manager is that fishermen are now allowed to cancel disputes that have been open for more than the dispute period. This protects their bond from being indefinitely locked up due to arbitrators' inaction.

The main change introduced by the new Dispute Manager is how the slashed amount is determined. In the current implementation it’s a fixed percentage on the total indexer’s stake. This results in an arbitrary amount which is not correlated in any way to the work that was found at fault, also preventing the indexer from adding more stake until the dispute is solved (if they do, the added stake will also get slashed). In the Subgraph Service both types of work an indexer can perform require setting aside portions of stake as collateral (allocations or stake claims). This already establishes a baseline measure of economic security behind the indexers work, it would be sensible to tie it to the amount to be slashed.

It’s not easy however to find a solution that works for all cases. If we take query fees for example, we could use the stake-to-fees ratio to determine the slash amount (i.e. slash the entire stake claim), but this produces a very bad economic security, as for example slashing 0.1 GRT for a 0.01 GRT query is likely a very lousy deterrent for an attacker that is profiting from giving bad data. A multiplier could be added on top but that would require fine tuning the parameter and it’s likely a dynamic/variable multiplier would give much better and more fair results. We propose then shifting the burden of the decision to arbitrators, they should get to decide what the slash amount is when accepting a dispute with the only limit being the amount of tokens in the slashed provision. Arbitrators are well versed in the protocol’s inner workings and are able to gather and understand context to make an informed and fair decision. This increases the responsibility and work required from arbitrators but allows for maximum flexibility.

# Detailed specification

A full specification for the Subgraph Service contracts can be found here: 

[Subgraph Service technical specification](https://edgeandnode.notion.site/Subgraph-Service-technical-specification-v1-0-WIP-fae5361e50bd4d8fa27684ba703062dd?pvs=25)

It's worth noting this specification is a living document containing implementation details that can slightly change during the course of the development and auditing cycles. Once a finalized version is available it will be added to this GIP.

# Backward Compatibility

The Subgraph Service consists primarily of new contracts. While they can be deployed independently from others it requires the changes introduced by Graph Horizon to work properly. Once everything is deployed and configured the Subgraph Service can start being used: indexers can register, allocate and start collecting indexing rewards and query fees. There is no data migration required however there are some caveats which we’ll cover in the following sections.

**Allocation management and transition period**

Allocations are originally managed on the staking contract. As described by GIP-0066, once the Graph Horizon upgrade is completed a transition period will begin where existing open allocations on the staking contract will be allowed to be closed before switching over to the Subgraph Service. Refer to the Horizon GIP for details. 

**Legacy allocations migration**

Allocations in the protocol are unique, they represent a stake commitment from an indexer on a given subgraph at a given time. With the introduction of the Subgraph Service the allocation space becomes fragmented: legacy allocations will exist in the staking contract, and new allocations on the Subgraph Service. Re-using legacy allocation ids on the Subgraph Service might lead to unexpected behavior on offchain components. To prevent this from happening the Subgraph Service will allow a governance role to import a list of allocation ids which will be checked whenever a new allocation is created. 

**Service Registry and Dispute Manager deprecation**

As stated before, the Subgraph Service includes an internal registry which previously was a separate contract. This registry contains the information necessary for offchain components to find indexers in the network (most notably a URL). The Service Registry contract will be deprecated at the end of the transition period, it should act as the source of truth for this information until that time or until an indexer registers in the Subgraph Service, whichever happens first. It’s worth noting that there are ongoing efforts to create a service registry that can be used by any data service. This will be described in detail in another GIP.

The old Dispute Manager in turn can also be deprecated once it’s no longer needed. Allocations can be closed right up until the transition period ends, so it makes sense to hold the deprecation until after the statute of limitations period passes (which means 56 epochs after the transition period).

**Curation**

Note that this proposal, or GIP-0066, does not include any modifications to the curation mechanism.

## Summary of changes

Here is a summary of the changes proposed in this GIP:

|  | Current protocol | Transition period | Graph Horizon |
| --- | --- | --- | --- |
| Indexer | Register at the Service Registry contract. | New registrations at the Subgraph Service.<br><br>Source of truth: Service Registry unless indexer registers in the Subgraph Service. | Register at the Subgraph Service. |
|  | Allocations managed at the staking contract. | Legacy allocations: can only be closed or collected at the staking contract.<br><br>New allocations: managed at the SubgraphService contract. | New allocations: (same as transition period) |
|  | Allocations need to be closed every 28 epochs to claim indexing rewards. | Legacy allocations: can be closed one last time. | Allocations don’t need to be closed. POIs required to be presented on chain every 7 or 14 days to claim rewards. |
|  | Allocations can be closed permissionlessly after 28 epochs. |  | Allocations can be closed permissionlessly after 7 or 14 days. |
|  | Query fees are collected through a rebates mechanism. Query fees can always be collected. | Depends on the type of allocation. | Collecting query fees locks stake for a period of time. Collection cannot happen if the indexer is under-staked. |
| Fisherman | Disputes raised on the current Dispute Manager. | Legacy allocations: disputes raised on the current Dispute Manager.<br><br>New allocations: disputes raised on the new Dispute Manager. | Disputes raised on the new Dispute Manager. |
|  | Bond is locked until arbitrators resolve the dispute. | Depends on where the dispute was raised. | Fisherman can reclaim their bond after the dispute period has passed if it has not been resolved. |
| Arbitrators | Slashing amount is a fixed percentage of the indexer’s stake. | Depends on where the dispute was raised. | Slashing amount determined by the arbitrators. Can slash the entire provision. |
| Curators | Curation used to signal subgraphs that are worth indexing. Indexing rewards proportional to curation signal. | Curation used to signal subgraphs that are worth indexing. Indexing rewards proportional to curation signal. | No changes proposed in this GIP. Direct Indexer Payments (DIPs) and/or other initiatives address the evolution of curation. |

## Appendix

### Appendix A: Maximum stake-to-fees calculation

Given the following definitions:

- $R$, the stake to fees ratio
- $q$, the amount of query fees being collected for a period of time $T$
- $L$, the amount of stake locked in a stake claim for a period of time $T$, defined as $L=R*q$

We can then compare for a period of time $T$ the expected return on investment versus the cost of capital as:

$$
ROI = \frac{q}{L} = \frac{q}{Rq} = \frac{1}{R}
$$

$$
CoC = r
$$

$$
ROI \geq CoC \Rightarrow\frac{1}{R} \geq r \Rightarrow \frac{1}{r} \geq R
$$

If we take $T=14 days$ and assume a 1% monthly cost of capital (~12% annual), then $r=0.05$ and $R<=200$.

### Appendix B: Current protocol estimation

We can estimate the equivalent value for $R$ in the current version of the protocol by calculating the overall ROI as follows:

- $S$, the total stake in the protocol (indexer and delegator’s stake)
- $I_R$, the indexing rewards issued over a period of time $T$

$$
ROI=\frac{I_R}{S}
$$

$$
ROI  = \frac{1}{R} \Rightarrow R=\frac{S}{I_R}
$$

Solving for current values, $S= 2492134415.56$ (roughly 0.7B GRT from indexers plus 1.8B GRT from delegators) and $I_R= 13219971.0579$ (approximately 317M GRT issued yearly):

$$
R=\frac{S}{I_R}=\frac{2492134415.56}{13219971.0579}=188.51
$$