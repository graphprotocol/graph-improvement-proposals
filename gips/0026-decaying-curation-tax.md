---
GIP: 0026
Title: Decaying Curation Tax
Authors: Juan Manuel Rodriguez Defago <juan@graphops.xyz>, Brandon Ramirez <brandon@edgeandnode.com>
Created: 2022-03-18
Stage: Proposal
Discussions-To: https://forum.thegraph.com/t/decaying-curation-tax/3210
Category: Protocol Logic
Depends-On: 0025
Implementations: TODO
---

# Abstract

This proposal modifies the existing curation tax on a subgraph deployment bonding curves to decay linearly over time.

# Background
The Graph currently has a curation tax aimed at discouraging too frequent upgrades of subgraphs. Indexing a subgraph from genesis implies a large upfront cost to Indexers before they have the opportunity to collect indexing rewards or serve queries. If subgraphs upgraded too frequently or unpredictably, then one might expect Indexer profitability, and by extension participation, to decline.

# Motivation

The curation tax as it exists today is a blunt instrument. It is levied irrespective of how long a Curator is signaled on the subgraph deployment bonding curve. This is in part due to the way curation royatlies are distributed--into the reserves of the bonding curve. The protocol currently levies the tax on deposit (signal) rather than withdraw (unsignal) because the latter would also imply taxing a portion of the earned royalties.

With the new principal-protected bonding curve design introduced in [GIP-0025](./0025-principal-protected-bonding-curves.md), curation royalty distribution is no longer carried out via the reserves of the bonding curve, but rather through separate payouts.

This presents the opportunity to revisit the decision to levy the curation tax on deposit. By taxing on withdrawal we may incorporate the amount of time the Curator was signaled into the size of the tax that is levied.

Reducing the curation tax makes the curation market more economically efficient and reduces the costs of using the network to subgraph developers.

# High Level Description
This proposal requires the following additional logic:
1. Tracking the cost-weighted average time a Curator is signaled on a subgraph.
2. Modifying the curation tax to be levied on withdraw and decay linearly to zero as a function of cost-weighted average time signaled.

## Tracking Cost-Weighted Average Time Signaled
When a Curator signals on a subgraph deployment bonding curve, they deposit GRT and receive newly minted curation shares in return. These shares are ERC-20 compatible[^1] and may be transfered to other Etheruem accounts. The ERC-20 standard leverages an account model, rather than a UTXO model[^2], meaning that when the same account receives curation shares multiple times, these shares are combined into a single balance rather than tracked separately.

This raises the question of how to track the amount of time signaled for such a balance that has been added to multiple times. The approach proposed here, and the one that requires the least additional bookkeping, is to track the *cost-weighted average time basis* of a balance of curation shares. 

This is simply a weighted arithmetic mean of the times at which shares were minted, measured in blocks, of all the balances of shares that were combined into a single balance, either through repeated signaling or transfering. The average is weighted by the amount that was deposited for each respective balance of shares, also known as the cost basis.

For reasons similar to those described above, when the same account receives shares in multiple distinct transfers, the respective cost bases are combined into a single share-weighted average cost basis. This bookkeping was introduced in [GIP-0025](./0025-principal-protected-bonding-curves.md) to support principal-protected bonding curves. 

In order to compute the cost-weighted average time signaled for a given amount of shares being burned, we simply subtract the cost-weighted average time basis from the current time.

## Linearly Decaying Curation Tax
The curation tax, currently levied on deposit, must be replaced by a curation tax levied on withdraw that decays linearly to zero as a function of the cost-weighted average time signaled.

A new governance parameter must be introduced to the protocol to modify the time it takes for the curation tax to decay to zero. We will call this the **curation tax decay time.**

# Detailed Specification

## Psuedocode

#### **Signaling.** Alice deposits tokens to mint curation shares.
  1. Alice has $M$ curation shares with an average cost basis of $x$ and an average time basis of $a$ 
  2. Alice deposits $y$ tokens to mint $N$ shares at time $b$
  3. Alice's new average time basis is $(x \cdot a + y \cdot b)/(x+y)$
     1. If Alice's previous balance was zero then we assign a previous average cost basis $x=0$.
  4. Alice's new average cost basis is $(M \cdot x + N \cdot y)(M+N)$.
#### **Transfer.** Alice has previously minted shares trasnfered to her.
  1. Alice has $M$ curation shares with an average cost basis of $x$ and an average time basis of $a$
  2. Alice receives $N$ shares having an average cost basis of $y$ and an average time basis of $b$.
  3. Alice's new average time basis is $(x \cdot a + y \cdot b)/(x+y)$
     - If Alice's previous balance was zero then we assign a previous average cost basis $x=0$.
  4. Alice's new average cost basis is $(M \cdot x + N \cdot y)(M+N)$.
  
#### **Unsignaling**. Alice burns previously minted shares to withdraw reserve tokens.
  1. Alice has $M$ curation shares with an average cost basis of $x$ and an average time basis of $a$. Curation tax decay time is represented by $\Delta$ and the initial curation tax before any decay is represented by $\tau_{c,t=0}$.
  2. Alice unsignals $N$, where $N \le M$ curation shares at time $b$.
  3. The *Cost-Weighted Average Time Signaled* is simply $b-a$.
  4. If $N=M$
     - Alice's new average cost basis is reset to $0$.
     - Alice's new average time basis is undefined.
  5. If $N<M$
     1. Alice's average cost basis remains $x$.
     2. Alice's average time basis remains $a$.
   6. The curation tax percentage $\tau_c$ is $\max(0, \tau_{c,t=0} - \frac{\tau_{c,t=0}}{\Delta} \cdot t)$ where $t$ is measured in blocks.
   7. The reserves to withdraw and the curation royalties to withdraw are computed separately, according to the logic in [GIP-0025](./0025-principal-protected-bonding-curves.md).
   8. The curation tax is levied only *on the reserves to withdraw* and not on any curation royalties being withdrawn.

# Prototype
The `withDecayingCapitalGains` mixin in this [Observable HQ notebook](https://observablehq.com/d/ee23aa407cb16c01) comprising Bonding Curve prototypes, has a calculation similar to the average time basis calculation described here--except that it uses a share-weighted average time basis rather than a cost-weighted average time basis.

# Upgrade Path
This change should be implemented for newly created bonding curves but cannot be implemented retroactively for existing subgraph deployment bonding curves.

# Backwards Compatibility
This proposal doesn't introduce any breaking changes in the bonding curve interfaces, though some UIs may need to be updated to display a different curation tax to users.

# Dependencies
This proposal relies on modifications to the bonding curve and curation royalties calculations as outlined as a part of the principal-protected bonding curves work in [GIP-0025](./0025-principal-protected-bonding-curves.md).


# Risks and Security Considerations

## Too Frequent Upgrades
A possible risk is that we do not set the curation tax decay time long enough, or similarly that we set the initial curation tax before decay too low.

This could result in upgrades that are too frequent for Indexers to effectively manage their allocations and indexing activity in a way that is profitable and that doesn't introduce a lot of wasted time and effort.

# Validation

The changes introduced here should go through a smart contract audit and receive community feedback, with special focus on Curators, Subgraph Developers and Indexers.

# Rationale and Alternatives

## Decay Function
We could choose different functional forms for the decay function, such as exponential, but using a linear decay function keeps the implementation simple to reason about and implement.

# Copyright Waiver

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

# Notes
[^1]: https://ethereum.org/en/developers/docs/standards/tokens/erc-20/

[^2]: https://en.wikipedia.org/wiki/Unspent_transaction_output