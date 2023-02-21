---
GIP: 0036
Title: Juno indexing for Graph node
Authors: Marc
Created: 2022-22-06
Stage: Draft
Discussions-To: https://forum.thegraph.com/t/gip-0036-juno-indexing-for-graph-node/3490
---

Abstract
========

This proposal describes the Juno integration into Graph Node. Juno is a smart-contract-based blockchain powered by the Cosmos ecosystem. Developers can design and deploy interoperable, cross-chain smart contracts using different languages like Rust and Go. Juno is the neutral [home of CosmWasm 1](https://cosmwasm.com/) smart contracts and the InterWasm DAO. The Juno blockchain is made up of free, public, and open-source software.

Motivation
==================================================================================================

Integrating Juno will allow Graph users to index the biggest smart-contract network in the Cosmos Ecosystem, which will set a foothold on the Cosmos ecosystem integration and will keep expanding The Graph's market and make its multi-chain vision a reality.

With a growing number of blockchains, dApps and builders, The Graph will have a vital role to play in the Cosmos ecosystem.

Prior Art
================================================================================================

The following GIP relies on the "Tendermint indexing for Graph node" [GIP 5](https://forum.thegraph.com/t/gip-0022-tendermint-indexing-for-graph-node/2815).

Juno is built on top of Tendermint and Cosmos SDK, therefore all the work done previously on Tendermint will be used as the foundation for this new integration.

In this document, we will be referring to the work described in [GIP-0022 5](https://forum.thegraph.com/t/gip-0022-tendermint-indexing-for-graph-node/2815) as `Cosmos Integration`.

High Level Description
==========================================================================================================================

Data extraction on Juno will take place inside a Juno-based instrumented network node with augmented Tendermint library. The data extracted will be passed through the Cosmos specific Firehose to reach the Graph Node.

Subgraph developers will be able to specify the dataSource kind introduced in the Cosmos integration (`cosmos`) and a new network name (eg. `juno-1`) for their subgraphs, with specific triggers relevant to Cosmos-based applications and [CosmWasm 1](https://docs.cosmwasm.com/docs/1.0/) smart contracts.

Juno is built on top of the Cosmos SDK. Because of this, all protobuf definitions added in the [Cosmos integration 5](https://forum.thegraph.com/t/gip-0022-tendermint-indexing-for-graph-node/2815) for major components through the whole data model are still relevant, and since Juno relies heavily on [CosmWasm 1](https://docs.cosmwasm.com/docs/1.0/), only the new messages defined by [CosmWasm 1](https://docs.cosmwasm.com/docs/1.0/) will need to be integrated. Practically, this means subgraph developers will keep working with the same high level types, and will be able to decode new [CosmWasm 1](https://docs.cosmwasm.com/docs/1.0/) messages using the provided decoding libraries.

Detailed Description
======================================================================================================================

### Data extraction from a Juno node

Juno uses Tendermint as a consensus mechanism. Therefore, the data extraction mechanism used is the same as in the [Cosmos integration 5](https://forum.thegraph.com/t/gip-0022-tendermint-indexing-for-graph-node/2815), where an instrumented node runs an augmented version of the Tendermint library that allows for data extraction.

This modification has to be done for all the required versions of the Juno node.

### Firehose data

Further in the process, the nodes output would be passed into a dedicated Cosmos Firehose stack instance to pass it into the graph node. At this point, all the Firehose components are identical to the ones used in the Cosmos integration.

This reutilization of components is possible due to the fact that all protobuf definitions are the same down to the `messages` in [`TxBody`](https://github.com/cosmos/cosmos-sdk/blob/6a9b8247f65188723979e41a7d6944fbb168d0f2/proto/cosmos/tx/v1beta1/tx.proto#L104). Therefore, Firehose can decode, store and filter data the same way.

```
message TxBody {
  // messages is a list of messages to be executed. The required signers of
  // those messages define the number and order of elements in AuthInfo's
  // signer_infos and Tx's signatures. Each required signer address is added to
  // the list only the first time it occurs.
  // By convention, the first required signer (usually from the first message)
  // is referred to as the primary signer and pays the fee for the whole
  // transaction.
  repeated google.protobuf.Any messages = 1;
	...
}

```

The [`google.protobuf.Any`](https://github.com/protocolbuffers/protobuf/blob/c4ddd84918b4b58c03c60f44e0e7a2c6df5846d5/src/google/protobuf/any.proto#L125) type used for the `messages` attribute ensures different types of messages can be passed under the same transaction. This type is passed as `bytes` along a URL string that acts as a globally unique identifier for and resolves to that message's type. The value of the URL is used further up at the subgraph level to identify and decode messages.

```
message Any {
  string type_url = 1;
  bytes value = 2;
}

```

### CosmWasm specific messages

Juno comes with a specific [set of new messages](https://github.com/CosmWasm/wasmd/tree/main/proto/cosmwasm/wasm/v1), most of them defined in the CosmWasm module integrated in the chain. These messages are the core of the Juno integration, and represent the different actions that can be taken within the network.

By identifying and decoding the message contents, subgraph developers will be able to index data directly related with the core functionality of the chain and its components, such as contract executions or IBC transfers.

### Example: CosmWasm contract

A message with type_url: `/juno.wasm.v1beta1.MsgExecuteContract` and value [MsgExecuteContract](https://github.com/CosmWasm/wasmd/blob/8ca55b78fcfdca72d813b509ae890ea7e84567ed/proto/cosmwasm/wasm/v1/tx.proto#L74) will trigger the execution of a CosmWasm smart contract.

```
// MsgExecuteContract submits the given message data to a smart contract
message MsgExecuteContract {
  // Sender is the that actor that signed the messages
  string sender = 1;
  // Contract is the address of the smart contract
  string contract = 2;
  // Msg json encoded message to be passed to the contract
  bytes msg = 3 [ (gogoproto.casttype) = "RawContractMessage" ];
  // Funds coins that are transferred to the contract on execution
  repeated cosmos.base.v1beta1.Coin funds = 5 [
    (gogoproto.nullable) = false,
    (gogoproto.castrepeated) = "github.com/cosmos/cosmos-sdk/types.Coins"
  ];
}

```

CosmWasm smart contracts also emit events to allow proper indexing of the transactions from smart contracts. Every call to instantiate or execute the contract will be tagged with the info on the contract that was executed and who executed it. The module is always wasm, there is also an action tag which is auto-added by the Cosmos SDK and has a value of either [store-code, instantiate or execute 1](https://docs.cosmwasm.com/docs/1.0/smart-contracts/contract-semantics/) depending on which message was sent.

```
{
    "Type": "message",
    "Attr": [
        {
            "key": "module",
            "value": "wasm"
        },
        {
            "key": "action",
            "value": "execute"
        },
        {
            "key": "signer",
            "value": "cosmos1zm074khx32hqy20hlshlsd423n07pwlu9cpt37"
        },
        {
            "key": "_contract_address",
            "value": "cosmos14hj2tavq8fpesdwxxcu44rty3hh90vhujrvcmstl4zr3txmfvw9s4hmalr"
        }
    ]
}

```

### Graph Node Juno integration

Eventually the Juno Firehose gRPC endpoint will be consumed by Graph Node, which in the exact same way as the Cosmos integration, will use the dedicated Cosmos FirehoseBlockStream, Cosmos specific types, and filtering for triggers, ultimately reaching subgraph code.

Based on the previous section, to properly process transaction data, the subgraph runtime environment has to be extended with a way to decode protobuf-encoded information inside messages value payload.

These will be provided in the form of Typescript decoding libraries that will automatically generate boilerplate code and the required handling methods to successfully decode messages based on their protobuf definitions. These libraries will be made available to subgraph users, enabling them to decode messages directly in the subgraph mapping functions.

The three types of handlers are kept the same as in the Cosmos Hub integration. These are:

-   blockHandlers: run on every new block appended to the chain. The handler will receive a full block and all its data containing, among other things, all the events and transactions.
-   eventHandlers: run on every event, except if there is an event type specified in the handler definition, in that case only events of that type are processed. A light representation of the block the event is contained in is also passed onto the mapping in order to have context of the event within the chain.
-   transactionHandlers: run for every transaction executed. The mapping is provided with all the relevant data related to the transaction and a light representation of the block it is contained in, that can be used to acquire context of the transaction within a block and within the chain.

Event and Transaction handlers are a way to process meaningful data from the chain without the need to process a whole block. The data processed by them can also be found in the block handlers, since events and transactions are also part of a block, but removes the need of processing unnecessary data.

### Operational requirements

In order to index Juno subgraphs, Indexers will therefore need to run the following:

A Juno node with augmented Tendermint library.

Cosmos Firehose Component(s).

A Graph Node with Firehose endpoint configured.

### GraphQL definition

No changes are anticipated to the schema.graphql

### Areas for further development

This integration is heavily focused on contract specific messages. In further development, the existing handlers could be extended to include some more helpful triggers for subgraph authors.

Backwards Compatibility
=============================================================================================================================

Similarly to Cosmos, this functionality is all additive, and because of both Firehose related changes and proto decoding in the subgraphs, it may require some changes to the subgraph.yaml specification.

Dependencies
=======================================================================================================

A Multiblockchain refactor - Tendermint library augmentation - Cosmos Firehose implementation - Firehose Graph Node integration

Copyright Waiver
===============================================================================================================

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).