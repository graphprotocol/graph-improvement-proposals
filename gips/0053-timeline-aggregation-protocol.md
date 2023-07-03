---
GIP: 0053
Title: Timeline Aggregation Protocol
Authors: 
Bryan Cole <bryan.cole@semiotic.ai>
Severiano Sisneros <severiano@semiotic.ai>
Alexis Asseman <alexis@semiotic.ai>
Zachary Burns <zac@edgeandnode.com>
Theo Butler <theo@edgeandnode.com>
Gokay Saldamli <gokay@semiotic.ai>
Tomasz Kornuta <tomasz@semiotic.ai>

Created: 2023-05-23
Updated: 2023-06-03
Stage: Draft
Discussions-To: <The forum post where discussion for this proposal is taking place.>
Category: Protocol Logic
Depends-On: -
Superseded-By: -
Replaces: -
Resolves: -
Community-Polls: -
Governance Proposals: -
Implementations: TAP <https://github.com/semiotic-ai/timeline_aggregation_protocol>, TAP Contracts <https://github.com/semiotic-ai/timeline-aggregation-protocol-contracts>
Audits: TOADD
---
(*All lists should comprise comma-separated elements. All dates are in ISO 8601 format.*)

*This is a template GIP proposal showing the layout and formatting described in GIP 0001. Italicized writing are comments and should be removed or replaced. Sections marked optional may also be removed if left empty.*

*All GIPs MUST include preamble, in RFC 822 format, preceeded and followed by three dashes ("—-"). This formatting aids in compatibility with most static site generators.*

*Given the heterogeneous types of proposals that this template is meant to reflect, most sections below, with the exception of "Abstract" and "Motivation" may be treated as optional. The author, however, should exercise good judgement in deviating from the template, as they are reflective of the type of information, when relevant, that editors will look for in reviewing a proposal.*

# Abstract

This GIP aims to establish a trust-minimized system of payment between Gateways and Indexers. The primary change proposed by this GIP is to extend the existing Scalar payment system by which Indexers and Gateways track and settle payments for queries within a specified allocation period.

The proposed protocol is called Timeline Aggregation Protocol (TAP). The TAP is designed to protect against potential fraudulent activity by ensuring that:
- each receipt and receipt aggregate is unique to a specific Gateway,
- each receipt can only be used once to claim a debt, and
- each receipt can only be used for the allocation in which it was issued.

The protocol also enables the Indexers to efficiently prove "on-chain" total sum of receipts for the queries they served.

# Motivation

- Minimizing trust requirements between payers and payees
- Creating a system which is scalable in terms of both query and participants
- Fast and robust aggregation of per-queries receipts
- Cheap on-chain verification of receipt aggregates


# Prior Art

TAP is an add-on to Scalar, a high throughput and low-latency microtransaction system enabling pay-per-query service between the Indexers and Gateways.
More info about Scalar can be found in [The Graph Foundation unveils Scalar: a microtransaction for every query](https://thegraph.com/blog/scalar/) blog post. We call the resulting combination Scalar TAP.

More on the prior art can be found in `Rationale and Alternatives` section

# High Level Description

The goal of Timeline Aggregation Protocol (TAP) is to enable the Indexers to aggregate and collect fees for all the queries they served within a specified allocation period. It functions on a few primary components: Receipts, Receipt Aggregate Vouchers (RAVs), and Allocations, in order to manage services and payments between Gateways and Indexers.

## Definitions

* Gateway - a distributed entity sharing a public/private key pair, pays Indexers for their  service
* Indexer -  provides a service for a Gateway (potentially many Gateways) and collects payments
* Receipt - a promissory note from a Gateway to an Indexer
* Receipt Aggregate Voucher (RAV) - a message from a Gateway to an Indexer which can be used to redeem a debt
* Gateway Receipt Aggregator - a Gateway service responsible for aggregating the Receipts into RAVs per Indexer request
* Allocation - defines a time span for which Receipts/RAVs are valid. RAVs can only be used once per allocation to claim a debt

## How does the protocol protect the integrity of receipts/RAVs?

* Each receipt has a debt value, a unique identifier for the note, a timestamp indicating when the note was created, and a unique identifier for the allocation for which the receipt was issued.
* Each receipt is signed by the Gateway using a Gateway specific signing key, binding the signed receipt to the specific Gateway
* Each RAV has a total debt value, a timestamp indicating the timestamp of the newest receipt in the accumulation, and a unique identifier for the allocation.
* Each RAV is signed by the Gateway using a Gateway specific signing key, binding the signed RAV to the specific Gateway

## Main TAP Flow

A Gateway pays an Indexer for query serving. On the other hand, an Indexer provides these services to a Gateway and collects payments in return. Fig. 1 presents the most important interactions between Gateway, Indexer and Gateway Receipt Aggregator.

| ![tap_diagram.png](../assets/gip-0053/tap_diagram.png) |
| - |
| <b>Fig 1: Diagram presenting the most important interactions between the core components of the protocol.</b>|

Payments from Gateways to Indexers are represented as receipts, which are essentially promissory notes that signify a debt owed by the Gateway to the Indexer. Gateway continuously sends TAP receipts along with query requests to the Indexers. Indexer serves those queries and stores the associated receipts. 

Once enough receipts have been collected, Indexer sends a Receipt Aggregate Voucher (RAV) request, sending a batch of receipts up to some timestamp (along with the previous RAV). Timestamp should be slightly further in past to ensure all receipts up to that timestamp have arrived.

Once a RAV request has been received, the Gateway Receipt Aggregator aggregates receipts associated with the request and issues a new Receipt Aggregate Voucher (RAV), a message which serves as a bundle of the receipts collected over a certain time period, and can be used by the Indexer to redeem the total debt owed. The period over which these receipts and RAVs are valid is referred to as an allocation, and each RAV and receipt can only be used once during its allocation to claim a debt.


# Detailed Specification

## Data structures

### TAP Receipt

TAP Receipts are used as proof of value owed from gateway to indexer. A single TAP Receipt contains the following items:
* Allocation ID - Number unique to each allocation that is constant for all receipts in that allocation. Stored immutably on-chain at allocation open.
* Value - Amount to be paid for the query being served
* Timestamp - Unix timestamp of when the gateway generated the receipt
* Random Nonce - Randomly selected number used concatenated with timestamp to create a unique ID for receipt within an aggregation batch
* Signature - ECDSA signature of all other items in receipt

| ![tap_receipt.png](../assets/gip-0053/tap_receipt.png) |
| - |
| <b>Fig. 2: TAP Receipt structure</b>|

Note we are using [EIP-712](https://eips.ethereum.org/EIPS/eip-712) for hashing and signing of TAP Receipts.

### RAV Request

At any point in time an Indexer can request a new RAV by sending:
* Batch of Valid Receipts: This includes all receipts with a timestamp greater than TS1 (if not including a RAV consider TS1 allocation open timestamp) and less than the some target timestamp (TS2), e.g. Batch 2
* Most Recent RAV (optional): The aggregate amount on this RAV (Agg1) will be added to the new total(Agg2) and the timestamp(TS1) will be used to indicate which receipts are invalid

### Receipt Aggregate Voucher (RAV)

Receipt Aggregate Voucher (RAV) is a response message sent by the Gateway Receipt Aggregator. It contains a ECDSA signed tuple of:
* Allocation ID - A value unique to each Allocation
* Value - Aggregate amount from all receipts in the batch plus the aggregate from a previous RAV if provided
* Timestamp corresponding to max timestamp from all receipts being aggregated

For hashing and signing RAVs we also leverage [EIP-712](https://eips.ethereum.org/EIPS/eip-712).

| ![tap_receipt_aggregation.png](../assets/gip-0053/tap_receipt_aggregation.png) |
| - |
| <b>Fig. 3: TAP RAV structure and aggregation of batches of TAP Receipt</b>|



## TAP APIs

TAP protocol is open-source and its source can be found [here](https://github.com/semiotic-ai/timeline-aggregation-protocol).

Detailed documentation can be found on docs.rs:
* For TAP core crate:
    - [README](https://docs.rs/crate/tap_core/0.1.0)
    - [Docs](https://docs.rs/tap_core/0.1.0/tap_core/)
* For TAP Aggregator:
    - [README](https://docs.rs/crate/tap_aggregator/0.1.0)
    - [Docs](https://docs.rs/tap_aggregator/0.1.0/tap_aggregator/)


### TAP Manager and Adapters 

TAP Manager is a high-level abstraction for TAP protocol. It provides several methods for:
* verification & storing of receipts
* preparation of RAV requests
* verification & storing RAVs

TAPManager uses 4 adapters:
* ReceiptStorageAdapter - for storing and managing the lists of receipts
* RAVStorageAdapter - for storing and managing the RAVs
* CollateralAdapter - for accessing/managing the collateralization
* ReceiptChecksAdapter - for checking receipts, allocation etc.

Detailed documentation can be found in [TAP Core docs.rs](https://docs.rs/tap_core/0.1.0/tap_core/)

### TAP Manager receipt and RAV request checks

When an Indexer receives a receipt (that comes along with an associated query), it has to perform several validations, namely it needs to check if:
* receipt is unique (`check_uniqueness`)
* the allocation is correct and open (`checkt_AllocationID`)
* receipt timestamp is correct (`check_timestamp`)
* value of the receipt matches it's price bid according to the latest Indexer's Agora model (`check_value`)
* Gateway signature is correct (`check_signature`)
* collateral is sufficiend to collect the payment once the query is served (`check_collateral`).

The order of execution of those checks is presented in fig. 4.

| ![tap_manager_receipt_checks.png](../assets/gip-0053/tap_manager_receipt_checks.png) |
| - |
| <b>Fig. 4: (Optional) receipt checks performed when a new receipt is received by the Indexer</b>|

Please note those checks are done as part of the critical path when an Indexer is serving a query.
This might impact the total serving time.
Hence TAP enables the Indexers to postpone all/or some of those checks to be executed during the RAV request preparation,
as presented in fig. 5.
In short, during RAV request preparation TAP manager checks if a given receipt passed all checks - and if some are missing it will run them at the very moment.

| ![tap_manager_rav_request_checks.png](../assets/gip-0053/tap_manager_rav_request_checks.png) |
| - |
| <b>Fig. 5: (Mandatory) receipt checks performed by the Indexer when preparing RAV request</b>|

Please refer to [TAP Manager documentation](https://docs.rs/tap_core/0.1.0/tap_core/tap_manager/struct.Manager.html) for more details on this topic.



### Gateway Receipt Aggregator

Gateway Receipt Aggregator is a A JSON-RPC service that lets clients request an aggregate receipt (RAV) based on a list of individual receipts. High-level specification can be found [TAP Aggregator README](https://docs.rs/crate/tap_aggregator/0.1.0).

Detailed documentation can be found in [TAP Aggregator docs.rs](https://docs.rs/tap_core/0.1.0/tap_core/).

## Smart Contracts (Tom/Bryan | Status: OK)

TAPVerifier - A contract for verifying receipt aggregation vouchers.

Collateral - This contract allows `senders` to deposit collateral for specific `receivers`, which can later be redeemed using Receipt Aggregate Vouchers (`RAV`) signed by an authorized `signer`. `Senders` can deposit collateral for `receivers`, authorize `signers` to create signed `RAVs`, and withdraw collateral after a set `thawingPeriod` number of seconds. `Receivers` can redeem signed `RAVs` to claim collateral.
This contract uses the `TAPVerifier` contract for recovering signer addresses from `RAVs`.

AllocationIDTracker - This contract tracks the allocation IDs of the RAVs that have been submitted to ensure that each allocation ID is only used once. It is external to the collateral contract to allow for updating the collateral contract without losing the list of used allocation IDs.
This contract is intended to be used with the `Collateral` contract.

Implementation along with detailed API documentation of TAP contracts can be found [here](https://github.com/semiotic-ai/timeline-aggregation-protocol-contracts).

# Backwards Compatibility

## Gateway:
Indexers are currently expected to report their software version (via the `/version` path of indexer-service). During the transition period, the gateway will use this version to determine if the indexer can be expected to support TAP. If so, the gateway will only send receipts to that indexer using TAP. Otherwise, the current Scalar receipt format will be used. This will ensure that any query sent to an indexer from a gateway will only be associated with one receipt, and that receipt will only be associated with one of the two Scalar protocol versions.

## Indexers:
We assume no backwards compatibility for the Indexer Service, i.e. once the Indexers will upgrate their stack, then they will switch to the TAP Scalar payments.
Hence the protocol will have to make advertise that this is a breaking change and Indexers will have to redeem your receipts and close all the allocations before this upgrade.

## Smart contacts:
Similarly to Indexer Service, update to new stack will automatically trigger switching to new Collateralization and TAP Verifier smart contracts.


# Risks and Security Considerations

At its core TAP relies on cryptographic protocols to ensure the integrity and authenticity of the receipts and RAVs issued by the Gateway. When building any system relying on cryptography, special care must be taken to ensure that the system is built on sound cryptographic primitives, that no assumptions are being made which might invalidate the security provided by those primitives, and that the implementation of those primitives is secure. 

A key design feature of TAP is that it is simple; it relies on standard cryptography, does not require special assumptions about the underlying cryptographic primitives, and the reference implementation is built on top of well known open source libraries. Specifically, receipts and RAVs are signed using the deterministic variant of the standardized ECDSA signature algorithm (see IETF RFC6979) with the secp256k1 elliptic curve, the same as currently used in Ethereum for signing transactions. Receipt messages are hashed according to the EIP-712 specification. The reference TAP implementation was built in Rust using well known open source libraries for all cryptographic operations, specifically signature generation and verification; building on the contributions of the community and mitigating the risks of “rolling your own cryptography”. 

Additionally, both the protocol and the implementation were assessed for security vulnerabilities internally and externally by independent teams. Findings from those assessments were incorporated into the design specified in this GIP. Detailed risks analysis can be found in:
[2023.03 TAP security.pdf](../assets/gip-0053/2023.03_tap_security.pdf)

Finally, the protocol and the reference implementations are open source. Abiding by Kerckoffs’s principle and providing the opportunity for the protocol to be further improved by the community. 



# Validation

We got the protocol reviewed by an external consultant. Details can be found in:
[2023.03 TAP Discussion (External Review).pdf](../assets/gip-0053/2023.03_tap_discussion_external_review.pdf)

We will also schedule an audit of all the implemented smart contracts before their deployment.

# Rationale and Alternatives

The journey to Graph-scale payments is storied and branched into several potential solution spaces. What follows is an abbreviated history of things we tried and why they didn't work.

## Other state channel systems
Before creating Scalar, we attempted to integrate several pre-existing or in-development state channel systems, including but not limited to Vector, Nitro, and Indra. Although each system had unique tradeoffs, each exhibited one or more design flaws that manifested in a total system breakdown at scale.

### Slow, serial channel creation
The ability to create channels quickly is desirable when the system is starting up, cycling allocations, handling long-running requests, or experiencing unexpected load. The systems we evaluated throttled state channel creation, usually by requiring participants to update a shared data structure (such as a Merkelized root of outstanding channels). The result of high-latency, low-throughput state channel creation ranged from degraded quality of service to denial-of-service attack vulnerability.

### Inefficient use of capital 
The systems we evaluated allocated capital to individual state channels rather than the state channel pool. Resource partitioning is generally wasteful. And in our case, the wasted cost of capital increased the system's friction.

### Fragile failure modes
The systems we evaluated were designed around the happy path, with failure modes as an afterthought. In a distributed system, however, "exceptional cases" are a common occurrence. Rather than degrading safely to a reasonable state, some systems would get stuck and require human intervention due to what should have been innocuous events like network failures or race conditions.

### Trust Assumptions
Scalar TAP reduces the amount of trust required between participants. But, even the previous, more trusted version of Scalar had lower trust requirements than the state channel systems we evaluated. The trust problem that surfaces when The Graph uses state channels is that with many small state channels, collecting the channels in a dispute can cost more than the total value locked in the channels. That means that the participants are assuming collaboration, and in the worst case, the system was effectively a fancy wrapper over asking people nicely to pay at the end.

### Thawing requirements
Some systems require payees to wait before collecting their payments, even though this increases the cost of capital in the happy path.

### Query latency
Many systems incurred high latency on the critical path to serving a query, such as database writes. Even with negligency that appears negligible in isolation (say, 20 microseconds), if all systems involved in serving a query take a lax approach toward performance, the latency would add up to significant amounts.

### Inefficient use of bandwidth
Some systems made so many requests to synchronize state that it saturated the network pipeline, incurring high costs and leaving no room for serving queries.

## SNARKs
One way to batch-collect state channels would be to use a SNARK to prove the knowledge of many signed receipts without making the individual receipts available for verification on-chain. Although the exploration was promising, we failed to meet our goals using SNARKs.
Specifics can be found in the "A SNARKs Tale: A story of Building SNARK Solutions on Mainnet" talk delivered at Devcon Bagota by Severiano Sisneros in 2022 [youtube](https://www.youtube.com/watch?v=GnYM9yxIRSs).

### High cost
Based on an analysis of historical on-chain data, we set a target of being able to collect 1 million state channels for less than $30. This target would allow for the net-positive collection of most query fees. Unfortunately, we could not reduce the total cost enough. Factors contributing to cost were mainly prover time and memory, proof size, and verify compute.

### Trust assumptions
Another consideration with SNARKs is whether you are taking on any additional trust assumptions. With a transparent SNARK, users do not need to accept trust assumptions from a trusted ceremony. But non-transparent SNARKs are typically more efficient.

## Homomorphic Signatures
The final alternative we considered was homomorphic signatures. This avenue seemed promising because summing signed numbers is what our aggregation requires and what homomorphic signature schemes are designed for.

Unfortunately, simultaneously achieving robustness, speed, and security with these systems was impossible. The main problem we ran into was that if a receipt was signed twice, the system was insecure for the sender, but if a receipt was not received, the receiver could not collect fees. The closest match to our needs using this system relied heavily on micro-trust. The idea of keeping the micro-trust and removing the novel cryptography led us to the final design of Scalar TAP.

# Copyright Waiver

All the code related to TAP (TAP protocol, TAP aggregator, TAP manager, smart contracts and tests) is distributed under [Apache v2 license](https://www.apache.org/licenses/LICENSE-2.0).

Copyright and related rights to this GIP are waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
