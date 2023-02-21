---
GIP: 0035
Title: GraphQL Live Queries
Authors: Dotan Simha dotan@the-guild.dev
Created: 2022-22-06
Stage: Draft
Discussions-To: https://forum.thegraph.com/t/gip-0035-graphql-live-queries/3432
---

Abstract
--------

Today, The Graph API provides a rich interface for querying data indexed from the blockchain. The interface today consists of `type Query` that provides an entry point for the graph of data.

To provide a more easy querying protocol for real-time data, we are looking into the option of implementing `@live` queries as part of the GraphQL interface of The Graph.

This GIP describes The Graph's real-time querying policy and a new suite of tools to help dApp developers query real-time data from their Subgraphs.

High-Level Description
------------------------------------------------------------------------------------------------------------------

This GIP describes the following:

-   The background for choosing GraphQL `@live` queries over GraphQL `Subscription`s.
-   The network transport that will be used for shipping the responses
-   The execution flow and integration with the network gateway
-   Compatibility layer for `graph-node`
-   Query costs calculations

### Subscriptions vs `@live` queries

At the beginning of the discussion around real-time GraphQL, we have the differences between GraphQL `@live` queries and GraphQL Subscriptions.

The main difference is around the approach:

While GraphQL `@live` queries is an extension of a regular GraphQL `query { ... }`, with just an added `directive` on specific fields, the GraphQL Subscription approach is based on an event.

Here's an example for creating a `@live` query, that asks to watch a specific field out of the entire query:

```
# Regular GraphQL query annotated with @live directive
# In this example, the consumer can control what are the relevant changes that needs to be watched in real-time.
query tokens {
  pairs {
    token1 {
      id
      name
      decimals @live
    }
    token2 {
      id
      name
      decimals @live
    }
  }
}

```

With `Subscription`s approach, the consumer is limited to the event that caused the change, and it requires managing complex filtering before getting to the actual data changes:

```
# GraphQL subscription watching specific events, then based on the event we can reach the data we need.
subscription onPairTokenChange {
  onPairTokenChange(pair: "...") {
    pair {
      token1 {
        id
        name
        decimals
      }
      token2 {
        id
        name
        decimals
      }
    }
  }
}

```

With this approach, the Subgraph definition needs to be in charge of exposing the relevant events, and the GraphQL schema is in charge of filtering and state management for each connection.

In addition to that, GraphQL Subscriptions are usually an extension for a GraphQL query: you perform an initial query, and only then subscribe to more data or specific data changes.

* * * * *

The initial thinking process was trying to figure out what dApps need: do they need to know that the data has changed, or do they need to know why the data has changed?

Since Subgraph developers can already manage the lifecycle of `why` the data has changed (either using Substreams, or just `dataSources` definition), we needed a way to allow Subgraph consumers to know that some data has changed, and make it easy and accessible to receive real-time data.

Subgraphs developer and dApps developer can often be different entities: you can only be a consumer of a Subgraph, without the ability to manage the events, schema, or definition.

With GraphQL subscriptions, we might have a limited event catalog for the consumer. With `@live` queries, the power of choosing the real-time views is in the hands of the consumer.

### SSE: The user-facing transport

For the transport layer, we had 2 options: `WebSocket` or `SSE` (Server-Sent Events). We decided to go with SSE for the real-time transport because of the following reasons:

-   WebSocket opens a two-way channel, while SSE only does a one-way channel (Server → Client). We only need the ability to push data from the server to the client.
-   WebSocket is using its protocol and requires a secondary TCP connection, while SSE can leverage the existing HTTP layer.
-   WebSocket is not (yet) an effective solution - it's heavy and requires a special setup for mobile apps. SSE acts as a regular HTTP request with multiple responses.
-   WebSocket requires a complex setup for consumers, while SSE can simply use `fetch` protocol.

With SSE setup, we can also easily track the health of the connection (since it acts as a regular, long-living HTTP request) and ensure the delivery of responses.

### `@live` queries execution

To implement `@live` queries in an easy way for consumers, we are going to integrate and extend the integration of the Gateway with indexers and `graph-node` instances.

-   The Graph Gateway will receive a GraphQL request and will detect if it's a `@live` query (by traversing the GraphQL operation AST)
    -   If the request does not include any `@live` directive → a normal execution is performed.
    -   If the request includes one or more `@live` directives defined on fields, then:
        -   If the request does not include `Accept: text/event-stream` header, an error will be thrown.
        -   The Graph Gateway will respond with the following HTTP attributes:
            1.  Data description headers: `Cache-Control: no-cache`, `Content-Encoding: none`, `Content-Type: text/event-stream` and `Connection: keep-alive`.
            2.  A periodic message with the payload: `event: ping` will be sent to keep the SSE connection alive
            3.  The Graph Gateway will create a clean GraphQL operation (without `@live` the directives), and execute it periodically (pooling) to `graph-node` instances:
            4.  The Graph Gateway will store the hash of the root GraphQL objects - and will use that to match every new polling response with the stored hashes,
                1.  In case of changes to hashes, the new version is stored, and an event is emitted to the stream with the payload: `data: {"data": { }, "errors": [] }`
            5.  In case of a socket disconnect event, the Graph Gateway will stop polling and drop the in-memory hash for the given queries.

#### Technical overview of the execution flow

Example of an input query that will trigger `@live` workflow in The Graph Gateway:

```
query tokens {
  pairs {
    token1 {
      id
      name
      decimals @live
    }
    token2 {
      id
      name
      decimals @live
    }
  }
}

```

This will trigger the following query periodically to `graph-node` instances that are part of The Graph Network:

```
query tokens {
  pairs {
    token1 {
      id
      name
      decimals
    }
    token2 {
      id
      name
      decimals
    }
  }
}

```

A hash will be calculated based on every selection-set coordinate that has the `@live` directive specified: `pairs.token1.decimals` and `pairs.token2.decimals` and will be stored.

A periodical query will be executed to fetch the latest data from `graph-node` instances, and when `pairs.token1.decimals` or `pairs.token2.decimals` changes, we'll publish a new event to the SSE transport stream.

> Side note: we can consider to drop all non-`@live` fields from the periodical queries, and run only the fields that we actually want to watch.

#### Compatibility layer in `graph-node`

To allow an easy and consistent development process, and also to allow users to run their instances of `graph-node` in standalone mode, we'll need to implement a backward compatibility layer in `graph-node` code, to run the same process (`@live` detection, stream execution based on polling, and an SSE transport) and reply with a streamed response based on SSE protocol.

### Integration with Graph Client

The Graph Client acts as a network and GraphQL execution layer, between the consumer's code (or the consumer's GraphQL client).

To comply with the SSE-based transport, and with the possibility of getting a streamed response (instead of a single response), the Graph Client network layer will need to perform the following (as part of the GraphQL-Mesh execution layer):

1.  If the outgoing GraphQL operation includes a `@live` query, a stream request needs to be constructed:
    1.  A header with `Accept: text/event-stream` will be added to the outgoing request.
    2.  An `AsyncInterable` will be returned from the GraphQL execution layer, where every iterator object will consist of a valid GraphQL response.
    3.  If a GraphQL client is configured for the consumer, an adaptation layer will need to handle the GraphQL normalized cache updated (by emitting multiple responses).

### Changes to `graph-node`

Besides the changes described above for the compatibility layer, we'll need the following changes implemented in `graph-node`:

1.  Add the SDL definition for the `@live` directive.
2.  Consider dropping the broken GraphQL `Subscription` implementation and removing it from the schema (note: this is a breaking change).
3.  Improve the HTTP layer to support streamed responses based on the SSE protocol.

### Query costs

To address the changes in the query execution flow with the new stream-based responses, The Graph Gateway will have to check for deterministic responses for every event emitted on the stream). This will also lead to a change in the way the Gateway calculates the query fees.

> Still need to figure out this part of the GIP.