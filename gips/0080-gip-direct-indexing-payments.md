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

![image](https://github.com/user-attachments/assets/1182cc24-9adf-4aea-969a-954b9364389f)

### Post-MVP Contracts-Based Implementation
The Post-MVP phase depends on the Subgraph Service implementation from Graph Horizon, and uses a trust-minimized payment mechanism for Indexers to collect payment from an Escrow (that is the same Escrow used by TAP for query fee payments). 

The Gateway commits to paying the specified price with a voucher, that is posted on-chain by the Indexer, and that the contracts will use as a pre-authorization. 

This allows the Indexer to collect payment directly when posting the POI, and having a guarantee that the payment will be available when the job is done (as long as the Escrow is sufficiently funded). 

At this phase, we will likely also iterate on the pricing approach and introduce a better "indexing gas" metric as an evolution to the MVP entities-and-blocks-based approach. 
