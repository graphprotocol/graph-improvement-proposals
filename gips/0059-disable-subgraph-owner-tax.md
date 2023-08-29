---
GIP: "0059"
Title: Disable subgraph owner tax when publishing a new version
Authors: Ariel Barmat <ariel@edgeandnode.com>, Adam Fuller <adam@edgeandnode.com>
Created: 2023-08-05
Updated: 2023-08-05
Stage: Candidate
Category: Economic Parameters
---

# Abstract

We propose setting the owner tax on the GNS to 0%, effectively disabling this mechanism that works any time a subgraph owner publishes a new version.

# Motivation

The idea of introducing an "owner tax" was to prevent a grieving attack in which a subgraph owner repeatedly publishes new versions and then drains its curators via the "curation tax".

The consequence is that it also introduces friction every time an honest subgraph owner updates their versions:

- They need to have GRT available in the wallet everytime they publish a new version. This in addition to having ETH to pay for transaction fees.

- Mental cost of planning the amount of GRT they need for paying the tax. If the curation pool is big enough it can amount to a high value.

Our belief is that this adds unnecessary high friction for little gain considering the reality of the curation market. 

Our observation about how the curation market developed in the last two years shows that subgraph owners do most of their own curation as they are naturally incentivized to have their subgraph indexed on the network. Additionally, we believe that if a subgraph owner misbehave it will be penalized by curators and they will then be more careful at verifying the accounts that create those subgraphs before supporting them.

# Specification

The protocol introduces a number of parameters to protect against undesired effects that can affect participants. One of such is the `ownerTax` which is intended to protect against a rogue subgraph owner draining the curation pool by repeated version updates.

To disable the `ownerTax`, governance must execute a transaction on the GNS contract `GNS.setOwnerTaxPercentage(0)` that will then emit the event `ParameterUpdated("ownerTaxPercentage")`. Any further call to `GNS.publishNewVersion()` will not require to pull GRT.

Participants involved in this proposal:

- Subgraph Owner: The account that owns the admin right to publish new versions on any particular subgraph.
- Subgraph Curator: Participant that deposits GRT into a subgraph curation pool to support its indexing.
- Governance: The Graph Council.

# Backwards Compatibility

The change does not break any interface and only requires a governance action.

# Risks and Security Considerations

There is a risk of a Subgraph owner to publish new versions multiple times which will gradually drain the curation pool for that subgraph via the charing of the curation tax. This can happen at no gain for the attacker because the funds collected through the tax are fully burned. We will continue monitoring usage and behaviour to understand if this becomes an issue and will propose follow up changes as required.

# Rationale and Alternatives

An alternative would be to remove the owner tax functionality entirely via a protocol upgrade. We believe that is not necessary as setting it to 0% will avoid the code branch entirely and leave the door open to restoring it. Governance can remove the parameter implementation on a successive upgrade once the belief behind the motivation is proven.

# Copyright Waiver

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
