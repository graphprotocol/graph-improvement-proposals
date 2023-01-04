---
GIP: 0040
Title: L2 Bridge and Protocol Deployment
Authors: Pablo Carranza Vélez <pablo@edgeandnode.com>, Ariel Barmat <ariel@edgeandnode.com>, Tomás Migone <tomas@edgeandnode.com>
Created: 2022-11-03
Updated: 2022-11-03
Stage: Draft
Category: Protocol Operations
---

# Abstract

This GIP proposes the deployment of The Graph contracts on Arbitrum from the latest [published version](https://github.com/graphprotocol/contracts/tree/pcv/l2-bridge). This includes a mirror of the contracts currently deployed on mainnet with no indexing rewards plus an L1 to L2 bridge.

# Motivation

As mentioned in GIP-0031, there is an interest in the community to move to an L2 to benefit from gas savings, and Arbitrum is thought by many to be the best candidate at this time. It is desirable to move to L2 gradually, however, as Arbitrum is still in beta and such a big protocol change is not without risks. Therefore, we've proposed having an experimental stage where rewards are not distributed in L2.

# High Level Description

We propose a gradual move to L2 by deploying the new L2 protocol in parallel to the existing L1 deployment. Initially, this new deployment would be experimental because indexing rewards would be disabled. Eventually, governance can enable reward distribution.

The L2 network will mostly work like the existing L1 network. Most of the contracts will be deployed with no functional changes, so staking and curation can be done in the same way as in L1. To participate in the L2 network, users can move GRT to L2 using the bridge proposed in [GIP-0031](./0031-arbitrum-grt-bridge.md), though future GIPs can propose ways to facilitate migration of staked tokens or subgraphs and curated signal.

We propose deploying the L2 in _3 stages_, though this GIP is only about the deployment of the first stage:

- **Stage 1:** Deploy a mirror of L1 contracts on Arbitrum, including an L1-L2 bridge with _disabled_ indexing rewards. In addition to that, we propose to deploy Curation in L2 with a flattened bonding curves as explained in GIP-0039.
- **Stage 2:** Upgrade the protocol to support indexing rewards on L2.
- **Stage 3:** Add migration helpers to facilitate users moving stake, delegations and subgraphs from L1 to L2.

A similar GIP to this will be written to propose the deployments of phase 2 and 3 in the future.

# Related Work

This GIP proposes the deployment of the updates described in **GIP-0031 and GIP-0039** and giving an outline about the different stages.

**Stage 1: L2 Bridge and Protocol Deployment**

- **GIP-0031: An Arbitrum GRT Bridge**
  https://forum.thegraph.com/t/gip-0031-arbitrum-grt-bridge/3305

- **GIP-0039: Curation 1.x**
  https://forum.thegraph.com/t/gip-0039-curation-v1-x/3613

**Stage 2: L2 Rewards**

- **GIP-0034: The Graph Arbitrum deployment with rewards distribution using drip mechanism** https://forum.thegraph.com/t/gip-0034-the-graph-arbitrum-deployment-with-a-new-rewards-issuance-and-distribution-mechanism/3418

- **GIP-0037: The Graph Arbitrum deployment with linear reward distribution minted on L2** https://forum.thegraph.com/t/gip-0037-the-graph-arbitrum-deployment-with-linear-rewards-minted-in-l2/3551

# L1 vs L2 Contracts Changes

Most of the protocol contracts are the same as the ones deployed on Mainnet Ethereum. The only difference is that the GraphToken is upgradeable on L2 and adds a few more functions required by the L2 Bridge.

# New Contracts

Having a functioning L2 requires the deployment of an L1-L2 Bridge that can move state over. The bridge is comprised of three contracts that are in charge of passing cross-chain messages and lock GRT. We propose deploying the **L1GraphTokenGateway**, **BridgeEscrow** and **L2GraphTokenGateway** and set them up as the canonical bridge for GRT. Please refer to GIP-0031 for more information.

# Operations

Most of the L2 deployment can be performed using a deployer account with sufficient funds that will then transfer governance to the Council. However, certain actions require the intervention from the Council.

- **Base L2 protocol contracts:** deployed and configured by deployer, and then transferred to the Council.
- **Bridge on L2:** deployed and configured by deployer, and then transferred to the Council.
- **Bridge on L1:** deployed by deployer, _configured by the Council_.

The gateways are Paused by default when deployed, the Council will send a transaction to unpause when the deployer verifies that the configuration is complete. The Council signals the acceptance of the L2 deployment by accepting ownership of the contracts and unpausing the bridge.

After deployment, the Council will need to perform one last transaction to signal to the Arbitrum team that the Council's intent is to connect the GraphToken on L1 to the L2GraphToken on Arbitrum through the custom bridge. This will allow the bridge to be used through Arbitrum's Gateway Router.

# Dependencies

This document is proposing the deployment of the work described in GIP-0031 and GIP-0039.

# Risks and Security Considerations

Please see the Related Work section for a full risk registry.

# Validation

- GIP-0031 has been audited by OpenZeppelin. You can find the [audit report](https://github.com/graphprotocol/contracts/blob/a2a09a5aac15dd468d73f9c02618a74edafb0fff/audits/OpenZeppelin/2022-07-pr552-summary.pdf) in the contracts repo.

- The GIP-0031 code has been audited by a CodeArena bounty https://code4rena.com/contests/2022-10-the-graph-l2-bridge-contest with no critical issues found.

- The updates that this GIP propose to deploy has been tested in a private testnet using Goerli and Goerli Arbitrum Nitro according to a test plan developed by Edge & Node.

- We will make public both the test plan and the contingency plan describing potential risks and failure modes.

# Copyright Waiver

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
