---
GIP: 0037
Title: The Graph Arbitrum deployment with linear rewards minted in L2
Authors: Pablo Carranza Vélez <pablo@edgeandnode.com>, Ariel Barmat <ariel@edgeandnode.com>, Tomás Migone <tomas@edgeandnode.com>
Created: 2022-09-02
Updated: 2022-09-06
Stage: Candidate
Discussions-To: https://forum.thegraph.com/t/gip-0037-the-graph-arbitrum-deployment-with-linear-rewards-minted-in-l2/3551
Category: "Protocol Logic", "Protocol Interfaces", "Economic Parameters"
Depends-On: GIP-0031
Implementations: https://github.com/graphprotocol/contracts/pull/700
Audits: TODO
---

# Abstract

This GIP is presented as an alternative to [GIP-0034](./0034-graph-arbitrum-deployment.md). Like GIP-0034, it introduces a new deployment of The Graph's smart contracts to the Arbitrum One Layer 2 blockchain. It proposes a plan to run the protocol on Ethereum mainnet (L1) together with the protocol on L2 instead of fully migrating in one go. The community would gradually move to L2 by starting with an experimental phase, in which indexing rewards are disabled, and then slowly increasing the amount of rewards in L2. To enable this, and instead of what is proposed in GIP-0034, this GIP proposes changing indexer rewards issuance to follow a linear function (instead of the current exponential / compound interest formula), with the GRT for rewards minted natively in L1 and L2.

# Motivation

As mentioned in GIP-0034, there is an interest in the community to move to an L2 to benefit from gas savings, and Arbitrum is thought by many to be the best candidate at this time. It is desirable to move to L2 gradually, however, as Arbitrum is still in beta and such a big protocol change is not without risks. Therefore, we've proposed having an experimental stage where rewards are not distributed in L2. We still need to figure out how to distribute rewards in L2, as it's not trivial to simply allow minting new GRT in L2.

However, the implementation for GIP-0034, which introduces new Reservoir contracts with a periodic rewards drip, ended up being quite complex, and will require a lot of monitoring to ensure the drip is working correctly. While we still believe GIP-0034 could work well, it seemed reasonable to explore potentially simpler alternatives, which is why we've come up with this alternative proposal, with some different risks and benefits.

# Prior Art

As with GIP-0034, we've looked closely at the way Livepeer carried out their L2 migration, as described in [LIP-73](https://github.com/livepeer/LIPs/blob/master/LIPs/LIP-73.md). They went for a full migration, whereas we're proposing a more gradual approach where L1 and L2 coexist.

# High Level Description

As in GIP-0034, we propose a gradual move to L2 by deploying the new L2 protocol in parallel to the existing L1 deployment. Initially, this new deployment would be experimental because indexing rewards would be disabled. Eventually, governance can enable rewards by increasing the issuance per block on L2 and reducing it in L1. This new network would be deployed on Arbitrum One (after testing on Arbitrum Goerli).

At a high level, the L2 network will mostly work like the existing L1 network. Most of the contracts will be deployed unchanged, so staking and curation can be done in the same way as in L1. To participate in the L2 network, users can move GRT to L2 using the bridge proposed in [GIP-0031](./0031-arbitrum-grt-bridge.md), though future GIPs can propose ways to facilitate migration of staked tokens or subgraphs and curated signal.

The main change to support the L2 deployment will be a redesign of how indexer rewards are issued and distributed both in L1 and L2. Currently, rewards for subgraphs and allocations are computed and snapshotted when signal or allocations change, and then minted and immediately distributed to indexers and delegators when an allocation is closed. This is shown in Figure 1.

![Figure 1: Current implementation of rewards issuance and distribution in L1](../assets/gip-0037/L1%20rewards%20issuance%20and%20distribution.png)

Rewards are accrued following a compound interest formula, and therefore GRT issuance follows an exponential curve, currently set with an issuance rate for a 3% annual rate.

In this GIP we propose simply replicating the same mechanism in L2, as shown in Figure 2, but changing the issuance formula to be linear. Additionally we propose adding some protections on the bridge to mitigate risks associated with minting tokens in L1 when bridging tokens from L2.

![Figure 2: Proposed rewards distribution mechanism for L1+L2](../assets/gip-0037/L1_L2%20rewards%20issuance%20and%20distribution%20(linear%20minting%20in%20L2).png)

Since GRT can now be minted in L2, the bridge needs to be able to mint new GRT in L1 when tokens are sent from L2 to L1, if there are not enough tokens in escrow. Here we propose adding that capability, but restricting the total minted amount to the maximum issuance that is possible in L2. This will be described in detail below but Figure 3 illustrates the flow in the case the BridgeEscrow balance is insufficient and the amount to mint is within the allowance.

![Figure 3: Minting GRT in L1GraphTokenGateway when BridgeEscrow balance is insufficient](../assets/gip-0037/Arbitrum%20GRT%20bridge%20-%20withdrawals%20with%20minting.png)

After deploying the Arbitrum bridge, it should be safe to deploy the L2 network using the existing version of the contracts, without necessarily updating the L1 side at the same time, or the new linear rewards mechanism, since rewards on L2 will be zero for some time. The deployment of the new rewards issuance formula can be done later, allowing us time to test each component separately.

# Detailed Specification

## L2 deployment like in GIP-0034

The proposal for the L2 deployment is similar to GIP-0034 in several ways, so please refer to this other proposal for the specification of the following:

 - [Payments and query fees](./0034-graph-arbitrum-deployment.md#payments-and-query-fees)
 - [Epoch management and block numbers](./0034-graph-arbitrum-deployment.md#epoch-management-and-block-numbers)
 - [Governance](./0034-graph-arbitrum-deployment.md#governance)

## Rewards calculation, issuance and distribution in detail

What follows is a description of the changes proposed to support L2. To understand the way this currently works in L1, please refer to the [background section in GIP-0034](./0034-graph-arbitrum-deployment.md#background-current-l1-only-design). We will also use the same notation as that GIP.

### Proposed calculation of rewards for L1 and L2

The main change in this proposal comes from changing the rewards per signal calculation from this current exponential form:

$$
\rho(t) = \rho(t_\sigma) + \frac{p(t_\sigma)r^{t-t_\sigma}-p(t_\sigma)}{\sum_{i\in{S}}{\sigma_i}(t)}
$$

Into this new linear formula for each layer $l$:

$$
\rho_l(t) = \rho_l(t_\sigma) + \frac{\kappa_l\cdot(t-t_\sigma)}{\sum_{i\in{S_l}}{\sigma_i}(t)}
$$

Where $\kappa_l$ is a constant (but configurable) "issuance per block" value in GRT that is different for L1 and L2.

This has several effects, namely:

 - It prevents issuance from growing exponentially with time,
 - It decouples issuance from token supply,
 - It drastically simplifies the issuance formula, and even if issuance per block changes, we get a piecewise-linear formula instead of a piecewise-exponential,
 - It probably reduces the gas needed to compute the rewards (this needs to be validated),
 - It makes it much easier to track, from L1, what the issuance in L2 should be, and will therefore allow us to mint in the bridge with a clear cap for the amount of tokens that can be minted. This can be done without any cross-layer messaging, which removes one of the main drivers for complexity in GIP-0034.

During the experimental phase, $\kappa_2$ would be set to zero, disabling rewards in L2 (note that $\rho_2(t_\sigma)$ starts at zero). We propose setting $\kappa_1$ to a value such that 300 million GRT are minted per year, so that it initially matches the current 3% annual rate. Using post-merge block times, this would mean an initial $\kappa_1$ value of `114.155251141552511` GRT per block.

### Rewards distribution

In this proposal, rewards would be minted using this new formula, but the distribution would stay the same: the RewardsManager on each layer is responsible for minting the new GRT whenever an allocation is closed, and sending them to the Staking contract.

## L2 mint allowance in L1GraphTokenGateway

Since tokens would now be minted in L2, we're breaking the previous assumption that every token in L2 is backed by a token in escrow in L1. This means that is possible that a user will bridge tokens from L2 to L1, and the balance in the BridgeEscrow will be insufficient to cover those tokens.

We need, therefore, to allow the L1GraphTokenGateway to mint new GRT. This is what GIP-0034 was designed to avoid, as a bridge vulnerability could allow an attacker to mint an arbitrary amount of GRT. To mitigate this risk, we propose adding a limit to how many GRT the gateway can mint, and this limit can track precisely the total amount of rewards that are accrued in L2.

We propose that the L1GraphTokenGateway be added as a minter in the GraphToken contract; but the gateway code would have an "L2 mint allowance" (here represented as $M(t)$ ) that also grows linearly:

$$
M(t) = M(t_{\kappa_2}) + \kappa_2 \cdot (t - t_{\kappa_2})
$$

This allowance will start at zero, and will stay at zero as long as the issuance per block in L2 is also zero. Whenever the L2 issuance per block is changed, the configuration of L1GraphTokenGateway must be updated _afterwards_ by governance, and using the L1 block number at which the L2 issuance per block was updated (i.e. $t_{\kappa_2}$).

When this happens, $M(t_{\kappa_2})$ (the total allowance at the last block where issuance changed) is updated using the old value for $\kappa_2$, thereby ensuring that the allowance always equals the total amount of rewards that could've been minted in L2, if all allocations had been closed correctly. As long as governance performs this update shortly after any update to $\kappa_2$, this should guarantee that the gateway's mint allowance is always sufficient, without any need for on-chain cross-layer messaging.

Whenever tokens are bridged from L2, and if the balance of the BridgeEscrow is insufficient to cover those tokens, the L1GraphTokenGateway will look at the allowance and the total amount of tokens minted since the contract was deployed. If the incoming transfer would take the historical total over the allowance (which should only happen if the Arbitrum bridge is compromised), the transaction will revert.

## New source of truth for GRT total supply

Up to this point, the single source of truth for the total supply
of GRT was the result of calling `GraphToken.totalSupply` in L1.

If this proposal is introduced, the new GRT total supply would be computed as:

$$
S_T = S_1 + S_2 - E
$$

Where:
- $S_T$ is the global total supply
- $S_1$ is the result of calling `GraphToken.totalSupply` in L1
- $S_2$ is the result of calling `GraphToken.totalSupply` in L2
- $E$ is the amount of tokens in escrow, i.e. the result of calling `GraphToken.balanceOf(BridgeEscrow.address)` in L1. (Note someone manually transferring GRT to the BridgeEscrow could skew this, so if it becomes an issue we can find a better way for accounting for the tokens bridged from L1 to L2)

## Spec for new L1GraphTokenGateway interfaces

### Public storage variables

This GIP proposes adding the following four variables to L1GraphTokenGateway storage:

- `totalMintedFromL2`: Will track the total amount of GRT minted by L1GraphTokenGateway.
- `accumulatedL2MintAllowanceSnapshot`: Will track the total amount of GRT that the gateway is allowed to mint, at the time of the last allowance configuration update. This corresponds to $M(t_{\kappa_2})$ as described above.
- `lastL2MintAllowanceUpdateBlock`: This is the block at which the allowance configuration was last updated, i.e. $t_{\kappa_2}$. This must match the last time issuance per block was updated in L2.
- `l2MintAllowancePerBlock`: The constant that specifies how many new GRT the gateway is allowed to mint per block; this must match the L2 `issuancePerBlock`, i.e. $\kappa_2$.

### Public functions

The following new functions are added to the L1GraphTokenGateway:

- `updateL2MintAllowance`: Callable only by governance. This is the function that must be called by governance whenever the issuance per block is updated in L2; governance will pass the new issuance per block and the block at which it was updated. The function automatically computes the accumulated allowance and updates the storage variables accordingly.
- `accumulatedL2MintAllowanceAtBlock`: Public view function; takes a block number $t$ and returns the accumulated mint allowance at that block: $M(t)$.
- `setL2MintAllowanceParametersManual`: Callable only by governance. This is only provided as a backup option, in case `updateL2MintAllowance` is ever called with the wrong values. It allows setting `accumulatedL2MintAllowanceSnapshot`, `lastL2MintAllowanceUpdateBlock` and `l2MintAllowancePerBlock` manually, without computing any values automatically.

## Spec for RewardsManager changes

### Public variables

- `issuancePerBlock`: New variable specifying the number of GRT for total rewards per block in the corresponding layer; i.e. $\kappa_l$.
- `issuanceRate` and `tokenSupplySnapshot` public variables are deprecated (renamed to `issuanceRateDeprecated` and `tokenSupplySnapshotDeprecated`, respectively).

### Public functions

- `setIssuancePerBlock`: Callable only by governance. This function sets the issuance per block for the network in which this RewardsManager is deployed, i.e. $\kappa_l$. A call of this function in L2 must be followed with a call of `L1GraphTokenGateway.updateL2MintAllowance` in L1.

# Backwards Compatibility

As mentioned  above, this proposal would introduce the following breaking changes in RewardsManager:
- `issuanceRate` and `tokenSupplySnapshot` public variables are deprecated and renamed.
- The `setIssuanceRate` function is removed.

The change from exponential rewards to linear can also be considered a breaking change (interfaces remain the same but the behavior is different).

# Dependencies

This proposal depends on GIP-0031, as we rely on the Arbitrum GRT bridge.

# Risks and Security Considerations

We've already identified some risks and set up a preliminary risk register in GIP-0031. Here we include the same risks with some updates, and a few additional ones. More may be identified through the audit and validation process.

| Risk | Impact | Likelihood | Severity | Mitigation             |
|------|--------|------------|-------------|------------------------|
| Arbitrum goes down / stops existing | Escrowed tokens are lost in the bridge contract | Low | Critical | Bridge contracts being pausable + upgradeable allow adding an arbitrary escape hatch, and escrow's approveAll allows adding a Merkle proof contract to reclaim funds based on an L2 snapshot. |
| Bridge or Arbitrum vulnerability allows someone to pipe out tokens from the bridge contract | Escrowed tokens are lost | Low | Critical | Bridge contracts pausable + upgradeable should allow us to stop an ongoing breach. Consider monitoring solutions to detect an ongoing breach! |
| Bridging tokens from a vesting contract  allows someone to escape a vesting lock | Vesting contracts are circumvented, tokens are transferred before they should | Low | Med | This would only be possible after adding the bridge to the authorized targets for a locked wallet, so only add this feature after deploying the corresponding wallet manager on L2 and validating that the locks are still effective. |
| A large amount of tokens are bridged to L2 and burned on L2 (instead of being withdrawn back to L1) | Unless we sync back the burning, total supply on L1 will not be representative, but this no longer affects inflation. | Med | Low | In this GIP we decouple inflation from burned supply, so this would no longer affect inflation. The total supply is no longer computed only from L1, so someone desiring a more precise computation can simply account for the burned tokens. |
| Bridge or Arbitrum vulnerability allows someone to mint tokens from the bridge contract | Attacker gets new tokens, inflation affects everyone | Low | Critical | Bridge contracts pausable + upgradeable should allow us to stop an ongoing breach. Consider monitoring solutions to detect an ongoing breach! The bridge's L2 mint allowance puts a cap on the potential impact, barring unknown bugs that could allow an attacker to work around it. |

# Validation

As with GIP-0031 and GIP-0034, the changes from this GIP will be audited and thoroughly tested in Goerli and Arbitrum Goerli. A detailed test plan will be developed and carried out. Detailed testing of the bridge and the L2 deployment itself have already been carried out to validate the two previous GIPs, so for this one we specifically need to test:
- Bridging tokens back to L1 where the bridging requires minting new tokens
- Attempting to mint tokens over the allowance (we can simulate this by updating the issuance in L2 without updating the allowance in L1)
- Updating from the current exponential issuance to linear, and ensuring open allocations can be closed.
- Opening and closing allocations with linear rewards, especially when issuance per block changes.

As with the other GIPs, a detailed deployment plan will be developed to ensure all the steps that have to be taken by governance are documented. Similarly, a contingency plan will be developed to account for potential risks and failure modes (that can be identified), and to clarify what the mitigation and recovery actions should be.

# Rationale and Alternatives

Several alternatives for different parts of this design were considered and discussed over the past months. What follows is our summary of the rationale for the design choices expressed in the specification above.

Please refer to [GIP-0034](./0034-graph-arbitrum-deployment.md) for the rationale behind the choice of L2, doing a full protocol deployment, governance and epoch management.

Here we'll focus on the main change proposed by this GIP:

## Exponential pre-minting on L1 vs. linear minting on L1+L2

In GIP-0034 we had presented arguments for pre-minting on L1 (with the drip function), as minting on L2 breaks the single source of truth for token supply and introduces additional risks in the bridge. These arguments are still valid, and minting only on L1 (or L2) would keep L1 (or L2) as a single source of truth and simplify the way we think about GRT supply.

The result from that line of thinking, however, is a mechanism that is rather complex: dripping rewards from L1 to L2 requires incentivizing keepers and validating that L1 and L2 gas prices are within certain bounds, and introduces new risks in the timing between sending messages from L1 and redeeming them on L2.

This makes us reconsider the alternative; if we can accept the fact that GRT now lives in more than one network and incorporate this when computing the total supply.

Furthermore, in GIP-0034 we had presented some potential state-syncing issues that still existed when trying to mint in L2 but keeping the exponential issuance formula. By introducing linear rewards in this GIP, those problems are no longer relevant, as issuance and token supply are now completely decoupled, and each layer can have an independent issuance speed. The protection in the bridge, by limiting the amount of minted tokens, acts as an additional risk mitigation and is made much easier with the linear formula. Rather than requiring cross-layer messaging to keep the L1 allowance and L2 issuance synced, using this linear formula makes it practically trivial to update the L1 side after updating the L2 side while ensuring there's no drift between the two layers.

# Improvements and future work

This proposal covers the initial deployment and the new rewards distribution mechanism for L2. There are improvements to these, and potential next steps in the protocol's move to L2, that are out of scope for this GIP but can be addressed in future proposals:
- **Schedule for L2 issuancePerBlock increments (and L1 decrements):** this GIP proposes setting the L2 `issuancePerBlock` to zero during the experimental phase, and gradually increasing it (while decreasing it in L1) to incentivize the move to L2, but doesn't specify the times at which this should happen, or what values should be used. This can be addressed with a separate GIP (or several).
- **State migration to L2:** As described in GIP-0034, helper functions can be provided to migrate subgraphs, stake, etc. to L2. using the bridge and callhook functions.
- **Changing to a burn-and-mint bridge**: Now that L1 is not the only source of truth, the escrow on the L1 side of the GRT bridge becomes less useful. It is still good to have it at this initial stage, while Arbitrum is still in beta, so that a catastrophic scenario is easier to recover from. In the future, however, if the community has more confidence in Arbitrum, and it is more fully decentralized, the bridge could be modified so that tokens in L1 are burned when sent to L2, and always minted when received back in L1. This would simplify the GRT total supply computation as it would now simply be total supply in L1 plus total supply in L2, but it takes away the current ability to have an allowance for the amount to mint.

# Copyright Waiver

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
