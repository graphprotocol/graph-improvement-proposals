---
GIP: "0020"
Title: Unattestable Indexer Responses
Authors: Jannis Pohlmann (jannis@edgeandnode.com)
Created: 2021-09-30
Updated: 2021-10-12
Stage: Proposal
Discussions-To: https://forum.thegraph.com/t/gip-00xx-unattestable-indexer-responses/2607
Community-Polls:
Governance Proposals:
Implementations:
---

# Abstract

This GIP proposes a mechanism through which indexers can bail out of executing a
query in a way that avoids slashing risk and allows clients such as gateways to
detect and handle indexer-specific issues gracefully.

# Motivation

Right now, indexers are expected to respond to any query sent their way with a
response that includes the query result and an attestation. The attestation
provides a way to cross-check the result with that of other indexers and create
on-chain disputes if there is a mismatch and the indexer is assumed to have
served bad data.

However, there is currently no way for the indexer to express that something is
wrong on their end and that they would like to bail out of executing the query
and providing an attestation for it, which would put them at a slashing risk.
Possible situations where this applies are if the indexer encounters a database
corruption during query execution or if the indexer is under high load and can't
handle the incoming query.

Framed more generally, any result that an indexer believes may be incorrect or
non-deterministic poses a slashing risk unless there is a way for the indexer to
signal that it cannot provide an attestation for the result.

As a side note, it is worth distinguishing between three types of indexer query
errors:

_Non-Deterministic_: An error that will not occur for all indexers. This type of
error is one example of a non-attestable error/result.

_Deterministic_: An error that will occur for all indexers.

_Attestable_: An error that is deterministic and standardized (in terms of the
conditions under which it happens and the error message in the response).

# High Level Description

This GIP proposes that indexers can bail out of executing a query and providing
an attestation for it by simply not returning an attestation. This allows
clients to distinguish between _attested_ (good) and _unattested_ (indexer
bailed out) results. Upon encountering unattested results, clients can then try
additional indexers in the hopes of finding one that is able to provide a result
with an attestation.

Several components need to work together to implement this distinction:

- graph-node needs to be able to flag unattestable results to indexer-service,
- indexer-service needs to detect such results and return them without providing
  an attestation; it must also drop the query fee receipt received with the
  query,
- clients such as gateways need to detect unattested responses and take measures
  to guarantee the best possible quality of services to dApps, e.g. by trying
  additional indexers and updating indexer reliability metrics accordingly.

The detailed specification below describes how this is achieved.

_Note that for subgraphs with experimental features like fulltext search,
indexers are expected to return no attestation either to prevent slashing. This
expectation conflicts with the above proposal, so as part of this GIP, the
author proposes to return attestations after all, even for subgraphs with
experimental features, and exclude these subgraphs from disputes/slashing._

# Detailed Specification

The following changes are proposed to graph-node and indexer-service:

1. A new `Graph-Attestable` HTTP header is introduced in graph-node. By default,
   it is set to `false`. Anything that that could make query results
   non-attestable while executing the query should set the `Graph-Attestable`
   header to `false`. Typically, this will be unexpected errors (e.g. database
   errors) but it can also be other non-deterministic queries such as queries
   against the latest block of a network, which may or may not be removed again
   from the chain later.

   The implementation could collect all problems/errors from executing a query
   and at the end compute whether the result is attestable by returning
   `Graph-Attestable` if any of them are non-attestable.

2. When indexer-service receives a query result from graph-node, it checks for
   the `Graph-Attestable` header. It defaults to `false` if the header is not
   present (note: this requires indexers to update both graph-node and
   indexer-service together when rolling out this feature).

   1. If the header is `true`, indexer-service does what it currently does: it
      remembers the query fee receipt received with the query and returns the
      query result along with an attestation:

      ```json
      {
        "graphQLResponse": "...",
        "attestation": "..."
      }
      ```

   2. If the header is `false`, indexer-service drops the receipt received along
      with the query and returns the response _without_ an attestation:
      ```json
      {
        "graphQLResponse": "..."
      }
      ```

# Backwards Compatibility

This is a breaking change in that clients like gateways may currently rely on
the indexer response to include an `attestation` field. However, the gateway
deployed at https://gateway.thegraph.com does not. And at the time of writing
this gateway is _the_ way of querying the network. As long as the gateway is
updated prior to releasing this feature to indexers, things should be fine.

# Dependencies

This GIP has no dependencies.

# Risks and Security Considerations

One risk is that indexers end up bailing out _a lot_. This is a problem that
clients can and should deal with by preferring indexers that bail out less frequently.

A risk of making `Graph-Attestable` a boolean flag is that we might want to
differentiate between different reasons for why a result is not attestable in
the future. This could be added via separate headers like e.g. a
`Graph-Unattestable-Reason` or similar.

# Validation

The changes to graph-node should come with graph-node integration tests that
verify that the `Graph-Attestable` header is set to `true` in all situations
where the query results are attestable. Since the default is `false`, we don't
need to test all possibly non-deterministic cases.

The changes in indexer-selection and clients such as gateways may best be
verified in a local network or on testnet.

# Rationale and Alternatives

The author sees no alternative to this feature in general. There needs to be a
way to distinguish between attestable or deterministic query errors (such as a
query/schema mismatch) and non-attestable query errors (due to indexer-internal
issues that lead to non-determinism).

There are, however, alternative ways to implementing the solution. Instead of a
`Graph-Attestable` header field in graph-node and omiting `attestation` fields
in responses returned by indexer-service, specific HTTP status codes could be
used. The author was unable to find suitable codes and finds header fields to be
more expressive.

# Copyright Waiver

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
