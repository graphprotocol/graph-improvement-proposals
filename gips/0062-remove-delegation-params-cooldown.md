---
GIP: "0062"
Title: Remove delegation parameters cooldown
Authors: Pablo Carranza Velez <pablo@edgeandnode.com>
Created: 2023-10-11
Updated: 2023-10-11
Stage: Draft
Discussions-To: TODO
Category: Economic Parameters
Implementations: https://github.com/graphprotocol/contracts/pull/866
Audits: TODO
---

# Abstract

This GIP proposes removing the "delegation parameters cooldown" feature. This cooldown defines how long an Indexer must wait before setting the delegation query fee and rewards cut. We present a proposal to remove it as it is largely unused and is error-prone.

# Motivation

Indexers can set a "cooldown" parameter when setting their delegation parameters (i.e. the query fee and rewards cut). This is largely unused, with no Indexers having used it in the last year on mainnet. Only a single Indexer has used it on Arbitrum One, and due to a UI issue they set it to an incorrect value, locking their delegation parameters for several months. Since the feature is unused and causes problems, we propose removing it and ignoring the value that was set by that single Indexer. This will allow them to change their delegation parameters immediately. Indexers are already motivated to not change parameters too often as this would deter Delegators from delegating to them in the future.

# High-Level Description

The change simply requires removing the check for the cooldown value when setting delegation parameters, and marking all related variables as deprecated. We also remove the minimum cooldown setting that was controlled by governance.

# Backward Compatibility

The `setDelegationParametersCooldown()` and `delegationParametersCooldown()` external functions are removed from the Staking contract on L1 and L2.

# Risks and Security Considerations

Indexers could grief delegators by changing rewards or fee cuts right before collecting rewards or fees. However, this would prompt Delegators to undelegate, which we believe should deter Indexers from doing this.

# Validation

The PR will be audited and tested on testnet as with other protocol changes.

# Copyright Waiver

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
