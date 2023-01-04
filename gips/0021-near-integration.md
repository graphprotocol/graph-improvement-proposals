---
GIP: 0021
Title: GIP-0021: NEAR Indexing for Graph Node
Authors: Adam Fuller (adam@edgeandnode.com)
Created: 2021-10-11
Updated: 2021-10-30
Stage: Draft
Category: Subgraph API
---

# Abstract

This proposal describes the integration of NEAR into Graph Node as the first non-EVM chain. This involves the extraction of data from NEAR, the introduction of a new `kind` of subgraph data source (`near`), and the corresponding extension of the Subgraph API.

# Motivation

Realising the vision of The Graph means bringing the Graph Node's indexing capabilities to other Layer 1 chains. This will expand the protocol's addressable market and opportunities for future growth.

NEAR is a smart contract platform, with a growing developer community. It is similar to Ethereum, but has some structural differences. It is therefore a good candidate to be the first new chain.

# Prior Art

This specification builds on the existing Graph Node implementation for Ethereum, the Multiblockchain refactor to remove Ethereum-specific components from the core of Graph Node, and the integration of the StreamingFast firehose into Graph Node.

In terms of NEAR specific implementations, we refer to the [NEAR indexer framework](https://github.com/near/nearcore/tree/master/chain/indexer) and [derivatives](https://github.com/near/near-indexer-for-explorer), and the [Flux Capacitor](https://github.com/fluxprotocol/flux-capacitor/) (a custom indexer built on NEAR).

# High Level Description

NEAR data extraction will take place via a NEAR-specific Firehose implementation. This will enable ingestion of NEAR blocks into the Graph Node, which will be updated to support this new chain.

Subgraph developers will be able to specify a new kind of dataSource (`near`) for their subgraphs, with specific triggers relevant to NEAR applications.

A Near API will be added to `graph-ts`, for the selected trigger-types and useful NEAR data-types.

# Detailed Specification

## Data extraction from NEAR into Graph Node

NEAR data extraction will take place via a [Firehose implementation](https://github.com/streamingfast/near-dm-indexer) leveraging the [NEAR indexer framework](https://github.com/near/nearcore/tree/master/chain/indexer). This will enable ongoing indexing, as well as indexing past blocks.

The NEAR Firehose gRPC endpoint will be consumed by Graph Node, which will implement a dedicated FirehoseBlockStream, NEAR specific types, and filtering for triggers.

### Operational requirements

In order to index NEAR subgraphs, Indexers will therefore need to run the following:

- NEAR Indexer Framework with Firehose instrumentation
- NEAR Firehose Component(s)
- Graph Node with Firehose endpoint configured

## NEAR subgraph developer experience

- NEAR subgraphs will be identifiable by their dataSources, which will have `kind = near`.
- NEAR dataSources may or may not refer to a specific NEAR contract (`source.account`). Contracts on NEAR are human-readable names (e.g. `mintbase.near`)
- NEAR dataSources will support the following trigger types:
  - Block - invoked on every block
  - Receipt - invoked on every action receipt execution to a given account
  - Function Calls - invoked on specific [FunctionCalls](https://nomicon.io/RuntimeSpec/FunctionCall.html) to a given contract (as part of a Receipt Execution Outcome).

_Receipt execution is when things **actually happen** on NEAR._

This does _not_ support potential other triggers:

- Transactions: transactions on NEAR only represent an initiation of events
- Non-function Actions: there are [actions beyond Function Calls](https://nomicon.io/RuntimeSpec/Actions.html), but Function Calls are the most interesting to developers

### Example Manifest definition

```yaml
specVersion: 0.0.X
schema:
  file: ./src/schema.graphql
dataSources:
  - kind: near
    name: NearExample
    network: mainnet
    source:
      account: app.good-morning.near
      startBlock: 1
    mapping:
      apiVersion: 0.0.X
      language: wasm/assemblyscript
      entities:
        - Thingy
        - Whatchamacall
      blockHandlers:
        - handler: handleNewBlock
      receiptHandlers:
        - handler: handleReceipt
      functionCallHandlers:
  	    - function: setPurpose
  	      handler: handleSetPurpose
      file: ./src/mapping.ts
```

At point of implementation each subgraph will only support one `kind` per subgraph.yaml (across datasources).

### NEAR AssemblyScript API

Additional NEAR-specific types will need to be introduced into Graph Node and graph-ts to support the new dataSources. These will need to tie back to the data available from the FirehoseBlockStream.

```typescript

class ExecutionOutcome {
      gasBurnt: u64,
      blockHash: Bytes,
      id: Bytes,
      logs: Array<string>,
      receiptIds: Array<Bytes>,
      tokensBurnt: BigInt,
      executorId: string,
  }

class ActionReceipt {
      predecessorId: string,
      receiverId: string,
      id: CryptoHash,
      signerId: string,
      gasPrice: BigInt,
      outputDataReceivers: Array<DataReceiver>,
      inputDataIds: Array<CryptoHash>,
      actions: Array<ActionValue>,
  }

class BlockHeader {
      height: u64,
      prevHeight: u64,// Always zero when version < V3
      epochId: Bytes,
      nextEpochId: Bytes,
      chunksIncluded: u64,
      hash: Bytes,
      prevHash: Bytes,
      timestampNanosec: u64,
      randomValue: Bytes,
      gasPrice: BigInt,
      totalSupply: BigInt,
      latestProtocolVersion: u32,
  }

class ChunkHeader {
      gasUsed: u64,
      gasLimit: u64,
      shardId: u64,
      chunkHash: Bytes,
      prevBlockHash: Bytes,
      balanceBurnt: BigInt,
  }

class Block {
      author: string,
      header: BlockHeader,
      chunks: Array<ChunkHeader>,
  }

class ReceiptWithOutcome {
      outcome: ExecutionOutcome,
      receipt: ActionReceipt,
      block: Block,
  }
```

[Reference](https://github.com/streamingfast/proto-near/blob/develop/sf/near/codec/v1/codec.proto)

The entity passed to the handler will depend on the handler type:

- Block handlers will receive a `Block`
- Receipt handlers will receive a `ReceiptWithOutcome`

#### Areas for further development:

- The NEAR indexer framework does not map logs to their corresponding function call, which may present difficulties for subgraph developers if an execution outcome contains multiple function calls.
- It is not simple to map back to a Transaction ID for a given receipt or execution outcome via the NEAR indexer framework, though this functionality has been requested.
- The initial integration will not allow subgraph developers to [make Contract Calls to a NEAR RPC node](https://docs.near.org/docs/api/rpc/contracts#call-a-contract-function). This pattern can significantly slow down indexing performance, but will be reviewed based on user requirements.
- The initial Firehose implementation does not populate a chain store, which means that the subgraph indexing status API will not have access to the chain head block, and will not be able to map block hashes in time travel queries to block numbers.

### GraphQL definition

No changes are anticipated to the `schema.graphQL`

# Backwards Compatibility

This functionality is all additive. However we may make some revisions to the `subgraph.yaml` specification as we introduce this new functionality.

# Dependencies

- NEAR Firehose implementation
- Multiblockchain refactor of Graph Node
- Firehose Graph Node integration

# Validation

This implementation will be validated and iterated on with stakeholders from the NEAR ecosystem:

- Core developers
- Dapp developers

# Rationale and Alternatives

We considered a custom integration of the NEAR indexer framework with Graph Node, but determined that the Firehose would be a more robust & scalable approach.

# Implementation

- [NEAR in Graph Node](https://github.com/graphprotocol/graph-node/tree/master/chain/near)
- [NEAR on StreamingFast](https://github.com/streamingfast/sf-near)
- [NEAR Deepmind indexer](https://github.com/streamingfast/near-dm-indexer)
- [NEARCore instrumentation](https://github.com/streamingfast/nearcore)
- [NEAR Protobuf definitions](https://github.com/streamingfast/proto-near)

# Copyright Waiver

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
