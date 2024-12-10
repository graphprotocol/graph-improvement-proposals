---
GIP: "0082"
Title: Integrate Substreams into The Graph Network
Authors: Alexandre Bourget <alex@streamingfast.io>
Created: 2024-10-27
Updated: 2024-10-27
Stage: Draft
Discussions-To: *To be added later*
Category: Protocol Logic, Protocol Interfaces
Implementations: *To be added later*
---

*This document tries to express and summarize the discussions that happened in-person at the October 2024 R&D Retreat.*

# Abstract

This GIP proposes integrating Substreams, a streaming-first blockchain indexing technology, into The Graph Network. This integration will enable Substreams to be fully recognized on the network and eventually receive indexing rewards. This proposal, however, does not entail immediately enabling rewards, but consists of the first approval and direction setting.

# Motivation

Substreams provide significant performance improvements for indexing, particularly for large datasets and complex computations. Integrating Substreams into the network will allow developers to leverage these performance gains while maintaining the decentralization and security benefits of The Graph, as well as encouraging the market to use The Graph as its prime infrastructure provider.

# Detailed Specification

## Indexer Selection Algorithm (ISA)

Given that there's no gateway involved with Substreams (it's direct point-to-point between consumer and provider), the existing Indexer Selection Algorithm (ISA) does not apply.  A round-robin selection of providers will be used in the front-end (https://thegraph.market) to ensure fair distribution of request loads, while still allowing consumers to choose a provider.

## Payments

Payments for Substreams work will be on-chain. The existing `collect` function in the staking contract will be used for payment processing.  It was deemed that no TAP was required to have Substreams collect payments and honor the promises of the protocol. However, as TAP brings augmented trust minimization properties, it would be incorporated in future work.

## Economic Security and Disputes

Economic security will be enforced through slashing and disputes. Indexers providing incorrect data can have their stake slashed.

- **Attestations:** Substreams endpoints will provide signed attestations to verify data integrity. The command-line tools (`substreams gui`) will display these attestations.  To accommodate different future attestation methods, and not only an Ethereum key signature, the attestation payload will be prefixed with the protocol used (e.g., `eth:` for Ethereum). Detailed documentation will describe the attestation format and verification process.
- **Detecting Faults:** Since Substreams are streaming, Proofs of Indexing (PoIs) are not applicable. Fault detection will happen at query time. Discrepancies in data returned by different providers for the same stream and block will trigger an investigation by an Arbiter. The Arbiter will use the `substreams gui` tool to inspect and compare stream data at specific block ranges, verifying data integrity and determinism.

In the `SessionInit` message of the response, when requiring SignedAttestation, the server would reply with the `attestation_address string` protocol would be `eth`, and for future proofing, and `address` would be an `eth` address (indexer address or operator address, whatever is less risky)

### Operator keys for signing

Registration will be done by the staking address. The operator key will be used for signing, and an in-memory mapping will link the operator key to the staking key.
- The `setOperator` function ([Staking.sol#L226](https://github.com/graphprotocol/contracts/blob/ce3ec16484dacc89a1b9cf08455256be830790e3/packages/contracts/contracts/staking/Staking.sol#L226)) links the operator key to the staking key and emits the `SetOperator` event ([IStakingBase.sol#L99](https://github.com/graphprotocol/contracts/blob/main/packages/contracts/contracts/staking/IStakingBase.sol#L99)).



## Arbitration

StreamingFast will provide support for arbitration cases involving Substreams, either by hiring an Arbiter or by collaborating with individuals experienced in arbitration.  A detailed document outlining the arbitration process for Substreams discrepancies will be created.


### Tools

A command-line tool will be provided as part of the `substreams tools` command-line interface to validate attestations.  This tool will take an attestation (signature), a fixed payload (the content disputed), and validate whether it was properly signed by a slashable indexer.

The command would look like:

```
substreams tools validate-attestation --indexer-address 0x123123123 --attestation eth:120398102398102938109283012983 --replay-log-file ./path/to/replay.log
```

This command will process all messages in the provided replay log and validate their attestations. Replay logs can be generated using  `substreams gui --with-replay`.


## Permissionless Discovery

Substreams providers will register their service on-chain using DataEdge (see below). Tooling for registration will be integrated into the `substreams` command-line tool.  A dedicated Substreams module will scan for new registrations on DataEdge, perform health checks (e.g. on `/health` or `/info` endpoints) and update the front-end accordingly.  Detailed documentation will be provided for the on-chain registration payload and protocol.

### Use of contracts

References:

 - [DataEdge addresses](https://github.com/graphprotocol/contracts/blob/main/packages/data-edge/addresses.js)
- The relevant contract sources are `DataEdge` ([DataEdge.sol](https://github.com/graphprotocol/contracts/blob/main/packages/data-edge/contracts/DataEdge.sol)) and `EventfulDataEdge` ([EventfulDataEdge.sol](https://github.com/graphprotocol/contracts/blob/main/packages/data-edge/contracts/EventfulDataEdge.sol)). `EventfulDataEdge` emits an event upon registration, while `DataEdge` does not.

The exact format of the published data will be documented in the Substreams documentation and implemented in the command-line tools, as well as in the Substreams module to keep the front-end up-to-date.


### CLI for Registration

To enable on-chain registration of Substreams providers for The Graph Market, network payments tools will be augmented (or the `substreams` command-line) with a new `register` command.  This command will publish provider information to DataEdge, signed by the operator's key.

```
substreams network register --operator-wallet 0x123123123 --operator-priv-key-file my.key --service substreams --endpoint mainnet.eth.superbob.com --logo ./mylogo.png --name Pinax Networks --network ethereum
```

This command allows providers to register their service, specifying their operator wallet, private key file, the service type (`substreams`), the endpoint URL, logo, name, and the supported network. The registration data is signed by the operator's key.  An update to an existing registration can be achieved by re-running the command with updated parameters.  A corresponding `unregister` or `revoke` command would be added to allow providers to take down their service from the front-end.

A background process will continuously monitor DataEdge for new registrations and updates.  Upon detecting a new registration, the system will perform health checks against the provided endpoint (e.g., calling `/health`, `/head`, or `/info`) to verify the service's availability and type. Only after successful health checks will the provider be added to the `providers.json` file, which is used to populate the front-end UI.  This ensures that only active and validated providers are displayed to users.  The system will incorporate block time checks to prevent displaying stale registrations until the block height is sufficiently close to the chain tip.


# Dependencies & Backwards Compatibility

This GIP has no dependencies. Most of the elements described above are already rolled out and work. What is missing will be executed provided there is general approval of the direction this integration takes.

To account for signed attestation, some backwards compatible fields will be added to the Substreams requests and responses.  These fields will be optional and will not impact existing Substreams implementations.

# Risks and Security Considerations

Risks associated with data integrity and malicious actors will be mitigated through signed attestations, the arbitration process, and economic security enforced via slashing. The performance impact of signed attestations will be evaluated, with opt-in mechanisms available.

# Copyright Waiver

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
