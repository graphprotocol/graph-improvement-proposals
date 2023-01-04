---
GIP: "0013"
Title: Reduce Curation Tax
Authors: Brandon Ramirez <brandon@edgeandnode.com>, Dave Kajpust <dave@edgeandnode.com>
Created: 2021-07-13
Updated: 2021-10-08
Stage: Candidate
Discussions-To: https://forum.thegraph.com/t/proposal-to-reduce-curation-tax/2212/17
Community-Polls: https://snapshot.org/#/graphprotocol.eth/proposal/QmRz29aE4TXpi9HrNbn6ZA1sRF1xEBeGbw8HxRpHZRZ4rD
Category: Economic Parameters
---

# Abstract

This GIP proposes reducing the Curation Tax parameter in the protocol from 2.5% to 1%. The proposal evaluates this change in the context of a previously undisclosed attack we refer to as a Subgraph Withholding Attack.

# Motivation

As production subgraphs migrate to The Graph's decentralized network, we have collected more data and feedback on the total costs of using the network as a subgraph developer. As many subgraph developers initially signaled ~100K GRT initially when using the network, the cost of upgrading a subgraph currently sits around 2.5K GRT. This cost is in addition to the opportunity cost of locking up the signal as well as the cost of actually paying for query fees.

| Note                                                                                                                               |
| ---------------------------------------------------------------------------------------------------------------------------------- |
| This ignores the curation tax costs associated with signal delegation, which will be addressed in a separate GIP. See Future Work. |

One of the major constraints in lowering the curation tax is imposed by a class of economic attack that we refer to as a _Subgraph Withholding Attack_. There are other benefits to a curation tax, such as discouraging churn of subgraphs, which leads to expensive and wasteful re-indexing work by Indexers, but formalizing these benefits are left to future work, and in this proposal, we assume that preventing the Subgraph Withholding Attack is the binding constraint in reducing the curation tax.

# High Level Description

Our analysis shows that under current network conditions, the curation tax could be reduced to ~0.46% if we assume the Subgraph Oracle can disable indexing rewards on unavailable subgraphs within 30 minutes.

See Appendix A for a complete discussion of the Subgraph Withholding Attack.

# Detailed Specification

## Zero Attack Profit Invariant

From Appendix A, we see that the invariant that must be satisfied to prevent a Subgraph Withholding Attack is:

$$
\tau_c > \frac{M*i}{\psi_{\bar{a}}}*\frac{1-(1+i)^T}{1-(1+i)}
$$

where

- $\tau_c$ is the curation tax.
- $M$ is the total token supply of GRT at the start of the attack.
- $\psi_{\bar{a}}$ is the total network signal not controlled by the attacker.
- $i$ is the per-block issuance rate
- $T$ is the number of blocks until the Subgraph Oracle disables indexing rewards on the attacker’s subgraph.

First, we assume the Subgraph Oracle can disable indexing rewards on any unavailable subgraph within ~30 minutes ($T=137$ blocks).

For the remaining variables, we can look at the state of the network as of this writing:

- $M$ is $10.168*10^9$
- $\psi_{\bar{a}}$ is $3.684*10^6$
- $i$ is $0.0000000122$

Plugging in the above gives us a lower bound for the curation tax of $~0.46\%$.

We choose $1\%$ as the new curation tax to leave ample margin for error due to changing network conditions or unforeseen circumstances that cause the Subgraph Oracle to take longer to disable indexing rewards on an unavailable subgraph.

## Smart Contract Variables

Adopting this proposal requires setting `_curationTaxPercentage` variable of the `Curation` contract currently deployed at `0x8FE00a685Bcb3B2cc296ff6FfEaB10acA4CE1538` to `10000` (the parameter is expressed in parts per million).

# Backwards Compatibility

This change is backwards compatible at the protocol level, however, user interfaces such as network explorers and rewards calculators may need to be updated to reflect the latest economic parameters.

# Risks and Security Considerations

A risk is that the economic variables in the network or the performance of the Subgraph Oracle change in such a way as to require the curation tax to be higher than $1\%$.

Another risk is that lowering the curation tax leads to more volatile curation activity in the network, even if not explicitly related to this economic attack, which produces a noisy signal for Indexers attempting to identify more valuable subgraphs to index.

# Validation

The analysis in this proposal should be reviewed by the community prior to acceptance. The code that implements the curation tax has already been audited as a part of previous adits.

# Future Work

## Lowering Subgraph Owner Share of Curation Tax

Subgraph owners don't just pay the curation tax for their own signaling, but also pay a portion of the curation tax for signal that has been delegated to their subgraph. A future GIP explores lowering the share of this additional curation tax that subgraph owners are charged.

# Appendix A - Subgraph Withholding Attack

## Background

Indexing rewards in The Graph are paid from new token issuance and distributed to Indexers proportional to the amount of signal on the subgraphs they are indexing relative to total network signal. Furthermore, the rewards are distributed proportionally between Indexers based on all allocated stakes on said subgraphs.

Signaling on a subgraph involves minting shares in a bonding curve by depositing GRT into a bonding curve and paying a curation tax. In a Subgraph Withholding Attack, an attacker signals on a subgraph, but does not publish the IPFS file that defines the subgraph, and thus is able to monopolize indexing rewards on the subgraph as the only Indexer that is able to submit the valid Proofs of Indexing (PoIs) that are required to collect rewards.

The Subgraph Oracle (SO) was designed to mitigate this attack by disabling indexing rewards on subgraphs whose subgraph manifests are not available, and thus cannot be indexed by other Indexers.

## Attack + Mitigation Steps

1. Attacker signals on new subgraph
   - Attacker pays curation tax
   - Attacker does not publish subgraph manifest
2. Attacker allocates stake on subgraph
3. _New epoch begins_
4. Concurrently:
   - After n epochs attacker claims indexing rewards
   - After m blocks SO disables indexing rewards on subgraph
5. Attacker unsignals and unallocates stake on the subgraph.

## Analysis

### Revenue

The attacker earns indexing rewards on the subgraph up until indexing rewards are disabled on the subgraph. Even though allocations are measured in epochs, indexing rewards accumulate continuously on a per-block basis. Thus the revenue from an attack is determined by the number of blocks until a Subgraph Oracle detects the unavailable subgraph and disables indexing rewards.

| Note                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| This assumes that an attacker is able to collect their indexing rewards immediately before the SO disables indexing rewards. Once the SO disables the subgraph, past rewards are nullified, and future rewards cannot be earned. We also assume that the attacker is able to time the creation of their allocation towards the end of the epoch such that they are always able to claim rewards as long as they can front-run the SO's transaction. |

We can describe the revenue from the attack $R$ as follows:

$$
R=\sum_{t=0}^{T-1}M*(1+i)^t*i*\frac{\psi_{a}}{\psi_{a}+\psi_{\bar{a}t}}
$$

Where:

- $M$ is the total token supply when the attacker opened their allocation.
- $T$ is the number of blocks required for the Subgraph Oracle to disable indexing rewards on the subgraph.
- $i$ is the per-block issuance rate of GRT, used to pay indexing rewards.
- $\psi_{a}$ is the GRT signaled by the attacker. For simplicity, we assume the attacker's signal is constant throughout the attack.
- $\psi_{\bar{a}t}$ is the GRT signaled by everyone but the attacker at time $t$.

### Cost

The cost of this attack is the curation tax, the opportunity cost of the signal and allocated stake used in the attack, as well as the gas costs of signaling and allocating. Note that we assume the allocated stake to be infinitesimally small, as there is no incentive for the attacker to provide more than the smallest unit of GRT as allocated stake.

We can describe the cost $C$ as follows:

$$
C = \psi_{a}*\tau_c + [\psi_{a}*(1+r)^T-\psi_a]+c_g
$$

Where:

- $\tau_c$ is the curation tax
- $r$ is the risk-free rate of return
- $c_g$ is the cost of gas.

### Profit

The profit from this attack can be expressed as follows:

$$
\Pi=\lbrack\sum_{t=0}^{T-1}M*(1+i)^t*i*\frac{\psi_{a}}{\psi_{a}+\psi_{\bar{a}t}}\rbrack - \psi_{a}*\tau_c - [\psi_{a}*(1+r)^T-\psi_a]-c_g
$$

### Zero-profit condition

Let's examine how we should set the curation tax $\tau_c$ to ensure zero profits for a subgraph-withholding attack.

First, let's make the following simplifying assumptions:

- $c_g \approx0$ because this makes the analysis more conservative and gas costs matter much less for an attacker with very large signal.
- $\psi_{\bar{a}t}\approx\psi_{\bar{a}}$ where $\psi_{\bar{a}}$ is the rest of the networks signal throughout the attack. We assume it's constant for simplicity.
- We will assume $r \approx 0$ given that the attack takes place over a relatively short period of time and so the opportunity cost is likely to be insignificant.

This reduces the above profit formula to:

$$
\Pi=\frac{\psi_a}{\psi_a+\psi_{\bar{a}}}*\lbrack\sum_{t=0}^{T-1} M*(1+i)^t*i\rbrack-\psi_a*\tau_c
$$

$$
\rightarrow \frac{\psi_a}{\psi_a+\psi_{\bar{a}}}*\lbrack\sum_{t=0}^{T-1} M*i*(1+i)^t\rbrack-\psi_a*\tau_c
$$

$$
\rightarrow \frac{\psi_a}{\psi_a+\psi_{\bar{a}}}*M*i*\frac{1-(1+i)^T}{1-(1+i)}-\psi_a*\tau_c
$$

Now we can describe the following inequality that must hold for profits to be less than zero.

$$
0>M*i*\frac{\psi_a}{\psi_a+\psi_{\bar{a}}}*\frac{1-(1+i)^T}{1-(1+i)}-\psi_a*\tau_c \\
$$

$$
\rightarrow \psi_a*\tau_c > M*i*\frac{\psi_a}{\psi_a+\psi_{\bar{a}}}*\frac{1-(1+i)^T}{1-(1+i)}
$$

$$
\rightarrow \tau_c > \frac{M*i}{\psi_a+\psi_{\bar{a}}}*\frac{1-(1+i)^T}{1-(1+i)}
$$

We can see that if the invariant holds for a relatively small $\psi_a \approx 0$, then it should hold for all values of $\psi_a$, as the quantity right-hand side decreases as it increases:

$$
\rightarrow \tau_c > \frac{M*i}{\psi_{\bar{a}}}*\frac{1-(1+i)^T}{1-(1+i)}
$$

# Copyright Waiver

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
