---
GIP: 0034
Title: The Graph Arbitrum deployment with a new rewards issuance and distribution mechanism
Authors: Pablo Carranza Vélez <pablo@edgeandnode.com>, Ariel Barmat <ariel@edgeandnode.com>
Created: 2022-06-01
Updated: 2022-09-29
Stage: Candidate
Discussions-To: https://forum.thegraph.com/t/gip-0034-the-graph-arbitrum-deployment-with-a-new-rewards-issuance-and-distribution-mechanism/3418
Category: "Protocol Logic", "Protocol Interfaces", "Economic Parameters"
Depends-On: GIP-0031
Implementations: Original implementations in https://github.com/graphprotocol/contracts/pull/571 and https://github.com/graphprotocol/contracts/pull/582/files - superseded by https://github.com/graphprotocol/contracts/pull/701
Audits: See https://github.com/graphprotocol/contracts/pull/710
---

# Abstract

This GIP introduces a new deployment of The Graph Protocol to the Arbitrum One Layer 2 blockchain. It proposes a plan to run the protocol on Ethereum mainnet (L1) together with the protocol on L2 instead of fully migrating in one go. The community would gradually move to L2 by starting with an experimental phase, in which indexing rewards are disabled, and then slowly increasing the amount of rewards in L2. To enable this, this GIP also proposes a change to how rewards are issued and distributed: instead of minting on the RewardsManager when allocations are closed, rewards would be pre-minted with a drip function in a new Reservoir contract. This function must be called at least once per week, and sends a configurable amount of the rewards to L2.

# Motivation

With rising gas costs, there is a growing interest in the community for a Layer 2 scaling solution. As mentioned in [GIP-0031](./0031-arbitrum-grt-bridge.md), there have been forums discussions highlighting this need, and conversations among core dev team members pointing towards optimistic rollups, and particularly Arbitrum, as a reasonable first step in this direction. This is a big change, however, and not risk-free. So it would make sense to approach this with caution, and gradually mitigate risk along the way. For this reason, we’d like to explore an Arbitrum protocol implementation that works initially with no indexing rewards and therefore useful for experimental subgraphs. This would be a first step towards running the protocol with rewards on Arbitrum, assuming the experimental phase is successful. It would be beneficial, however, for the mechanism for rewards distribution to be implemented from the beginning, so that governance can gradually increase rewards in Arbitrum as the community gains confidence in the L2 network.

# Prior Art

We've looked closely at the way Livepeer carried out their L2 migration, as described in [LIP-73](https://github.com/livepeer/LIPs/blob/master/LIPs/LIP-73.md). They went for a full migration, whereas we're proposing a more gradual approach where L1 and L2 coexist.

The rewards accrual and drip system proposed here is inspired by the way the Maker [Rates Module](https://docs.makerdao.com/smart-contract-modules/rates-module) works.

# High Level Description

We propose a gradual move to L2 by deploying the new L2 protocol in parallel to the existing L1 deployment. Initially, this new deployment would be experimental because indexing rewards would be disabled. Eventually, governance can enable rewards by increasing the configurable fraction of the total rewards that are distributed in L2. This new network would be deployed on Arbitrum One (after testing on Arbitrum Goerli).

At a high level, the L2 network will mostly work like the existing L1 network. Most of the contracts will be deployed unchanged, so staking and curation can be done in the same way as in L1. To participate in the L2 network, users can move GRT to L2 using the bridge proposed in [GIP-0031](./0031-arbitrum-grt-bridge.md), though future GIPs can propose ways to facilitate migration of staked tokens or subgraphs and curated signal.

The main change to support the L2 deployment will be a redesign of how indexer rewards are issued and distributed both in L1 and L2. Currently, rewards for subgraphs and allocations are computed and snapshotted when signal or allocations change, and then minted and immediately distributed to indexers and delegators when an allocation is closed. This is shown in Figure 1.

![Figure 1: Current implementation of rewards issuance and distribution in L1](../assets/gip-0034/L1%20rewards%20issuance%20and%20distribution.png)

Here we propose changing this to a periodic drip function that can be called by Indexers and potentially other addresses whitelisted by governance, and mints tokens to cover rewards for a period of time (one week) in the future. This drip function will exist on a new contract called a "Reservoir", particularly the "L1Reservoir", which will hold the tokens until the RewardsManager pulls them when an allocation is closed.

On L2, the L2Reservoir will not include a drip function. Instead, the L1Reservoir will send a configurable fraction of the rewards to L2 using the GRT bridge, together with calldata to call an `onTokenTransfer` function on the L2Reservoir, whenever the drip function is called on L1. This will update the variables so that accrued rewards can be computed correctly. This fraction (`l2RewardsFraction`) will be set to zero throughout the experimental phase, and can then be increased gradually by governance as the community gains confidence in the L2 solution.

Figure 2 shows the way the rewards issuance (now separated from distribution) would work in the L1+L2 scenario.

![Figure 2: Proposed rewards issuance mechanism for L1+L2](../assets/gip-0034/L1_L2%20rewards%20issuance.png)

The drip function can provide a keeper reward for whoever called it, to incentivize Indexers to call this periodically without incurring additional costs from gas.

Once rewards have been issued, and until the reservoir runs out of funds, indexers can close allocations as usual. The RewardsManager will query the reservoir on its corresponding layer to calculate the rewards owed for a particular allocation, and then pull the rewards from the reservoir so that the Staking contract can distribute them. This is illustrated in Figure 3.

![Figure 3: Proposed rewards distribution mechanism for L1+L2](../assets/gip-0034/L1_L2%20rewards%20distribution.png)

After deploying the Arbitrum bridge, it should be safe to deploy the L2 network including the L2Reservoir, without necessarily updating the L1 side at the same time, since rewards on L2 will be zero for some time. The L1Reservoir deployment and L1 RewardsManager update can happen at a later time, allowing us time to test each component separately.

# Detailed Specification

## Payments and query fees

We would deploy a new AllocationExchange with enough funds to pay for query fees that are served through L2 allocations. Indexers must know that a Voucher they receive is for an L2-allocation and they would claim it through the Arbitrum AllocationExchange.

## Epoch management and block numbers

The EpochManager contract would be deployed to L2 as-is, and L2 would follow its own epoch count. Off-chain components like the Block Oracle should follow the epoch and epoch start block reported by the EpochManager on each layer. This would use the `block.number` as seen by the contract, which follows the L1 block number with a precision around 10 minutes (or up to 24 hours in a Sequencer downtime scenario).

Epoch length would initially be the same as in L1, but it might change in the future.

The existence of two different block numbers (`block.number`, which follows L1, and the RPC block number, that reports a separate L2 block number) means that off-chain components should be careful to use the correct number for each purpose:
- To compute epochs, e.g. to decide when to close allocations, use the block number reported by EpochManager, or `block.number`.
- To compute the Arbitrum chain head when indexing Arbitrum, use the block number reported by the Block Oracle, that should follow the L2 block number reported by the RPC node.

## Governance

The multisig for the Graph Council has been deployed to Arbitrum at [this address](https://arbiscan.io/address/0x3e43EF77fAAd296F65eF172E8eF06F8231c9DeAd#code), and the multisig for the Arbitrator has been deployed to [this address](https://arbiscan.io/address/0x113DC95e796836b8F0Fa71eE7fB42f221740c3B0). Governance actions on L2 can be performed directly on L2 through these accounts.

## Rewards calculation, issuance and distribution in detail

What follows is a detailed description of how rewards calculation works at the time of writing this GIP, for background, and then a description of the changes proposed to support L2.

We will use the following notation:

$\rho$: rewards per signal

$R$: rewards

$p$: GRT total supply, or base amount for the inflation calculation

$\sigma$: signal, i.e. tokens on the Curation contract

$\omega$: allocated tokens, i.e. tokens from indexers' stake that are allocated to a subgraph

$\gamma$: rewards per allocated token

$r$: issuance rate (including the +1)

$S$: set of all subgraphs

$A$: set of all allocations

And we'll be using subscripts for subgraphs and superscripts for allocations (e.g. $R_i^k$ is "rewards on subgraph $i$ and allocation $k$"). When mentioning snapshots, since signal and allocations are discontinuous, we use the superscript minus sign (as in $\rho_i(t_{\sigma_i}^-)$) to denote a left-hand limit, i.e. snapshotted right before the signal or allocation changed.

Values are expressed as functions of time $t$ in blocks.

### Background: current L1-only design

Rewards are calculated and snapshotted in two dimensions:

- Rewards per signal, updated when signal for a subgraph changes, and
- Rewards per allocated token, updated when allocations for a subgraph change

Reward distribution follows the approach from [Batog et al](http://batog.info/papers/scalable-reward-distribution.pdf) but it is done twice: first, treating each subgraph as a staker for the total rewards (computing rewards per signal), and secondly, treating each allocation as a staker for the subgraph’s rewards (computing rewards per allocated token).

If $t_\sigma$ is the last time signal was updated on any subgraph, every time we want to calculate rewards, we should calculate the new value for accumulated rewards per signal as:

$$
\rho(t) = \rho(t_\sigma) + \frac{p(t_\sigma)r^{t-t_\sigma}-p(t_\sigma)}{\sum_{i\in{S}}{\sigma_i}(t)}
$$

And then we use this to compare the current value with a particular snapshot:

$$
\Delta\rho_i(t) = \rho(t) - \rho_i(t_{\sigma_i}^-)
$$

Where $\rho_i(t_{\sigma_i}^-)$ is the snapshot for the subgraph’s accumulated rewards per signal, last updated when signal for that subgraph changed ($t_{\sigma_i}$).

Then the new rewards for subgraph $i$, accumulated since $t_{\sigma_i}$, are:

$$
\Delta{R}_i(t; t_{\sigma_i}) = \Delta\rho_i(t)\sigma_i(t)
$$

And the accumulated rewards for subgraph $i$ (on the signal dimension) will be:

$$
R_i(t) = R_i(t_{\sigma_i}) + \Delta{R}_i(t; t_{\sigma_i})
$$

Now, when we want to compute the rewards for an allocation, we look at the new rewards since the last allocation change for the subgraph (which we’ll say happened at time $t_{\omega_i}$):

$$
\Delta{R}_i(t; t_{\omega_i}) = R_i(t)-R_i(t_{\omega_i}^-)
$$

Note we snapshotted $R_i(t_{\omega_i}^-)$ right before the last allocation change happened for the subgraph.

Then, we can compute the new rewards per allocated token:

$$
\Delta\gamma_i(t; t_{\omega_i}) = \frac{\Delta{R}_i(t; t_{\omega_i})}{\omega_i(t)}
$$

(This only works because $\omega_i(t) = \omega_i(t_{w_i})$, given that $t_{w_i}$ is the last time the allocations for $i$ changed)

Which gives us the final value of rewards per allocated token on this subgraph:

$$
\gamma_i(t) = \gamma_i(t_{\omega_i}) + \Delta\gamma_i(t; t_{\omega_i})
$$

Now, when we take the rewards for a particular allocation $k \in A_i$, that was created at time $t^k$, we can compute the rewards like this:

$$
R_i^k(t)=({\gamma_i(t)-\gamma_i(t^{k-})}){\omega_i^k}
$$

Note that $\gamma_i(t^{k-})$ had to be computed and snapshotted when the allocation was created (with the snapshot computed before the allocation is added to the pool).

When an allocation is closed, these rewards are minted and sent to the indexer (and delegators, if any).

All of these calculations are currently done inside the RewardsManager contract, especially in the `onSubgraphSignalUpdate` and `onSubgraphAllocationUpdate` functions that are triggered when curation signal or allocations change.

### Proposed calculation of rewards for L2

Once we move to L2, since we want to avoid minting on L2 (see [the related section below](#pre-minting-on-l1-vs-minting-on-l1l2) for the rationale), we should decouple the rewards issuance and distribution, so we should mint on L1, send a fraction of the rewards to a Reservoir on each layer, then let RewardsManager use those already-minted tokens as needed.

So let’s define $\lambda(t)$ as the proportion of total rewards that is sent to L2 at time $t$. In the contract code, this will be stored in the variable `l2RewardsFraction`.

Let's also define $t_0$ as the last time the rewards drip function was called (expected to happen at least once per week). This drip function will be described [below](#proposed-drip-function).

$\lambda$ is set by governance to a value between 0 and 1, and should be changed over time to incentivize the move to L2.

We can then define a global rewards function $R(t)$, and total rewards functions for L1 and L2, $R1(t)$ and $R2(t)$ respectively:

$$
\Delta R(t, t_0) = p(t_0) r^{t-t_0} - p(t_0)
$$

$$
R(t) = R(t_0) +\Delta R(t, t_0)
$$

$$
R1(t) = R1(t_0) + (1-\lambda(t_0))\Delta R(t, t_0)
$$

$$
R2(t) = R2(t_0) + \lambda(t_0) \Delta R(t, t_0)
$$

Note $R(t) = R1(t) + R2(t)$.

Besides this, we propose redefining the meaning of $p(t)$. Now that we have an L2 deployment, it would be quite complex to sync the number of GRT burned in L2 back to L1. Moreover, now that we mint tokens in advance, the total supply at a certain time will represent the desired supply at a *future* time, because we’ve minted tokens that are to be distributed eventually, using a drip function as described below.

So our proposal is to define $p(t)$ as: “the total supply of GRT produced by accumulating rewards up to time $t$”. So this definition would exclude burnt tokens and any tokens that were minted to cover future rewards; we can therefore compute it as follows:

$$
p(t_0') = p(t_0) + \Delta R(t_0', t_0)
$$

Where we’ve initialized with the real GRT total supply at the time the Reservoir contract (described below) was deployed, and then we snapshot it at every new $t_0'$ by adding the computed value of accumulated rewards.
This makes the issuance always follow the same exponential, so while issuance rate stays constant we could actually skip the snapshot and always use the same $t_0$. It is still convenient to do the snapshots on every drip, however, to prevent numerical errors when computing the exponential in fixed-point notation.

### Proposed drip function

As we mentioned above, the new L1Reservoir contract will have a a `drip()` function. This function works as follows:

- The last time the function ran, it minted tokens for all rewards up to time $t_1$. This was saved in contract storage, together with $t_0, p(t_0), R1(t_0)$. (So during initialization these variables will have to be set to their appropriate values when an initialization function was called, and the appropriate amount of GRT will have to be minted as well).
- We compute an error value $\epsilon = \Delta R(t_1, t_0) - \Delta R(t, t_0)$ using the stored $t_0, t_1, p(t_0)$. This is the number of tokens for *future* rewards that have already been minted (if $t < t_1$, i.e. drip was called early) or the rewards for the interval $[t1,t]$ that should’ve been minted, but weren’t (if $t \geq t_1$, i.e. drip was called late). Note $\epsilon$ can be negative in this latter case.
- Set and store $t_0 = t$, $t_1 = t_0 + (7 \times 7200)$. This new $t_1$ is calculated to produce rewards for the next week (counting 7200 blocks per day as it will be post-Merge).
- Compute $n = \Delta R(t_1, t_0)$ (using the updated $t_0$, $p(t_0)$ and potentially updated $r$). This is the total rewards to be distributed from now up to the new $t_1$.
- Compute $N = n - \epsilon$. This is the amount of tokens that need to be minted to cover rewards up to $t_1$.
- Mint $N$ tokens.
- Store $p(t_0), R1(t_0)$.
- Send $N \lambda(t_{0})$ tokens to the L2Reservoir contract using the bridge, and also notify the L2Reservoir of the values $q(t_0) = p(t_0) \lambda(t_0)$ and $r$. Note: if $\lambda$ is updated, this needs to change a bit, to compute the correct difference based on the previous $t_1$ like we did to compute $N$ above. See below for details.

This drip would then be such that, if all rewards at any particular time up to $t_1$ are computed based on the stored values for $p, R1, R2, \lambda$, the tokens available on the Reservoir on each layer should be sufficient to cover these rewards.

Calls to take rewards after $t_1$ may revert if there aren’t enough funds, in which case a call to `drip()` is needed before closing the allocation.

When sending the tokens to L2. the L1Reservoir uses the L1GraphTokenGateway described in GIP-0031, and includes the necessary calldata so that the gateway on L2 calls `L2Reservoir.onTokenTransfer` to update the necessary variables.

One more thing to note is updating $\lambda$ will only take effect the next time `drip` is called. Updating $r$ (issuance rate) requires snapshotting rewards per signal, and it will also now require a call to `drip` as well for it to take effect.

As mentioned before, when the value for $\lambda$ changes between two drips, the behavior of `L1Reservoir.drip` must be slightly modified to correctly compute the amount that should be sent to L2. Suppose the last time we called the drip, the value was $\lambda_{old}$ and it’s now $\lambda_{new}$. Recall the value for $\epsilon$ and $n$ that we computed (where $\epsilon$ can be positive or negative depending on whether $t<t_1$) then the value to send to L2 (let’s call it $N_{L2}$) is:

$$
N_{L2} = \lambda_{new}n - \lambda_{old}\epsilon
$$

This corrects for the tokens that have already been sent to L2 with the previous $\lambda$, that should be effective until the current block. In the edge case where the new $\lambda$ is lower than the old value, it's possible that this subtraction produces an underflow and reverts; in this case, we would need to wait for some blocks (in the worst case, until $t_1$), so that $\epsilon$ becomes small enough and the call can succeed.

It’s worth noting that the block number for $t_0$ stored on L2 might differ from the value on L1 because of the drift between block numbers on each layer, or because the retryable ticket wasn't redeemed immediately. We can mitigate this in several ways, but we propose simply storing $t_{0_{L2}} = block.number$ when the message is received in L2.

This means the amount for L2 rewards might not *exactly* match what’s needed to cover 1 full week if the time difference between the chains changes throughout the week. This means the drip function might have to be called slightly before the 7-day mark if funds on L2 are running low.

Additionally, we have to consider the risk that the drip transaction will fail on L2 for lack of gas, in which case it could be retried later, so we could have messages received out of order. To mitigate this, we propose including an incrementing nonce in the message sent from L1, and checking on L2 that the nonce has the expected value. A governance function can allow setting an arbitrary expected nonce on L2 to mitigate the edge case where a retryable ticket expires and is never received. Retryable tickets staying unredeemed for more than a few blocks would be undesirable, however, so it would be good for several actors to set monitoring tasks that redeem any drip tickets that have failed for insufficient L2 gas. A keeper reward described [below](#keeper-reward-for-calling-the-drip-function), and the economic security from Indexer staking, should incentivize callers of `drip` to redeem the tickets immediately.

### Updated rewards distribution

By defining our total rewards function like we did above, most of the rewards calculation and distribution can stay the same. The only thing that changes is how we calculate $\rho$ (rewards per signal) at any point in time to distribute the rewards or when we compute a snapshot. Rather than computing it directly, we compute accumulated rewards for layer $l$ at time $t$ with the formula defined above:

$$
Rl(t) = Rl(t_0) + \Delta Rl(t, t_0)
$$

i.e.

$$
R1(t) = R1(t_0) + \Delta R1(t, t_0)
$$

$$
R2(t) = R2(t_0) + \Delta R2(t, t_0)
$$

Where $\Delta Rl$ is $\Delta R1$ on Layer 1 and $\Delta R2$ on L2:

$$
\Delta R(t, t_0) = p(t_0) r^{t-t_0} - p(t_0)
$$

$$
\Delta R1(t, t_0) = (1-\lambda(t_0))\Delta R(t, t_0)
$$

$$
\Delta R2(t, t_0) = \lambda(t_0)\Delta R(t, t_0)
$$

And $Rl(t_0)$ is the $R1(t_0)$ or $R2(t_0)$ stored in the Reservoir for each layer. Note that to compute $R2(t)$ in L2, for gas efficiency we can keep $q(t_0) = \lambda(t_0)p(t_0)$ as a single storage variable and compute:

$$
\Delta R2(t, t_0) = q(t_0)r^{t-t_0}-q(t_0)
$$

Now on to how we compute the rewards for each subgraph and allocation:

We want to ensure that, for each layer $l$, and for every time interval $[t_1, t_2]$ **in which signal is constant,** the rewards accumulated by each subgraph are the fair share of the total rewards:

$$
{\frac{\sigma_i}{\sigma_{T_{Ll}}}(Rl(t_2)-Rl(t_1))} = {R_i(t_2)-R_i(t_1)}
$$

Where $\sigma_{T_{Ll}}$ is the total signal on layer $l$, so $\sum_{i \in S_{Ll}}{\frac{\sigma_i}{\sigma_{T_{Ll}}}} = 1$.

So the rewards per signal for this interval, where $\sigma_{T_{Ll}}$ is constant, is:

$$
\Delta\rho{l}(t_2,t_1) = {\frac{Rl(t_2)-Rl(t_1)}{\sigma_{T_{Ll}}}}
$$

Since this is only valid while signal is constant, we need to compute and keep the accumulated rewards per signal snapshotted whenever signal changes (like we currently do at each $t_\sigma$), and then we can compute the new value for it:

$$
\rho1(t) = \rho1(t_\sigma) + \frac{R1(t)-R1(t_\sigma)}{\sigma_{T_{L1}}(t)}
$$

$$
\rho2(t) = \rho2(t_\sigma) + \frac{R2(t)-R2(t_\sigma)}{\sigma_{T_{L2}}(t)}
$$

Note this also requires us to snapshot the total accumulated rewards on each layer ($Rl(t_\sigma)$) whenever signal changes on that layer, and it makes the rewards calculation a bit more expensive.

For completeness, the expanded formula for Layer 1 is:

$$
\rho1(t) = \rho1(t_\sigma) + \frac{R1(t_0)+(1-\lambda)(p(t_0)r^{t-t_0}-p(t_0))-R1(t_\sigma)}{\sigma_{T_{L1}}(t)}
$$

And for Layer 2:

$$
\rho2(t) = \rho2(t_\sigma) + \frac{R2(t_0)+\lambda(p(t_0)r^{t-t_0}-p(t_0))-R2(t_\sigma)}{\sigma_{T_{L2}}(t)}
$$

Now that we can compute rewards per signal at any given time, we can compute the delta for a subgraph $i$ in layer $l$:

$$
\Delta\rho_i(t) = \rho{l}(t) - \rho_i(t_{\sigma_i}^-)
$$

We simply have to use the correct formula for $\rho$ on each layer, and as we currently do, compute and snapshot the value when the signal for a subgraph has changed.

After that, the algorithm for rewards distribution is identical to its current form as described above.

Considering how these values depend on information stored on the Reservoir, we suggest exposing the function for $Rl(t)$ on the Reservoir (and L2Reservoir) and calling it from RewardsManager as needed.

### Burning denied/unclaimed rewards

Since the drip function will now mint all the potential rewards for an upcoming week, it's possible that some of these will not be claimed or distributed, in which case they could accumulate in the Reservoir. To mitigate this, we propose burning the accumulated rewards for an allocation in three scenarios where we currently don't mint them:
- When an allocation is closed with a zero POI
- When an allocation is closed late (after `maxAllocationEpochs`) by someone who is not the allocation's indexer.
- When the Subgraph Availability Oracle denies rewards for a subgraph

There is a fourth scenario where rewards may accumulate: when a subgraph is under the minimum signal threshold. In this case, we do not compute the rewards as accrued for the subgraph while the condition holds, and start accumulating afterwards. To keep this behavior, we propose not doing anything special here, so a small amount of GRT might still accumulate in the Reservoir. Future GIPs may find ways address this if it becomes a significant amount at any point.

### Keeper reward for calling the drip function

Any indexer wishing to close an allocation has an incentive to call the drip function so that rewards are available to be distributed. For this reason, we propose adding this drip call as an optional feature in the Indexer Agent. This call will, however, incur a potentially high gas fee, as it will include some computation on L1 plus a retryable ticket for L2.

Therefore, it would make sense to reward the caller of this function (the "keeper") so that they can offset the cost of gas. We propose using additional GRT issuance to cover this reward, minted by the L1Reservoir but delivered to the keeper on L2 by the L2Reservoir, so that the reward is only given if the caller uses the correct parameters and redeems the retryable ticket in L2. Moreover, a fraction of the reward would be sent to whoever redeems the ticket in L2, to incentivize the use of correct L2 gas parameters.

The proposed formula for the keeper reward $K$ at block $t$ is:

$$
K(t) = \kappa (t-(t_0 + t_{Kmin}))
$$

Where $t_0$ is the last time `drip` was called, $t_{Kmin}$ is a minimum interval for calling `drip` set by governance (`minDripInterval` in the L1Reservoir), and $\kappa$ is a constant set by governance (`dripRewardPerBlock` in the L1Reservoir). The value of $\kappa$ can be set to ensure that, as long as the price of gas in GRT stays within a certain range, it is always profitable to call this function before the week-long drip interval is over. This should be negligible when compared to GRT issuance from indexer rewards (or if it's not, it's probably a sign that L1 gas is high enough that it's time to move the issuance to L2).

The Indexer Agent can therefore check whether a call to `drip` would be profitable, and do it if that is the case. This call is vulnerable to MEV/frontrunning if sent in the open, so it would be preferrable to do it through a private auction channel like Flashbots, so as to not consume gas if someone else will run it before in the same block.

Calling `drip` should be done with the correct parameters to ensure auto-redeeming of the retryable ticket, but since this is hard to guarantee, Indexers should try to redeem the ticket immediately if the auto-redeem fails. It should be a slashable offense to repeatedly and purposefully create tickets with incorrect parameters and not attempt to redeem them. It should also be a slashable offense to cancel a retryable ticket, since this will affect the whole network and require a corrective action from governance (i.e. manually adding funds to the L2Reservoir, and fixing the expected nonce).

## L1Reservoir specification

The L1Reservoir should expose the following external functions:

- `drip`: This function follows the behavior described above ([Proposed drip function](#proposed-drip-function)). It takes arguments to specify the max submission cost, gas price and max gas for the L2 retryable ticket. These must be set to 0 when the `l2RewardsFraction` is set to zero, as the L2 ticket is unnecessary in that case. The `msg.value` must be equal to `maxSubmissionCost + (gasPrice * maxGas)` to cover the L2 ticket costs. The function also takes a beneficiary address in L2 to whom the L2Reservoir should send the reward. The function is only callable by addresses whitelisted by governance, or by Indexers. An alternative form of the function allows an Operator for an Indexer to also call this function, but specifying the corresponding Indexer's address.
- `initialSnapshot`: This function can only be called by governance, and is meant to be called only once. It snapshots the GRT total supply to store it as the initial value for $p(t)$, and marks the current block as the first $t_0$. Rewards will start accruing after this is called (if issuance rate is nonzero). This function takes a parameter for an amount of GRT to mint to cover any pending rewards for open allocations.
- `getNewGlobalRewards`: Computes the new rewards accrued on both layers at a particular block, since the last time the drip function was called. This is the same as $\Delta R(t)$.
- `getNewRewards`: Computes the new rewards accrued on this layer (L1) at a particular block, since the last time the drip function was called. This is the same as $\Delta R1(t)$.
- `getAccumulatedRewards`: Computes the total rewards accrued on this layer (L1) at a particular block, since the deployment of this contract. This is the same as $R1(t)$.
- `approveRewardsManager`: Called only by governance, it provides the RewardsManager with approval to manage this contract's GRT funds.

The L1Reservoir should also expose setter functions, callable only by governance, to update the configuration variables in storage (issuance rate, L2 rewards fraction, etc).

## L2Reservoir specification

The L2Reservoir should expose the following external functions:

- `onTokenTransfer`: This function is called by the L2GraphTokenGateway when the message and tokens from the L1Reservoir are received. It validates that the sender is the L1Reservoir, then ABI-decodes the additional data to extract the issuance base, issuance rate, nonce, keeper reward, and address of the keeper to send the reward. It must validate that the nonce matches the expected value. It will receive values for the issuance base $q(t)$ and the issuance rate $r$, and store them, snapshotting the accumulated rewards. It will mark this block as the latest $t_0$ and if needed, will trigger a snapshot of rewards per signal on the RewardsManager. Finally, it transfers the GRT for the keeper reward to the address specified in the message. Using the `ArbRetryableTx.getCurrentRedeemer` interface, this function will check if there is a redeemer for the retryable ticket, in which case part of the keeper reward (using a fraction configurable by governance) will be sent to the redeemer's address.
- `getNewRewards`: Computes the new rewards accrued on this layer (L2) at a particular block, since the last time the drip function was called. This is the same as $\Delta R2(t)$.
- `getAccumulatedRewards`: Computes the total rewards accrued on this layer (L2) at a particular block, since the deployment of this contract. This is the same as $R2(t)$.
- `approveRewardsManager`: Called only by governance, it provides the RewardsManager with approval to manage this contract's GRT funds.
- `setNextDripNonce`: Called only by governance, it provides a fallback in case a retryable ticket is lost, so that the next drip can be accepted if it has the specified nonce value.

# Backwards Compatibility

The Pull Request implementing the new rewards issuance mechanism, and introducing the Reservoir (PR 571), includes the following breaking changes:
- The `setIssuanceRate` function has been removed from RewardsManager, and the `issuanceRate` public storage variable has been  deprecated there (it is now set in the Reservoir)
- The `RewardsDenied` event now includes a token amount (this wasn't calculated in the past when rewards were denied, but now that we have this value we might as well make it available).

# Dependencies

This proposal depends on GIP-0031, as we rely on the Arbitrum GRT bridge and its callhook mechanism to implement the L1-L2 communication.

It also requires Arbitrum Nitro to be deployed to mainnet / Arbitrum One before we roll out the drip function on L1, as the current version might cause the drip to L2 to fail on L2 while the transaction on L1 succeeded (if the retryable ticket submission cost is too low), which could cause the layers to fall out of sync. Nitro will fix this by making the transaction fail in L1 in this scenario.

Additionally, an upcoming Arbitrum release will add the `ArbRetryableTx.getCurrentRedeemer` interface, so the keeper rewards PR should be deployed after this is live.

Considering these dependencies, we propose the following order of deployment, both for testnet and mainnet:

1) Deploy the bridge from GIP-0031 to L1 and L2
2) Deploy the L2 side of the changes from this GIP. Experimental phase begins!
3) Wait for Arbitrum Nitro and the getCurrentRedeemer interface to be deployed
4) Deploy the L1 side of the changes from this GIP (and any necessary L2 updates/fixes developed in the meantime)
5) Eventually, governance enables rewards in L2 by setting `l2RewardsFraction` to a nonzero value.

Note that there are two implementation PRs linked to this GIP:
- [#571](https://github.com/graphprotocol/contracts/pull/571), which implements the new rewards issuance using the Reservoir, and
- [#582](https://github.com/graphprotocol/contracts/pull/582/files), which adds the keeper reward for calling drip.

Both PRs have been audited, and the implementation is now combined into a separate rebased [PR #701](https://github.com/graphprotocol/contracts/pull/701).

# Risks and Security Considerations

We've already identified some risks and set up a preliminary risk register in GIP-0031. Here we include the same risks with some updates, and a few additional ones. More may be identified through the audit and validation process.

| Risk | Impact | Likelihood | Severity | Mitigation             |
|------|--------|------------|-------------|------------------------|
| Arbitrum goes down / stops existing | Escrowed tokens are lost in the bridge contract | Low | Critical | Bridge contracts being pausable + upgradeable allow adding an arbitrary escape hatch, and escrow's approveAll allows adding a Merkle proof contract to reclaim funds based on an L2 snapshot. |
| Bridge or Arbitrum vulnerability allows someone to pipe out tokens from the bridge contract | Escrowed tokens are lost | Low | Critical | Bridge contracts pausable + upgradeable should allow us to stop an ongoing breach. Consider monitoring solutions to detect an ongoing breach! |
| Bridging tokens from a vesting contract  allows someone to escape a vesting lock | Vesting contracts are circumvented, tokens are transferred before they should | Low | Med | This would only be possible after adding the bridge to the authorized targets for a locked wallet, so only add this feature after deploying the corresponding wallet manager on L2 and validating that the locks are still effective. |
| A large amount of tokens are bridged to L2 and burned on L2 (instead of being withdrawn back to L1) | Unless we sync back the burning, total supply on L1 will not be representative, and this also affects token inflation. Exactly what the consequences of this would be is not immediately clear. | Med | Low? | Limit the total amount of GRT bridged? Account for bridged GRT as “burned” when considering total supply? Something else? Eventually syncing back the state should allow reconciling both layers, but this is not trivial. In this GIP we decouple inflation from burned supply, so this would no longer affect inflation. |
| A retryable ticket for `drip` expires before being redeemed | L1 and L2 fall out of sync, so rewards can't be sent to L2 anymore | Low | Med | Governance can mitigate by setting an updated nonce and transferring tokens to cover rewards. Rewarding the redeemer adds an economic incentive to make this less likely. |
| A malicious / lazy keeper sets low L2 gas values for `drip`, and never redeems the ticket | Until someone else redeems the ticket, rewards aren't received in L2. If the delay is too long, L1 and L2 can fall out of sync and L2Reservoir can run out of funds | Low | Low | Other actors (indexers, delegators, core devs?, the Foundation?) can set up recurring tasks (e.g. using Defender) to redeem L2 tickets, which should be much cheaper than calling `drip` in L1. Rewarding the redeemer adds an economic incentive to make this less likely. |


# Validation

PR 571 and PR 582, implementing most of this GIP, have already been audited. Additional changes added after the audit ([PR #705](https://github.com/graphprotocol/contracts/pull/705)) would also have to be audited before deployment.

It will be important to perform a detailed test of the network in a testnet deployment to Goerli and its corresponding Arbitrum Nitro L2. Continuing from the rough test plan presented in GIP-0031, we should also test (at least) the following:

- Deploying and initializing all the L2 contracts.
- Operating the network (staking, allocating, curating) without deploying any changes to L1.
- Updating the L1 side to include the changes from this GIP, and updating the L2 side accordingly.
- Calling the drip function through Flashbots. This should be tested repeatedly, with allocations being opened and closed, and double checking that the rewards amounts are as expected and that the Reservoir will not run out of funds on either layer.
- Increasing and decreasing the `l2RewardsFraction` and `issuanceRate` and calling drip again.
- Pausing and unpausing the L2 protocol.

This should be incorporated into a formal test plan that we will carry out before deploying the network to mainnet Arbitrum.

If possible, we should also test and practice catastrophic scenario recovery processes, for instance, producing a Merkle trie snapshot of the L2 network at a specific block and using that to recover funds from escrow in L1.

# Rationale and Alternatives

Several alternatives for different parts of this design were considered and discussed over the past months. What follows is our summary of the rationale for the design choices expressed in the specification above.

## Choice of L2

This is probably the most central decision behind this GIP. We've already described some of the reasons for choosing Arbitrum in GIP-0031, and they're worth repeating here, as they still apply:

- Arbitrum is the most popular L2 chain (by TVL).
- It’s EVM-compatible (or, more precisely, provides an EVM-compatible abstraction on top of the AVM), unlike existing ZK rollups.
- It has a working deployment on mainnet (albeit in beta).
- It has a credible, concrete path towards proper decentralization. There’s one key component -the Sequencer- that looks like it’ll be harder to decentralize, but they’re still ahead of Optimism, whose fraud proof mechanism is yet to be deployed (even though it sounds like it’ll be as good as Arbitrum’s or better).

The consensus we've gathered from discussions among core devs was that ZK rollups are very attractive but not at a level of maturity that would make us confident in moving the protocol there at this time. The other main potential option would be Optimism, and there are good arguments in that their recent governance token launch is a great move towards decentralization, and their simpler EVM architecture could prove more reliable in the long run (though it's hard to tell at this point). Their lack of a fraud proof system in production made us lean towards Arbitrum in the end, and the upcoming updates with Nitro, that would give the community greater gas savings, also make this the more attractive choice at this moment.

As mentioned in GIP-0031, however, choosing an L2 at this point does not mean this is a _definitive_ move. The experimental phase is meant to give the community time to validate that this is the right decision right now, and even though a future move to another L2 would not be simple, there is a lot of learning to be done while rolling out this GIP that would be useful if this decision becomes necessary or more convenient in the future.

As part of rolling out this GIP, we are also working on designing and practicing fallback actions so that any catastrophic scenarios with Arbitrum can be mitigated, either by temporarily moving back to L1, or launching a separate Arbitrum instance and moving the protocol there.

## Full vs. partial deployment

One thing to consider is how we perform the initial deployment of the network.

A way to approach it would be to perform a full deployment of all the existing contracts in Mainnet and have rewards disabled, as proposed in this GIP. However, knowing that we are not going to support indexing rewards in the experimental phase we could initially do a partial deployment (i.e. only deploy some of the contracts).

**Benefits of a partial deployment:**

- We don't need to update the code to disable rewards or pause individual contracts.
- We can run a version of the network while we work on improved features both on Curation and GNS.
- Cheaper deployment by not paying for contracts we will not use.

**Requires:**

- Updating the Staking contract to test if the *rewards system* is connected and not updating snapshots if that is the case.
- Allowing for hot-plugging the rewards system into the Staking contract.
- All outstanding Staking improvements should be merged before the deployment, this includes Stale Allocations, Altruistic Allocations and Subsidized Query Settlement to reduce the number of upgrades.

**Downsides:**

- The network will be less representative of the final result
- Since we remove parts of the protocol, the off-chain components would have to mock the missing pieces
- The partial deployment requires more development work than a full deployment, to modify all the contracts to remove the missing pieces

On the other hand, doing a full deployment involves the following tradeoffs:

**Benefits of a full deployment:**

- We only need to do minimal changes to the contracts, to modify the rewards distribution and (for now) disable the rewards.
- The resulting network is fully representative of the future mainnet
- Off-chain components can interact with the L2 contracts in the same way as for L1

**Requires:**

- Disabling indexing rewards, which can be achieved by either a) setting issuance rate to zero, or b) having the block oracle disable rewards for any subgraphs that are on L2., or c) implementing the new L1+L2 rewards distribution (as described in this GIP), but setting the fraction of rewards sent to L2 to 0.

**Downsides:**

- Any state that is created during the experimental phase might make the migration to mainnet harder. We can mitigate this by warning users that a future migration might require moving the funds to the new deployment, or that we might “wipe” the state after the experimental phase.

Considering all these tradeoffs, we propose doing a full deployment, as described in this GIP, and communicating clearly with the community that any state in the Staking and Curation contracts could be cleared/reset at the end of the experimental phase (while allowing users to recover their staked funds, of course).

### Pre-minting on L1 vs. minting on L1+L2

One of the early decisions we discussed was whether to mint only on L1 vs. to mint rewards also on L2. Minting on L2 would allow us to mint on demand when allocations are closed, like we currently do, so it removes the need for a drip function. Minting only on L1 also has the downside that we need to pre-mint with some buffer (e.g. one week) so that the RewardsManager on both layers has funds when allocations are closed.

However, our discussions (between authors of this GIP) leaned strongly towards pre-minting on L1 because we believe it’s cleaner and safer to have a **single source of truth for token issuance and supply**. Minting on L2 independently from L1 means we need to look at both layers to figure out the total supply of GRT. It also means we need to be extra careful when bridging back to L1, as it’s possible that the bridge doesn’t have enough funds in escrow to cover the tokens minted on L2. In this scenario, we could allow the bridge (or RewardsManager, called by the bridge - but this is equivalent from a security point of view) to mint an additional amount of tokens up to a certain amount. This amount could be determined by the rewards issuance formula. This makes the bridge more complex and intuitively riskier, as we break the assumption that we had up to now that every token in L2 has an L1 counterpart. A certain amount of the expected L2 rewards will never be minted on L2 (those denied by the subgraph oracle, or for allocations closed with a zero PoI), so the bridge will potentially be able to mint those on L1 (unless we sync back state from L2, which we’re strongly trying to avoid/minimize). So overall, our main reason for choosing the drip from L1, knowing that it adds complexity, is to keep the bridge itself simpler, so it never ever mints tokens (or triggers some other contract minting) in L1.

There are other (secondary) factors for which minting in L2 would also prove challenging. Some state syncing would be needed for the calculation of rewards per signal:

$$
\rho(t) = \rho(t_{\sigma}) + \frac{p(t_\sigma)r^{t-t_\sigma}-p(t_\sigma)}{\sum_{i\in{S}}{\sigma_i}}
$$

Namely: how do you compute $p(t_\sigma)$? There are a few options I can think of, all with their downsides:

a) Sync total supply from L1. This means we still need to send a periodic message from L1, and snapshot total rewards and and rewards per signal whenever this message is received.

And now we have a problem: we snapshot $p(t_\sigma)$ when any subgraph’s signal changes, but now we won’t be able to do this because we can’t request this value at arbitrary times - only when the message is received from L1. This requires modifying the rewards formula to one that is equivalent to the pre-minting case (but losing the single source of truth for token supply).

b) Use only total supply on L2 to compute the L2 rewards. But this means rewards on L2 will be much smaller, and completely decoupled from L1. But then, (and this also applies to option a) on L1, how does the bridge/RewardsManager know the supply on L2? unless we sync back state from L2, we can’t compute the expected amount of rewards from L2 that can be bridged back to L1, so we have to allow the bridge to mint unlimited tokens.

c) Initialize $p(0)$ with a value (a snapshot of the total supply), and then accumulate it using the inflation formula. This fully decouples L1 and L2 issuance, and is similar to what we ended up proposing in this GIP, but again, it opens up the possibility of the two layers falling out of sync over time (especially if issuance rate changes). We'd need to make sure the value on both layers stays perfectly in sync, or we risk opening the possibility for the bridge to mint more tokens on L1 than expected.

Note that in a potential future when we only have the network on an L2, it should be feasible to move the minting of the token to L2, disable the minting on L1, move the escrow to L2, and therefore change the source of truth to L2 - but preserving the fact that it’s a _single_ source of truth.

## Fixed vs. dynamic l2RewardsFraction

We discussed whether to use a fixed value for $\lambda$ / `l2RewardsFraction` that is set by governance at specific points in time, or to have a way to dynamically compute this. The most natural solution would have been to compute it based on the total signal on each layer. However, the complexity of implementing the synchronization of total signal between layers makes this riskier and therefore less attractive. Moreover, if the experimental phase works out well, the most likely outcome is that $\lambda$ will monotonically increase from 0 to 1, so the dynamic mechanism would become irrelevant relatively soon. This is why we went with the proposal of simply using a variable set by governance.

## Governance

There are two main alternatives to governing the network parameters as well as any contract upgrades:

1. **Governance at a distance:** Allow the Council to send transactions from Mainnet through the bridge by providing special functions to pass messages to the L2 deployment.
2. **Chain-local Governance:** Deploy a Gnosis Safe multisig in Arbitrum and add the same Council owners.

We propose using a **local Arbitrum Governance Deployment** to keep the bridge implementation simple. Additionally, both Gnosis Safe and Defender are now supported on Arbitrum which will make the operation easier, no need to craft custom transactions to operate through a bridge. The idea is to try to keep both L1 and L2 deployments on feature parity but we might see them diverging in the future when some features will only be feasible on a lower gas environment, so we can treat them as two different environments with mirrored governance.

## Epoch Management

The EpochManager contract sets the pace of the protocol by affecting the lifecycle of allocations. This means it can control when allocations should be closed, query fees collected and rebates claimed. 

We could try to have a *global* clock both for L1 and L2 but we find that is not advisable as it would probably require moving epochs forward manually from L1 and sync them on L2. In addition to that, we might want to set a different pace for the protocol on L2. As a consequence, it makes sense to have a standalone EpochManager deployed that governance can tune separately.

Based on Arbitrum documentation and discussions with the Arbitrum team, it appears that the `block.number` and `block.timestamp` available on L2 should match the values from L1 to around 10 minutes precision, as this is updated by the Sequencer periodically. However, Sequencer downtime could cause these values to drift away, up to a maximum of 24 hours (as per current Arbitrum SequencerInbox configuration on mainnet, see `maxDelayBlocks` and `maxDelaySeconds` in [this contract](https://etherscan.io/address/0x4c6f947ae67f572afa4ae0730947de7c874f95ef#readProxyContract)). However, the block number and timestamp are guaranteed to be monotonically increasing, and to not be higher than those on L1; this means the impact of the difference in block number and timestamp would be that, in the worst case, the L2 epochs would lag behind L1 for a while until the L2 “catches up”.

Considering this, we can deploy the EpochManager to L2 as-is.

## Keeper reward from Reservoir or using a keeper service

When thinking about how to reward keepers that call the drip function, there are several possible approaches. The most straightforward one (as implemented in PR 571), is to not provide a reward at all, and rely on altruistic keepers or network participants that are incentivized because they want rewards to be distributed. This, however, could lead to a "musical chairs" / tragedy of the commons scenario where nobody wants to call the drip, and the last indexer left without rewards is forced to call it and spend the gas. Depending on how indexers time their allocations, this could end up falling repeatedly on the same indexer producing an unfair burden. Moreover, a small indexer would be at a disadvantage here as the gas cost would be a higher fraction of the allocation rewards.

So if we want to provide a reward, there are decentralized protocols that allow this, namely Gelato, keep3r network, and Chainlink Keepers. These could provide us with a market for keepers and several ways to provide the rewards. The challenge in using these, however, lies in enforcing the right parameters for submission cost, gas price and max gas for the L2 retryable ticket that are arguments to the drip functions. These values must be queried from the Arbitrum RPC so they are not easily checked on-chain, and these protocols provide varying levels of support for off-chain actions when setting up a rewarded task. Another thing that wouldn't be straightforward when using these is how to top up the rewards, which could come from token issuance or from protocol participants "chipping in", but we'd need to get it to the corresponding third-party protocol's billing system.

This leads us to our proposal of using token issuance for the reward, and not relying on any external protocol. Network participants, in particular Indexers, are well suited to perform this task, as they have an incentive as mentioned above but also already have infrastructure (the Indexer Agent) that interacts with the network contracts. Adding a reward from token issuance should solve the tragedy of the commons issue, and we can encode a linearly increasing reward so as to encourage actors to call the function within a specific time range.

# Improvements and future work

This proposal covers the initial deployment and the new rewards distribution mechanism for L2. There are improvements to these, and potential next steps in the protocol's move to L2, that are out of scope for this GIP but can be addressed in future proposals:
- **Schedule for l2RewardsFraction increases:** this GIP proposes setting the `l2RewardsFraction` to zero during the experimental phase, and gradually increasing it to incentivize the move to L2, but doesn't specify the times at which this should happen, or what values should be used. This can be addressed with a separate GIP (or several).
- **State migration to L2:** There are various ways in which contracts in L1 can provide functions to facilitate moving state (stakes, allocations, curation signal) to L2. We're exploring some of these and future GIPs can propose implementations for these. For example (though the actual proposal might vary): we can allow migrating subgraphs to L2 by adding a function on GNS that sends the curation signal at the GNS level to the GNS on L2, and encodes a blockhash which particular Curators can then use to prove (using the native EVM Merkle-Patricia tries) their particular amount of signal. Similar approaches could be used for Indexer and Delegator stakes.

# Copyright Waiver

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
