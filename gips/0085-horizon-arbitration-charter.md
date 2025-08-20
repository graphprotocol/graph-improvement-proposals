---
GIP: "0085"
Title: Graph Horizon: Arbitration Charter
Authors: Tomás Migone <tomas@edgeandnode.com>
Created: 2025-08-20
Updated: 2025-08-20
Stage: Candidate
Discussions-To:
Category: Protocol Charters
Depends-On: GIP-0066, GIP-0068
---

# Abstract

The Graph has a protocol role called an Arbitrator who is assigned through decentralized governance. The Arbitrator is an Ethereum Account that has the ability to decide the outcome of disputes in the protocol. The purpose of the Arbitration Charter is to establish norms that constrain the Arbitrator's actions beyond what is expressible or currently expressed in smart contract code. We propose that any Arbitrator that does not comply with the norms laid out in this charter be replaced via decentralized governance.

This charter applies specifically to the **Subgraph Service**, a data service in The Graph Network that supports indexing subgraphs and serving queries to consumers. Other data services introduced as part of the Horizon upgrade may establish their own arbitration charters.

# Motivation

Given the Arbitrator’s role nature there are certain parts of its behavior that are not specified in smart contract code running on-chain. Having a protocol charter for this actor's behavior creates clarity for the ecosystem in how the role of the Arbitrator will be executed and establishes a standard for measuring the effectiveness of an Arbitrator, which can be referenced in protocol governance discussions around the appointment of an Arbitrator.

The substance of the Arbitration charter is intended to ensure that the Arbitrator is fulfilling their role of supporting a healthy and functioning network where Indexers perform their work correctly, while minimizing the risk to honest Indexers of being economically penalized while interacting with the protocol in good faith.

---

*This section contains the body of the arbitration charter. Later sections elaborate on the context and motivation for the design.*

# Arbitration Charter

## 1. Role of the Arbitrator

The role of the Arbitrator is to decide the outcome of indexing and query disputes within the Subgraph Service. Their goal is to maximize the utility and reliability of the service.

They fulfill this purpose by examining **Proofs of Indexing (POIs)** or query **Attestations** associated with a dispute and checking if they correspond to the results the Arbitrator produced when reproducing the work themselves. The Arbitrator might also choose to rely on other Indexers to reproduce results to aid in an investigation. Finally they decide the outcome according to the norms laid out in this charter.

Disputes are created by a **Fisherman** against Indexers. A successful dispute resolved in the Fisherman's favor results in a financial penalty for the Indexer and a reward to the Fisherman.

## 2. Recourse for an Arbitrator not fulfilling their duties

The only recourse for an Arbitrator not fulfilling their duties is to be replaced via decentralized governance. Note that only the **Subgraph Service** governing body has the authority to appoint or replace an Arbitrator. 

Arbitrators have no legal or fiduciary responsibility to any stakeholder in the network and cannot be held responsible for any consequence that arises from incorrect data being served in the network or any economic penalties incurred by stakeholders involved with the dispute process. 

## 3. Types of disputes and associated outcomes

There are two categories of work that can be disputed in the network: indexing and querying with three dispute types that facilitate their dispute:

1. **Single Query Attestation Dispute:** A Fisherman puts a deposit at stake (bond) to challenge a query Attestation provided by an Indexer. Possible outcomes:
    - **Accept dispute:** A portion of the Indexer's stake is slashed and the Fisherman is awarded a percentage of the slashed amount plus their bond.
    - **Reject dispute:** There is no penalty to the Indexer and the Fisherman's deposit is forfeit.
    - **Draw dispute:** The Indexer is not penalized, Fisherman’s bond is returned.
2. **Conflicting Query Attestation Dispute:** A Fisherman puts a deposit at stake (bond) and submits two Attestations to the same query that attest to contradictory responses. Possible outcomes:
    - **Accept both disputes:** If both attestations are deemed invalid both Indexers are slashed. The Fisherman gets a percentage of both slashed Indexer’s stake plus their bond.
    - **Accept one dispute, draw the other one:** Only one attestation is invalid. The Fisherman gets a percentage of the slashed Indexer’s stake plus their bond.
    - **Draw dispute:** None of the Indexers are found at fault. The Fisherman gets their bond back and no Indexer is penalized.
    
    By nature conflicting query disputes cannot be rejected as one of them is necessarily incorrect. The draw scenario allows to resolve disputes in a fair manner in the event the discrepancy cannot be attributed to an Indexer but rather a software bug or similar. 
    
3. **Proof of Indexing Dispute:** A Fisherman puts a deposit at stake (bond) to challenge a POI provided by an Indexer. Possible outcomes:
    - **Accept dispute:** A portion of the Indexer's stake is slashed and the Fisherman is awarded a percentage of the slashed amount plus their bond.
    - **Reject dispute:** There is no penalty to the Indexer and the Fisherman's deposit is forfeit.
    - **Draw dispute:** The Indexer is not penalized, Fisherman’s bond is returned.

## 4. Slashing Amount Determination

When an Arbitrator accepts a dispute they need to specify the amount of tokens the associated Indexer will get slashed by. The protocol contracts enforce minimum and maximum values:

- Minimum slashing amount: `0 GRT`
- Maximum slashing amount: `maxSlashingCut * stakeSnapshot`
    - Where:
        - `maxSlashingCut` is a parameter set by governance in the `DisputeManager` contract.
        - `stakeSnapshot` is the sum of the Indexer’s stake provisioned to the **Subgraph Service** plus any delegated stake they might have. This value is calculated on-chain during the dispute creation and emitted in an event.
- Also note that the actual slashed amount might be smaller than the requested slash amount due to the staking mechanisms at play. This behavior is encoded in the `HorizonStaking` contract, any differences resulting of this should not be compensated elsewhere by the Arbitrator.

To determine an amount Arbitrators must consider:

- Severity of the fault
    - Was the fault due to clear negligence, outdated software, or malicious behavior?
    - Did the Indexer ignore known bugs or fail to act after being warned?
    - Was the POI or Attestation knowingly forged?
- Impact on the network
    - Did the invalid data affect real consumers?
    - Was there financial harm or reputational risk?
    - Was the subgraph high signal (many queries, large rewards)?
- Repeat offenses
    - Did the Indexer repeat the same behavior across multiple epochs or disputes?
    - Is there a pattern of recurring faults?
- Indexer response and cooperation
    - Did the Indexer respond to the forum thread and engage constructively?
    - Did they share logs, acknowledge issues, or show willingness to fix their setup?
- Precedents and consistency
    - Have similar disputes been resolved recently? What slash amounts were used?
    - Are there existing dispute precedents posted by this Arbitrator or others?
- Stake snapshot composition
    - How much of the stakeSnapshot is provisioned vs. delegated?
    - Would the selected slash amount begin to affect the delegation pool?

The recommendation is to slash 2.5% of the Indexer’s stake snapshot. However note that the actual amount to be slashed is at the complete discretion of the Arbitrator. 

Additional recommendation regarding delegation pool slashing: due to a limitation in the current `DisputeManager` implementation, it’s not recommended to slash an Indexer down to `0 GRT` as that would effectively block any further disputes from being created against the Indexer. This is only relevant if delegation slashing is globally enabled at the Graph Horizon level, as the block would only prevent delegation from being slashed. The recommendation is to slash the Indexer to a low value such as `10 GRT`.

## 5. Request for information about disputes

The Arbitrator must create a forum thread (https://forum.thegraph.com/c/governance-gips/17) to give an Indexer the chance to provide logs and interact with core developers to help assess whether a determinism bug is at fault.

Even if an Indexer does not provide evidence on their behalf in the forum, the Arbitrator is encouraged to exercise discretion in resolving disputes as a Draw if there is a reasonable probability that it was caused by a known determinism bug in the software.

The Arbitrator should create this post in the forums promptly after a dispute is submitted, and leave at least seven (7) days for the disputed Indexer(s) to engage before arriving at a decision. 

## 6. Settling disputes in a timely manner

The Arbitrator should seek to settle all disputes within one week and a day of a dispute being submitted. This timeline is to take into account the seven days allotted for Indexers to respond to information requests. 

The Arbitrator should take whatever measures are required, such as indexing the subgraphs with the most Curator signal ahead of time to increase the likelihood of being able to settle an indexing or query dispute expediently.

In the event of subsequent back to back cases of POI disputes, the Arbitrator need not wait the full seven (7) days to take further action. It is recommended that the Arbitrator either place a hold on the portion of the Indexer operation that is at risk of serving incorrect data, and / or slash the Indexer operation again in cases where there are clear signs of malicious intent. 

## 7. Publishing information about decision

The Arbitrator must post a decision in the dispute's forum thread that briefly describes what outcome was chosen for a dispute, why it was chosen, and what factors were considered. This should be posted at least 48 hours before the dispute is settled on-chain, assuming the offending Indexer doesn't have any un-bonding stake that will become available to withdraw during that time period. If any new mitigating circumstances are presented in that window, the Arbitrator may revise their decision.

In addition to the published decision, the Arbitrator must post a post-mortem in the dispute's forum thread for disputes that are settled as a Draw due to some malfunction or shortcoming of the software. This should include the specific root causes of the fault and what actions are recommended to prevent such software-related faults in the future. The Arbitrator may coordinate with core developers of the protocol to produce this post-mortem.

## 8. Indexing and query faults due to software malfunctions

Incorrect Proofs of Indexing or query Attestations may be produced due to software malfunctions in the Graph Node software or the Indexer's blockchain client. We will refer to these malfunctions collectively as *determinism* *bugs*.

The Arbitrator is encouraged to resolve disputes as a Draw if they believe an incorrect POI or Attestation to be the result of a determinism bug.

## 9. Double Jeopardy

Indexers should only be slashed once for a given indexing or query fault. Any duplicate disputes for the same disputable element must be settled as a Draw. In the event that multiple Fishermen raise simultaneous disputes for the same disputable elements, only the first Fisherman to log a dispute will be considered. The remaining disputes will be settled as a Draw.

A disputable element is:

- For query disputes: a query attestation.
- For indexing disputes: a POI submission (the POI plus the block number when it was submitted onchain by the indexer). Note that a POI can be challenged multiple times if it’s repeatedly being re-submitted.

In the event a dispute is raised where the block number specified by the Fisherman does not include the associated POI submission the recommendation is to resolve the dispute as a Draw.

## 10. Statute of Limitations

Indexers should not be slashed for faults that occurred more than two thawing periods prior unless two thawing periods is less than seven (7) days. For example, if the thawing period is set to 28 epochs, then an Indexer should not be slashed for any fault that occurred more than 56 epochs prior. Note that for the **Subgraph Service** the thawing period is defined as equal to the `disputePeriod` global variable in the `DisputeManager` contract. The Arbitrator must decide any such dispute that is outside this statute of limitations as a Draw.

If a POI or query attestation discrepancy is raised for dispute, leveraging historical Indexer activity beyond the statute of limitation to assist in making a determination by the arbitration council is still acceptable.

## 11. Data Availability

In order for an Arbitrator to settle a dispute correctly, they must have access to the following data, which is not stored on-chain:

- The subgraph manifest, schema and mappings.
- The query body (for query disputes).

If the Arbitrator cannot access any of this data via the IPFS network, then they must resolve the dispute as a Draw.

## 12. Indexing data integrity

The Graph Node software indexes data from blockchain inputs. If the input data is inaccurate, the resulting subgraph and any derived POI will also be incorrect. Depending on the subgraph code, indexing bad data may even cause subgraph failures which could be misinterpreted as determinism bugs. Upholding the quality of the data is essential for the network's overall health and reliability. Indexers are responsible for ensuring the integrity and chain of custody of the data they serve, which includes sourcing blockchain data from reputable sources.

The Arbitrator is encouraged to resolve disputes as a Draw if they believe an incorrect POI to be the result of a blockchain client malfunction (RPC/firehose).

However, upon a discrepancy being noticed the Indexer should take reasonable measures to rectify the issue or submit a zero POI for any subsequent allocations. Note that the Indexer must be notified by the Arbitrator by posting in the forum; the Indexer will then be given a seven (7) day period to work with the Arbitrator and the community on addressing the problem after which any new disputes against a non zero POI can be resolved at the discretion of the Arbitrator.

## 13. Maximum allowable slashing for query disputes

Indexers should only be slashed for a maximum of once per epoch per allocation.

For example, if an Indexer was allocated for 28 epochs, then they may be slashed 28 separate times for query disputes associated with that allocation.

Any dispute that would result in an Indexer being slashed in excess of this number of instances should be settled as a Draw at the discretion of the Arbitrator. If there are multiple POI issues uncovered during an investigation it is recommended that the specific portion of the indexer operation in question be halted until a resolution is reached.

Multiple slashes should only take place in exceedingly exceptional circumstances where malicious intent or serious risk to the network is determined beyond a reasonable doubt.

## 14a. Valid Proofs of Indexing for a given epoch

When closing an allocation during an epoch, an Indexer must submit a valid POI as of the first block of that epoch. In a dispute, if a POI is invalid for the epoch in which the allocation was closed, but valid as of the first block of the preceding epoch, then the Arbitrator should settle the dispute as a Draw.

If the Indexer is unable to produce a valid POI for whatever reason, then they must close the allocation with a so-called "zero POI," which is a POI comprising all zeros. For example, if a subgraph has a bug that prevents indexing up until the current epoch, then a zero POI should be submitted and indexing rewards must not be collected for that subgraph. An exception to this rule is if the allocation being closed was opened before the subgraph bug occurred. In this case, the Indexer may submit the last valid POI they produced for the subgraph and collect indexing rewards for the allocation.

## 14b. Matching POIs

When posting a POI to the **Subgraph Service** Indexers are required to post both the calculated POI (raw POI hashed with their address) plus the public POI (raw POI hashed with zero address). It’s expected that both POI “match”, that is that they were both generated from the same base raw POI. This can be easily verified by any other entity indexing the related subgraph. In case there is a POI mismatch, an indexing dispute can be created against the Indexer, the recommendation is to accept the dispute and slash the Indexer. 

## 15. Subgraph API and Indexer software versioning

It's possible that a POI or Attestation provided by an Indexer is invalid due to the Indexer running an outdated version of the protocol software. As described in GIP-0008, protocol upgrades of the Indexer software and Subgraph API will specify a grace period during which the previous official version of the Indexer software may still be run. For disputes involving a POI or Attestation that is only correct with respect to the previous official version of the Indexer software, the Arbitrator must settle any such dispute as a Draw.

If no grace period is specified for new software versions, the default of a 30 day window applies.

## 16. Fisherman Dispute Cancellation

If a dispute remains unresolved after a full dispute period passes since its creation the Fisherman can choose to cancel the dispute. A cancelled dispute will return the original bond to the Fisherman and will no longer subject the Indexer to slashing.

## 17. Fisherman Malfeasance

The Arbitrator may **reject** a dispute and **slash the bond** of a Fisherman if they are found to be submitting malicious or spam disputes to waste arbitration resources(e.g., DoS attacks, dispute spamming, etc). Note that the dispute can be rejected even if the underlying disputable element was found to be correctly disputed. 

## 18. Legacy disputes

During the transition to Graph Horizon certain situations might require the use of a new, temporary type of dispute: legacy disputes. 

An Indexer that gets one of their legacy allocations slashed during the transition period might have also moved a significant part of their stake to the **Subgraph Service** provision. This would create a discrepancy in the amount that is slashed. To compensate for this difference the Arbitrator can create at their discretion a legacy dispute.

A legacy dispute sole purpose is to settle outstanding slashing amounts with an Indexer that has previously been “legacy slashed” during or shortly after the Graph horizon transition period. This type of dispute can only be created by the Arbitrator, it does not require a bond and it’s automatically accepted when created. Additionally, note that this type of disputes allow the arbitrator to directly set the slash and rewards amounts, bypassing the usual mechanisms that impose restrictions on those. This is done to give arbitrators maximum flexibility to ensure outstanding slashing amounts are settled fairly. Legacy disputes should be removed after the transition period.

*This concludes the body of the Arbitration charter.*

---
