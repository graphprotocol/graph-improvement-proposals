---
GIP: "0009"
Title: Arbitration Charter
Authors: Brandon Ramirez <brandon@edgeandnode.com>, Adam Fuller <adam@edgeandnode.com>
Created: 2021-04-09
Updated: 2022-11-01
Stage: Proposal
Discussions-To: https://forum.thegraph.com/t/an-arbitration-charter-to-clarify-arbitrator-behavior/
Category: Protocol Charters
Depends-On: GIP-0005, GIP-0007, GIP-0008

---

# Abstract

The Graph has a protocol role called an Arbitrator who is assigned through decentralized governance. The Arbitrator is an Ethereum Account that has the ability to decide the outcome of disputes in the protocol. The purpose of the Arbitration Charter are to establish norms that constrain the Arbitrator's actions beyond what is expressible or currently expressed in smart contract code. We propose that any Arbitrator that does not comply with the norms laid out in this charter be replaced via decentralized governance.

---
*This section contains the body of the arbitration charter. Later sections elaborate on the context and motivation for the design.*

# Arbitration Charter

## 1. Role of the Arbitrator.

The role of the Arbitrator is to decide the outcome of indexing and query disputes. Their goal is to maximize the utility and reliability of The Graph Network.

They fulfill this purpose by looking at the **Proofs of Indexing (PoIs)** or query **Attestations** associated with a dispute and checking if they correspond to the results that the Arbitrator produced when reproducing the work themselves. They decide the outcome of the dispute according to the norms laid out in this charter and based on whether the PoI or Attestation was produced correctly.

Disputes are created by a **Fisherman** against one or more Indexers. A successful dispute resolved in the Fisherman's favor results in a financial penalty for the Indexer, along with an award to the Fisherman, thus incentivizing the integrity of the indexing and query work performed by Indexers in the network.

## 2. Recourse for an Arbitrator not fulfilling their duties
The only recourse for an Arbitrator not fulfilling their duties is to be replaced via decentralized governance. They have no legal or fiduciary responsibility to any stakeholder in the network and cannot be held responsible for any consequence that arises from incorrect data being served in the network or any economic penalties incurred by stakeholders involved with the dispute process.

## 3. Types of disputes and associated outcomes
The two categories of work that can be disputed in the network are indexing and querying. There are three dispute types that facilitate this.
1. **Single Query Attestation Dispute.** A Fisherman puts a deposit at stake to challenge a query Attestation provided by an Indexer. Possible outcomes:
 - *Indexer Wins* - There is no penalty to the Indexer and the Fisherman's deposit is forfeit.
 - *Fisherman Wins* - A percentage of the Indexer's stake is slashed and the Fisherman is awarded a portion of the slashed stake.
 - *Draw* - Neither the Indexer nor the Fisherman is penalized.
2. **Conflicting Query Attestation Dispute.** A Fisherman submits two Attestations to the same query that attest to contradictory responses. Possible outcomes:
  - *First Indexer Wins*. A percentage of the second Indexer's stake is slashed and the Fisherman receives a portion of the slashed stake. There is no penalty to the first Indexer.
  - *Second Indexer Wins*. A percentage of the first Indexer's stake is slashed and the Fisherman receives a portion of the slashed stake. There is no penalty to the second Indexer.
  - *Draw*. Neither the first Indexer, nor the second Indexer nor the Fisherman are penalized.
3. **Proof of Indexing Dispute**. A Fisherman puts a deposit at stake to challenge a PoI provided by an Indexer. Possible outcomes:
 - *Indexer Wins*. There is no penalty to the Indexer and the Fisherman's deposit is forfeit.
 - *Fisherman Wins*. A percentage of the Indexer's stake is slashed and the Fisherman is awarded a portion of the slashed stake.
 - *Draw*. Neither the Indexer nor the Fisherman is penalized.

## 4. Indexing and query faults due to software malfunctions
Incorrect Proofs of Indexing or query Attestations may be produced due to software malfunctions in the Graph Node software or the Indexer's blockchain client. We will refer to these malfunctions collectively as *determinism bugs*.

The Arbitrator is encouraged to resolve disputes as a *Draw* if they believe an incorrect PoI or Attestation to be the result of a determinism bug.

The Arbitrator must create a forum thread [here](https://forum.thegraph.com/c/governance-gips/arbitration/) to give an Indexer the chance to provide logs and interact with core developers to help assess whether a determinism bug is at fault.

Even if an Indexer does not provide evidence on their behalf in the forum, the Arbitrator is encouraged to exercise discretion in resolving disputes as a *Draw* if there is a reasonable probability that it was caused by a known determinism bug in the software.

The Arbitrator should create this post in the forums promptly after a dispute is submitted, and leave at least seven days for the disputed Indexer(s) to engage before arriving at a decision.

## 5. Double Jeopardy
Indexers should only be slashed once for a given indexing or query fault. Any duplicate disputes for the same PoI or query Attestation must be settled as a *Draw*.

## 6. Statute of Limitations
Indexers should not be slashed for faults that occurred more than two thawing periods prior.

For example, if the thawing period is set to 28 epochs, then an Indexer should not be slashed for any fault that occurred more than 56 epochs prior.

The Arbitrator must decide any such dispute that is outside this statute of limitations as a *Draw*.

## 7. Data Availability
In order for an Arbitrator to settle a dispute correctly, they must have access to the following data, which is not stored on-chain:
- The subgraph manifest, schema and mappings.
- The query body (for query disputes).

If the Arbitrator cannot access any of this data via the IPFS network, then they must resolve the dispute as a *Draw*.

## 8. Maximum allowable slashing for query disputes
Indexers should only be slashed for a maximum of once per epoch per allocation.

For example, if an Indexer was allocated for 28 epochs, then they may be slashed 28 separate times for query disputes associated with that allocation.

Any dispute that would result in an Indexer being slashed in excess of this number of instances must be settled as a *Draw*.

## 9. Valid Proofs of Indexing for a given epoch.
When closing an allocation during an epoch, an Indexer must submit a valid PoI as of the first block of that epoch. 

Prior to the introduction of the Data Edge & Epoch Block Oracle (GIP-0038), epoch blocks are defined by the EpochManager contract. After the introduction of the Data Edge & Epoch Block Oracle, the Epoch Block subgraph becomes the source of truth for Epoch Blocks across networks.

In a dispute, if the discrepancy is found to be the result of a bug in the Data Edge, or Epoch Block oracle, then the Arbitrator should settle the dispute as a *Draw*.

In a dispute, if a PoI is invalid for the epoch in which the allocation was closed, but valid as of the first block of the preceding epoch, then the Arbitrator should settle the dispute as a *Draw*.

If a subgraph encounters an error that prevents it from being synced up to a the first block of the epoch, then an Indexer should submit the last valid PoI for the block before the error occured. Indexers may continue to collect indexing rewards on a broken subgraph until the market indicates it is no longer useful to index that subgraph and removes all signal from the subgraph.

## 10. Subgraph API and Indexer software versioning.
It's possible that a PoI or Attestation provided by an Indexer is invalid due to
the Indexer running an outdated version of the protocol software. As described in
GIP-0008, protocol upgrades of the Indexer software and Subgraph API will
specify a grace period during which the previous official version of the Indexer
software may still be run. For disputes involving a PoI or Attestation that is only
correct with respect to the previous official version of the Indexer software, the
Arbitrator must settle any such dispute as a *Draw*.

## 11. Settling disputes in a timely manner.
The Arbitrator should seek to settle all disputes within one week of a dispute
being submitted. The Arbitrator should take what ever measures are required, such as
indexing the subgraphs with the most Curator signal ahead of time to increase the
likelihood of being able to settle an indexing or query dispute expediently.

# 12. Publishing information about decision.
The Arbitrator must post a decision in the dispute's forum thread that briefly describes what outcome was chosen for a dispute, why it was chosen, and what factors were considered. This should be posted at least 48 hours before the dispute is settled on-chain, assuming the offending Indexer doesn't have any unbonding stake that will become available to withdraw during that time period. If any new mitigating circumstances are presented in that window, the Arbitrator may revise their decision.

In addition to the published decision, the Arbitrator must post a post-mortem in the dispute's forum thread for disputes that are settled as a "Draw" due to some malfunction or shortcoming of the software. This should include the specific root causes of the fault and what actions are recommended to prevent such software-related faults in the future. The Arbitrator may coordinate with core developers of the protocol to produce this post-mortem.

*This concludes the body of the Arbitration charter.*

---
# Motivation
The Arbitrator is a protocol role that is assigned via decentralized governance, and as such there are certain parts of its behavior that are not specified in smart contract code running on-chain. Having a protocol charter for this actor's behavior creates clarity for the ecosystem in how the role of the Arbitrator will be executed and establishes a standard for measuring the effectiveness of an Arbitrator, which can be referenced in protocol governance discussions around the appointment of an Arbitrator.

The substance of the Arbitration charter is intended to ensure that the Arbitrator is fulfilling their role of supporting a healthy and functioning network where Indexers perform their work correctly, while minimizing the risk to honest Indexers of being economically penalized while interacting with the protocol in good faith.

The next section elaborates on the specific rationale behind select parts of the charter and how they help accomplish said goals.

# Rationale and Alternatives

## Indexing and query faults due to software malfunctions
Allowing the Arbitrator to exercise discretion in the event of determinism bugs removes possible disincentives and barriers for participation for Indexers. The alternative would require Indexers to become experts on the Graph Node software, which is written in Rust and under active development, or else assume the risk of having their stake slashed for software malfunctions that are outside their direct control.

## Double Jeopardy
This is a form of replay protection which prevents an Indexer from being slashed an indefinite number of times for a single fault. Allowing for such an outcome would be a massive disincentive to Indexers participating in the protocol.

A drawback of the current approach is that given the current Attestation structure, an Indexer actually *could* give the same incorrect response to multiple Consumers and those Attestations would be indistinguishable from one another and could result in at most one instance of the Indexer being slashed.

An alternate approach would be to alter the Attestation structure such that the replay protection only applied to incorrect query responses provided to a specific Consumer.

Furthermore, it might be preferable for the protocol smart contracts to store a record of past disputes in order to enforce the double jeopardy rule, rather than leaving it up to the Arbitrator, which might be prone to error.

Both of the above alternative designs would be far heavier to implement, however, as they require changes to the protocol software and additional on-chain bookkeeping. These may be explored in future GIPs.


## Statute of Limitations
There are several reasons for placing a statute of limitations on the faults for which an Indexer can be slashed:
1. The alternative, allowing Indexers to be slashed for faults that occurred an indefinite amount of time in the past, would punish Indexers that participate in the protocol for a long period of time and thus have a far greater number of instances where they might be slashed due to software malfunctions or operational errors.
2. Without this clause, Indexers might periodically exit and re-enter the protocol under new identities, to manage their risk of slashing due to possible accumulated faults. This lack of continuity between Indexer identities would create noise in the protocol and would possibly obstruct useful Indexer reputation systems being built around The Graph.
3. A malicious Indexer attacking the protocol can unstake their GRT immediately after an attack and thus would not be slashable after a thawing period. It would be unfair to make the stake of honest Indexers slashable for a period substantially longer than that of a malicious Indexer.

An alternate design that addresses the above issues would be to enforce the statute of limitations in the protocol smart contracts. A drawback of this approach, as before, is that it requires software changes and so is a heavier change. This approach may be explored in a future GIP.

## Data Availability
An Arbitrator can only perform their job if they are able to reproduce the work of an Indexer. This requires access to data such as subgraph manifests or the body of a query, both of which are too large to store on-chain. A Fisherman should have a built-in incentive to make sure this data is available so they can be awarded for a successful dispute.

The proposed charter requires settling disputes where data is unavailable as a *Draw*. An alternative design would be to punish the Fisherman for submitting a dispute without making the data available. A drawback of this approach is that discovering files through the IPFS distributed hash table (DHT) is not 100% reliable and could introduce undue risk to the Fisherman. This might be mitigated by requiring the Fisherman to pin all necessary files to a specific IPFS node, but this feels overly centralized and brittle.

Another alternative could be to require the necessary files to be pinned to a storage chain such as Arweave or Filecoin, and only allow a dispute to created if all necessary files are available. This may be explored in a future GIP.

## Maximum allowable slashing for query disputes
By capping the amount that an Indexer can be slashed for queries in a given time period, the charter addresses an important asymmetry between indexing and querying in the protocol: specifically, the fact that Indexers only submit one PoI per allocation and thus can be slashed a maximum of once per allocation for indexing faults, but Indexers may respond to an unbounded number of queries during an allocation and thus might expose themselves to a similarly unbounded level of risk due to slashing, capped only by their total stake, from software malfunctions or other operational errors. Without the cap specified in this clause, an Indexer *could theoretically have all their stake slashed in a single allocation.*

An alternative design would be to implement this cap in the protocol smart contracts. This is a heavier change as it requires an upgrade to the protocol smart contracts. This may be explored in a future GIP.

## Valid Proofs of Indexing for a given epoch.

Currently, the protocol smart contracts do not let you specify as of which epoch you would like to close an allocation; rather, this is inferred from the current epoch as of when the transaction gets mined that closes the allocation. This leaves open the possibility that an Indexer may submit a PoI in a transaction that is intended to close an allocation in one epoch, but the transaction might not actually get mined until the following epoch. Given the unpredictability of gas costs and interacting with the blockchain, it would be undesirable to punish Indexers for such a situation.

An alternative design would be to allow Indexers to specify an epoch number when closing an allocation. This is a heavier change as it requires an upgrade to the protocol smart contracts. This may be explored in a future GIP.

## Experimental features

Subgraph API Versioning and Feature Support (GIP-0008) describes the introduction of new subgraph features, including "experimental" features. These may be known to require further development to achieve determinism, in which case they will not be eligible for rewards and therefore disputes. But in some cases, features may be expected to be deterministic, but might require observation in a production environment in order to achieve full confidence. An example might be the introduction of a new Ethereum network. In these cases, features might be marked as eligible for rewards, but in the event of a dispute, indexers who cooperate with the Arbitrator can expect a higher likelihood of a draw, given the contribution towards a better understanding the behaviour of the experimental feature.

# Dependencies
This GIP depends on two other GIPs:
- GIP-0005 (TBD) - Makes timeouts in subgraph fully deterministic using WASM gas metering. This is necessary for the Arbitrator to reproduce the work of Indexers in a way that is completely deterministic.
- GIP-0008 - Specifies the process for making upgrades to the Subgraph API and Indexer software. This is necessary for defining the canonical version of the protocol software that the Arbitrator should run in reproducing the work of Indexers.
- GIP XXXX (TBD) - Allows the protocol to set separate slashing percentages for indexing and query faults. This is necessary because this charter proposes allowing Indexers to be slashed once per epoch for query faults, which with current network parameters of 2.5% slashing and 28 epochs max allocation duration, equates to up to 70% possible slashing from query faults for a single allocation. This is disproportionately high compared to the 2.5% maximum slashing per allocation that currently exists for indexing faults.

## Risks and Security Considerations
As this GIP is a protocol charter, it does not introduce any new technical risks or security considerations that do not already exist in the protocol today.

A possible non-technical risk is that this charter provides a false sense of security to Indexers participating in the protocol that it is impossible to be slashed while interacting with the protocol in good faith, but then are slashed due to the Arbitrator executing their duties incorrectly.

## Future work

### Delayed dispute decision finalization
It was suggested in the review of this GIP that there could be a protocol-enforced window before the decision of a dispute can be finalized in order to provide ample time for appealing to the community and The Graph Council. This may be explored in a future GIP.

### Enforcing conventions in this charter at the protocol level
While most of the Arbitration charter describes conventions for behavior that are difficult to specify on-chain, some clauses could be enforced at the smart contract layer. This may be explored in future GIPs.

### Only make allocated stake slashable
Currently, all of an Indexer's stake may be slashed for a fault that occurred involving a single allocation. A possible change would be to allow Indexers to limit their risk by only having the allocated stake associated with a fault be slashable. This also removes a disincentive for Indexers to create many simultaneous allocations. This would require a change to the smart contracts and may be explored in a future GIP.

### Make shorter allocations slashable for a smaller amount that longer allocations for indexing disputes
Currently, an Indexer is punished for closing their allocations more frequently, as they submit more PoIs in this process, which creates more opportunities to be slashed. An alternate design would be to set a slashing percentage per epoch of indexing, such that longer allocations may be slashed for a greater percentage than shorter allocations. This would require a smart contract upgrade and may be explored in a future GIP.


# Copyright Waiver

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
