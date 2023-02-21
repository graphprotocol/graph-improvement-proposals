---
GIP: 0039

Title: Curation v1.x

Authors: Howard Heaton, Anirudh Patel

Created: 2022-09-29

Updated: 2022-09-29

Stage: Accepted

Category: Protocol Economics
---

* * * * *

GIP-0039: Curation v1.x
=======================

Disclaimer: As indicated by the version name in the title, this GIP suggests a patch, not a long-term curation mechanism. Also, the proposed change is only for subgraphs deployed on L2.

Abstract
--------

The Graph community has interest in moving many operations from Ethereum to Arbitrum One Layer 2 blockchain (L2). To aid developers in this transition, this GIP requests flattening curation bonding curves (i.e. making their slopes equal zero). This will result in a curation pool for each subgraph rather than a bonding curve, whereby curators will be rewarded exclusively by query fees. In particular, no curator will be able to realize gains at the expense of other curators' signal. This change will apply only to subgraphs deployed on L2. This proposal serves as a curation patch, setting the stage for future updates to benefit developers and indexers.

Motivation
----------

This GIP aims to improve the experience of subgraph and dapp developers (hereafter called "developers"). Developers want to ensure specific subgraphs are indexed, and they can do this by also acting as curators. That is, if a developer puts signal on a subgraph they create, then the signal guarantees indexers can obtain indexing rewards by indexing that subgraph (i.e. signal incentivizes indexers to index the subgraph). However, bonding curves make many developers find the signaling experience both expensive and unintuitive. For example, a common inquiry asks how much GRT a developer needs to signal on a subgraph to ensure their data will be served, the answer to which is often elusive with bonding curves.

A subgraph will typically be indexed only if indexers can profit from it. Today, indexing rewards far exceed collected query fees, which causes signal to strongly correlate with which subgraphs get indexed. In other words, more signal on a subgraph (roughly) means more indexers on that subgraph. Thus, signal presently provides a clear way for developers to get the data they care about indexed. Our patch provides a short-term solution for enabling developers to get indexed in a straightforward and principal-protected way.

Proposal Specification
----------------------
The mechanism works as follows. Suppose a curator makes deposit d>0 into the curation pool (*a.k.a.* reserves) for a subgraph while there is currently signal s≥0, resulting ⋆ in signal 0.99⋅d+s. This new curator is minted shares equaling their proportion 0.99⋅d/(0.99⋅d+s) of the subgraph's signal. That is, if there are currently S shares, then the new curator will be minted S⋅0.99⋅d/(0.99⋅d+s) new shares. So, each curator is entitled to the same proportion of reserves as their proportion of shares. This implies curators can also burn their shares to retrieve GRT from curation reserves in proportion to the shares burned. All incoming query fees to curation are added to the reserves held by the subgraph's smart contract. For example, if there is 900 GRT of signal and a new curator adds ⋆⋆ 101.01 GRT, then the new curator will be entitled to about 10% of the incoming query fees (until a next time curator mints/burns shares).

⋆ The reason 0.99⋅d is listed rather than simply d is curators pay an upfront 1% "curation tax" to mint shares. See this [prior GIP](https://forum.thegraph.com/t/proposal-to-reduce-curation-tax/2212) for more discussion.

⋆⋆ The number in this example is 101.01≈101.¯¯¯¯¯¯01=10099⋅100, which is the exact amount of GRT to obtain 10% of curation shares.

Backwards Compatibility
-----------------------

This upgrade will occur exclusively to subgraphs deployed on L2. For this reason, no backward compatibility issues are expected. Independently from this proposal, we anticipate some "run on the bank" phenomena when subgraphs migrate from Ethereum to L2.

Risks and Security Considerations
---------------------------------

**Note**: Most of the considerations below concern long-term effects of implementing this GIP. These are included for completeness and as a record for considerations in the upcoming design of Curation v2.

1.  Limited incentive to be early. Since curators are rewarded solely from query fees and are not incentivized to curate prior to the arrival of query fees, new curators can mint shares immediately prior to the arrival of query fees, thereby "stealing" the majority of query fees from early curators. These late curators can immediately burn their shares to retrieve their signal and query fee profits, and then repeat this behavior on another subgraph.
2. Increasingly expensive to get indexed. If implemented long-term, we expect this proposal to result in a "race-to-the-top". Consider this example: Subgraph A has 1 GRT signal and is currently indexed by all indexers. Next subgraph B is created and receives 2 GRT of signal. As a result, A might no longer is indexed by all indexers as its relative amount of signal decreased from 100% to 33%, decreasing its quality of service (QoS). If the reduction in QoS is unacceptable, developers who rely on A must increase the signal on A, *e.g.* to be >2 GRT. As a result of this action, developers who rely on B having a certain QoS may need to increase the signal on B, and so on. This process continually increases the cost to get indexed via curation signal. Although this is a toy example, it illustrates a possible phenomenon that yields increasing barriers to entry.
3.  Increasing centralization. If subgraphs have different amounts of query fees collected, then the relative amount of signal on the subgraph with the most query fees will increase over time, assuming at least one curator on that subgraph never burns shares. This implies, as time passes, the vast majority of indexing rewards will be distributed to the one subgraph with the most query fees.
4.  Hinder adoption of queries. The proposed upgrade protects the principal of invested signal, which enables curators to, without penalty, put large signal on subgraphs without queries (*e.g.* "broken" subgraphs). Also, most indexers presently chase indexing rewards *rather* than seek query fees (as indexing rewards are currently more profitable). These two facts enable this curation upgrade to draw indexers away from indexing subgraphs that would entail many queries, *i.e.* *hinder* the generation of query fees in The Graph.
5.  Limited value if query fees outpace indexing rewards. Most indexers follow rewards (not necessarily signal). If query fee profits greatly exceed indexing reward profits, then indexers may chase query fees rather than indexing rewards. If this happens, it may become increasingly difficult for developers to get subgraphs indexed via curation signal. In that case, this proposal may have limited value to both developers and indexers.

Rationale
---------

The curation patch must satisfy the following constraints:

-   Simple to describe

-   Simple to implement

-   No ability for curators to take each other's investment

These constraints arise from the need to both efficiently to deploy on L2 and improve developer experiences. Regarding the first constraint, this proposal is straighforward as each curator obtains curation shares in proportion to their GRT investment and is rewarded in proportion to their curation shares (i.e. no "rug pulling"). Implementation will require a single parameter change (i.e. possibly a "no code" update). As mentioned, this change also removes the potential for curators to exploit one another monetarily by preventing front-running and profit scalping attacks. Curators can still dilute one another, resulting in less rewards than expected; however, they can recover their investment (minus gas fees and curation tax). Lastly, despite the notable concerns listed above, we believe our proposed design is more compatible with potential long term solutions than bonding curves, making this an apt upgrade.

Considered Alternatives
-----------------------

Late Fees. It is possible to transfer a portion of newly added signal (i.e. GRT) to the holdings of existing curation shareholders. This one-time late fee (e.g. 2% of investment) can be known prior to providing signal, preventing the unknown downside exposure curators often experience today. This alternative gives explicit downside protection and rewards early curators.

Delayed Rewards. Another option considered is to delay distribution of curation rewards. Here curators provide signal, but do not start accumulating rewards for their signal until after a fixed amount of time passes. This delay prevents early curators from having their rewards "stolen" from late curators that hop in after query fees become substantial for a subgraph. This incentivizes early curation, but may require multiple transactions, complicating users' experience.

Continuous Issuance. The third alternative is to continually issue new curation shares. Here curators place deposits and are issued shares in proportion to their deposited GRT (relative to other deposits), and rewards are distributed in proportion to shares held. This scheme is i) cognitively difficult to reason about, ii) potentially difficult to implement, and iii) potentially expensive with respect to gas costs (if tractable). Ongoing investigations are stress testing variations of this idea for future consideration in Curation v2.

[Principal-Protected Bonding Curves](https://forum.thegraph.com/t/gip-0025-principal-protected-bonding-curves/3162). A prior proposal suggests keeping bonding curves, but also adding bookkeeping so that each curator's investment is principal-protected. This is yet another scheme for incentivizing early curation. However, the skewed in perpetuity rewards from query fees are undesirable (e.g. a curator that is 1 minute earlier than another curator may pay the same to signal yet obtain notably larger query fee rewards forever).

Potential Future Work
---------------------

Upcoming curation upgrades will strive to better serve developers' ability to clearly and accurately communicate value to indexers so that valuable subgraphs can be efficiently indexed.

Copyright Waiver
----------------

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).