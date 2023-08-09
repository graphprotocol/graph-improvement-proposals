
---
GIP: <0056>
Title: Permissionless payment
Authors: Zac Burns <zac@edgeandnode.com>
Created: 2023-07-28
Updated: 2023-07-28
Stage: Candidate
Discussions-To: https://forum.thegraph.com/t/permissionless-payers-gip/4397
Implementations: https://github.com/graphprotocol/contracts/pull/749
Audits: https://github.com/graphprotocol/contracts/blob/765d43f1d0ea18789ca2e4fad31fdbdd620732c5/audits/Trust/2023-02-operator-decentralization-pr749.pdf, https://github.com/graphprotocol/contracts/pull/853
---

# Abstract

This proposal removes the limitation of specific governance-approved entities to pay query fees to Indexers in the network.

# Motivation

Community members have expressed interest in paying query fees directly through the protocol rather than through a pre-approved Gateway. Use cases include, but are not limited to:
* Gateways such as Playgrounds Analytics differentiated from the existing Gateway offerings
* Indexer-to-Indexer services, such as Firehose
* Large dapp developers who operate at a scale large enough not to need a Gateway
* Gateway redundancy
The mission of The Graph is to enable a flourishing data ecosystem with a decentralized protocol. By removing permissions, we unblock competition, leading to a more flourishing ecosystem. And we make the system more decentralized by increasing censorship resistance.


# High-Level Description

The code changes remove the `assetHolders` mapping from the staking contract and all associated functions, checks, and events.

# Risks and Security Considerations

Until Scalar TAP is deployed, Indexers must trust that a payer will sign the required receipt to collect fees. (You can read about Scalar TAP at GIP 0054-timeline-aggregation-protocol.) The current trust model is that an Indexer may delegate their trust to The Graph Council members, who vet Gateways who are long-term incentive-aligned with the network. After this change, Indexers must vet payers until Scalar TAP is deployed. It is worth noting that the Indexer software already has an allowlist, so no action is necessary if an Indexer does not want to participate with more Gateways. This change will simplify Scalar TAP by removing the need for governance to leak into its design.

Some have expressed concern that opening the Gateway role could lead to brand dilution. This concern is orthogonal to this proposal because The Graph Foundation owns the Graph trademark. Participating in the network does not automatically infer usage rights to The Graph branding.

Another concern is that opening the Gateway role could lead to protocol disintermediation. A Gateway could initially use The Graph network as a backend and then disintermediate it after building a reputation with its customers. This possibility is considered a necessary risk of decentralization. Our thesis is that a decentralized network will win the market on its merits. And our response will be to continuously improve The Graph protocol so that it remains the best option for consumers.

# Validation

Trust Security has audited this change. Their findings can be found [here](https://github.com/graphprotocol/contracts/blob/765d43f1d0ea18789ca2e4fad31fdbdd620732c5/audits/Trust/2023-02-operator-decentralization-pr749.pdf) and [here](https://github.com/graphprotocol/contracts/pull/853).

# Copyright Waiver

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).


