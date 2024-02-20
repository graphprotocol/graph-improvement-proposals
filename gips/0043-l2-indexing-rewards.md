---
GIP: "0043"
Title: Activate indexing rewards in The Graph Network on Arbitrum One
Authors: Pablo Carranza Vélez <pablo@edgeandnode.com>, Ariel Barmat <ariel@edgeandnode.com>, Tomás Migone <tomas@edgeandnode.com>
Created: 2023-02-01
Updated: 2023-02-01
Stage: Candidate
Category: Protocol Operations
---

# Abstract

In this GIP we propose to upgrade the protocol contracts to support indexing rewards on the L2 as described in GIP-0037. That GIP proposed changing indexer rewards issuance to follow a linear function instead of the current exponential / compound interest formula, with the GRT for rewards minted natively in L1 and L2. In addition, we propose activating indexing rewards in Arbitrum One using a small initial step of 5% to 10% of the total issuance rate.

# Motivation

The Graph Protocol contracts have been deployed to Arbitrum One in block **42,449,166** as proposed in [GIP-0040](https://forum.thegraph.com/t/gip-0040-l2-bridge-and-protocol-deployment/3695) and approved by the Council in [GGP-0017](https://snapshot.org/#/council.graphprotocol.eth/proposal/0x8c3c94e6a3064023eac582d609041524c1f41b70bece62f6c58d303faa42b7e8). Since then, all the participants can run indexers and subgraphs on Arbitrum One. However, indexers don't have any additional incentive to query fees collected from consumers. This GIP proposes to activate indexing rewards on Arbitrum One using a small initial step of 5% to 10% of issuance to increase the supply of indexers on the L2 deployment of The Graph Network.

# High-Level Description

Enabling indexing rewards in Arbitrum One involves several steps that we can divide into three parts:

### Part One: Upgrade

- Upgrade the Rewards Manager in Ethereum Mainnet (L1) to use the new linear issuance mechanism.

- Upgrade the L1GraphTokenGateway in Ethereum Mainnet (L1) to add extra protections for minted tokens coming from L2.

- Upgrade the Rewards Manager in Arbitrum One (L2) to use the new linear issuance mechanism.

- Council calls `RewardsManager.updateAccRewardsPerSignal()` in L1 to storage all pending accrual.

- Set indexing rewards configuration in terms of issuance speed instead of a percentage of the total token supply on Ethereum Mainnet (L1). We can calculate the current total issuance rate as speed in GRT per block:

$$ totalIssuancePerBlock = totalSupply \times issuanceRatePerYear \div blocksPerYear $$
$$ totalSupply = 10,576,000,000 $$
$$ issuanceRatePerYear = 0.03 $$
$$ ethBlockTime = 12 \text{sec}\ $$
$$ blocksPerYear = secondsPerYear / ethBlockTime $$
$$ 10,576,000,000 \times 0.03 / (60 \times 60 \times 24 \times 365 \div 12) = 120.73 \text{GRT/block}\ $$

- Finally, check that everything is properly configured and update relevant monitoring that tracks indexing rewards issuance.

### Part Two: Activate indexing rewards in Arbitrum One

- After reviewing that the upgrade is working correctly, the Graph Council will send one transaction to reduce the per-block issuance speed on Ethereum Mainnet (L1) and another one to increase it on Arbitrum One (L2).

- The Council must ensure that the transactions to L1 and L2 get mined as close as possible to avoid a gap in the issuance rate.

- Considering that total issuance speed is now the combined speed of L1 and L2:
  $$ totalIssuancePerBlock = issuancePerBlockL_1 + issuancePerBlockL_2 $$

To set the rewards distribution in Arbitrum One to be 10% of the global issuance, the Council should set the issuance speed this way to keep the global issuance at the same rate:

$$ issuancePerBlockL_1 = 0.9 \times totalIssuancePerBlock $$
$$ issuancePerBlockL_2 = 0.1 \times totalIssuancePerBlock $$

### Part Three: Set L2 Bridge enhanced protections

One of the features introduced by GIP-0037 implementation is a "Mint Allowance Protection" as an extra line of defense against uncontrolled minting of L1 GRT after a large withdrawal from the L2.

- The Council must set this value to follow the issuance speed on L2 using `RewardsManager.updateL2MintAllowance()` to match the configuration used in Part Two.

# Additional Considerations

- After the Council upgrades the contracts to use a linear reward distribution, it will lead to less issuance over a very long period. The Council might consider reviewing the issuance per block parameter to match a desired target inflation to total supply.

- This GIP kickstarts the activation of indexing rewards in Arbitrum One. However, the Graph Council will need to decide on future updates in the distribution parameters to migrate most and eventually all of the reward issuance to L2.

# Related Work

For additional details, please read the related work in the following GIPs:

- **GIP-0037: The Graph Arbitrum deployment with linear rewards minted in L2**
  https://forum.thegraph.com/t/gip-0037-the-graph-arbitrum-deployment-with-linear-rewards-minted-in-l2/3551

- **GIP-0040: L2 Bridge and Protocol Deployment**
  https://forum.thegraph.com/t/gip-0040-l2-bridge-and-protocol-deployment/3695

The feature is implemented in [PR-700](https://github.com/graphprotocol/contracts/pull/700)

# Risks and Security Considerations

Please read [GIP-0037](https://forum.thegraph.com/t/gip-0037-the-graph-arbitrum-deployment-with-linear-rewards-minted-in-l2/3551#risks-and-security-considerations-20) for a detailed description.

# Validation

- The code was audited by OpenZeppelin and the audit report can be found here: [OpenZeppelin Audit](https://github.com/graphprotocol/contracts/blob/12f17994f73603ff683abaaf53d19b8a588dbf8b/audits/OpenZeppelin/2022-11-graph-linear-rewards.pdf)

- Edge & Node team is running the test and deployment plan that involves simulating the upgrade, activating the rewards and checking that the issuance is correct.

# Copyright Waiver

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
