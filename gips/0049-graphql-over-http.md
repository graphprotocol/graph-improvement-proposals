---
GIP: "0049"
Title: GraphQL over HTTP spec
Authors: Denis Badurina <badurinadenis@gmail.com>
Created: 2023-02-28
Updated: 2023-02-28
Stage: Proposal
Category: GraphQL API
---

# Abstract

The GraphQL foundation is in the process of standardising how GraphQL is transported over HTTP, with the responsable team of the [GraphQL over HTTP work group](https://github.com/graphql/graphql-over-http). Efforts are in gear and the spec is slowly, but steadily, moving forwards. At the time of writing the [GraphQL over HTTP spec](https://graphql.github.io/graphql-over-http/) is in "Stage 1: Proposal".

# Motivation

Implementing the [GraphQL over HTTP spec](https://graphql.github.io/graphql-over-http/) will make sure The Graph supports, and is compliant, with the current standard of transporting GraphQL over HTTP.

Doing so will ensure that connecting clients have a standardised and predictable means of communication with the servers.

# Detailed Specification

Alongside the specification, there is an official reference implementation [graphql-http](https://github.com/graphql/graphql-http) (and an accompanying online audit tool [graphql-http.com](https://graphql-http.com)) that can aid the process and establish a clear path to compliant implementation.

Currently, [GraphQL Yoga server](https://the-guild.dev/graphql/yoga-server) is fully compliant which in turn makes the [graph-client fully compliant](https://github.com/graphql/graphql-http/tree/main/implementations/graph-client) too.

However, hosted subgraphs (like the [sushiswap subgraph](https://api.thegraph.com/subgraphs/name/sushiswap/exchange/graphql)) [have a few hiccups](https://github.com/graphql/graphql-http/blob/main/implementations/thegraph/README.md) and are therefore not compliant.

# Backwards Compatibility

The GraphQL over HTTP spec is aimed to support existing server implementations.

# Copyright Waiver

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
