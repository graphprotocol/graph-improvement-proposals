---
GIP: 0033
Title: Osmosis indexing for Graph node
Authors: Marc
Created: 6-22-2022
Updated: 
Stage: Draft
Discussions-To: https://forum.thegraph.com/t/gip-0033-osmosis-indexing-for-graph-node/3396
Category: "Protocol Logic"
Depends-On:
Implementations: 
Audits: 
---

Abstract
--------

This proposal describes the Osmosis integration into Graph Node. Osmosis is a decentralized, cross-chain automated market maker (AMM) protocol build on top of the Cosmos SDK. It allows users to create custom liquidity pools and trade IBC enabled tokens. A recent [proposal 1](https://www.mintscan.io/osmosis/proposals/205) has also been passed to integrate a bridge with the Ethereum chain. The Osmosis blockchain is made up of free, public, and open-source software.

* * * * *

Motivation
-----------------------------------------------------------------------------------------------------

Integrating Osmosis will allow Graph users to directly work with one of the biggest AMM in the Cosmos Ecosystem, empowering Osmosis users who don't currently have an indexing framework. At the same time, it will set a foothold on the Cosmos ecosystem integration and will keep expanding The Graph's market and make its multi-chain vision a reality.

With a growing number of blockchains, dApps and builders, The Graph will have a vital role to play in the Cosmos ecosystem.

* * * * *

Prior Art
---------------------------------------------------------------------------------------------------

The following GIP relies on the "Tendermint indexing for Graph node" [GIP 3](https://forum.thegraph.com/t/gip-0022-tendermint-indexing-for-graph-node/2815).

Osmosis is built on top of Tendermint and Cosmos SDK, therefore all the work done previously on Tendermint will be used as the foundation for this new integration.

In this document, we will be referring to the work described in [GIP-0022 3](https://forum.thegraph.com/t/gip-0022-tendermint-indexing-for-graph-node/2815) as `Cosmos Integration` .

* * * * *

* * * * *

High Level Description
-----------------------------------------------------------------------------------------------------------------------------

Osmosis data extraction will take place inside an Osmosis-based instrumented network node with augmented Tendermint library. The data extracted will be passed through the Cosmos specific Firehose to reach the Graph Node.

Subgraph developers will be able to specify the dataSource kind introduced in the Cosmos integration (`cosmos`) and a new network name (eg. `osmosis-1`) for their subgraphs, with specific triggers relevant to Cosmos-based applications.

Osmosis is built on top of the Cosmos SDK. Because of this, all protobuf definitions added in the [Cosmos integration 3](https://forum.thegraph.com/t/gip-0022-tendermint-indexing-for-graph-node/2815) for major components through the whole data model are still relevant, and only the new messages defined by Osmosis will need to be integrated. Practically, this means subgraph developers will keep working with the same high level types, and will be able to decode new Osmosis messages using the provided decoding libraries.

Osmosis uses [CosmWasm](https://docs.cosmwasm.com/docs/1.0/) as its chain agnostic smart contract platform. It integrates with it as a Cosmos SDK module, also defining its own additional message types. These types will also need to be integrated in order to index smart contract data.

* * * * *

Detailed Description
-------------------------------------------------------------------------------------------------------------------------

### Data extraction from an Osmosis node

Osmosis uses Tendermint as a consensus mechanism. Therefore, the data extraction mechanism used is the same as in the Cosmos integration, where an instrumented node runs an augmented version of the Tendermint library that allows for data extraction.

This modification has to be done for all the crucial (and latest) versions of the Osmosis node.

### Firehose data

Further in the process, the nodes output would be passed into a dedicated Cosmos Firehose stack instance to pass it into the graph node. At this point, all the Firehose components are identical to the ones used in the Cosmos integration.

This reutilization of components is possible due to the fact that all protobuf definitions are the same down to the [`messages` 1](https://github.com/cosmos/cosmos-sdk/blob/6a9b8247f65188723979e41a7d6944fbb168d0f2/proto/cosmos/tx/v1beta1/tx.proto#L104) in `TxBody`. Therefore, Firehose can decode, store and filter data the same way.

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

The [`google.protobuf.Any`](https://github.com/protocolbuffers/protobuf/blob/c4ddd84918b4b58c03c60f44e0e7a2c6df5846d5/src/google/protobuf/any.proto#L125) type used for the `messages` attribute ensures different types of messages can be passed under the same transaction. This type is passed as `bytes` along a URL string that acts as a globally unique identifier for and resolves to that message's type. The value of the URL is used further up at the Graph Node level to identify and decode messages.

```
message Any {
  string type_url = 1;
  bytes value = 2;
}

```

### Osmosis specific messages - CosmWasm contract example

Osmosis defines a set of new messages as part of its [Cosmos SDK modules](https://github.com/osmosis-labs/osmosis/tree/main/proto/osmosis). These messages are the core of the Osmosis integration, and represent the different actions that can be taken within the network.

By identifying and decoding the message contents, subgraph developers will be able to index data directly related with the core functionality of the chain and its components, such as liquidity pools, delegations or superfluid staking.

Example #1: Superfluid Staking

Superfluid staking is one of the main features of Osmosis, and allows users to both stake and provide liquidity without making any general network tradeoffs, i.e. security for liquidity.

In this case, when a user performs a superfluid delegation to lock its tokens, the [superfluid module](https://github.com/osmosis-labs/osmosis/tree/main/x/superfluid) would issue a transaction containing a message with `type_url`: `/osmosis.superfluid.MsgLockAndSuperfluidDelegate` and `value` of [`MsgLockAndSuperfluidDelegate`](https://github.com/osmosis-labs/osmosis/blob/db6e756f66122e0896ddbdd328efbebac6ba97c9/proto/osmosis/superfluid/tx.proto#L62).

```
// MsgLockAndSuperfluidDelegate locks coins with the unbonding period duration,
// and then does a superfluid lock from the newly created lockup, to the
// specified validator addr.
message MsgLockAndSuperfluidDelegate {
  string sender = 1 [ (gogoproto.moretags) = "yaml:\"sender\"" ];
  repeated cosmos.base.v1beta1.Coin coins = 2 [
    (gogoproto.nullable) = false,
    (gogoproto.castrepeated) = "github.com/cosmos/cosmos-sdk/types.Coins"
  ];
  string val_addr = 3;
}

```

Example #2: GAMM (Generalized Automated Market Maker)

When a user joins a liquidity pool in Osmosis, a message would be issued from the [GAMM module](https://github.com/osmosis-labs/osmosis/tree/main/x/gamm), it would have `type_url`: `/osmosis.gamm.v1beta1.MsgJoinPool` and `value` of [`MsgJoinPool`](https://github.com/osmosis-labs/osmosis/blob/db6e756f66122e0896ddbdd328efbebac6ba97c9/proto/osmosis/gamm/v1beta1/tx.proto#L28).

```
message MsgJoinPool {
  string sender = 1 [ (gogoproto.moretags) = "yaml:\"sender\"" ];
  uint64 poolId = 2 [ (gogoproto.moretags) = "yaml:\"pool_id\"" ];
  string shareOutAmount = 3 [
    (gogoproto.customtype) = "github.com/cosmos/cosmos-sdk/types.Int",
    (gogoproto.moretags) = "yaml:\"pool_amount_out\"",
    (gogoproto.nullable) = false
  ];
  repeated cosmos.base.v1beta1.Coin tokenInMaxs = 4 [
    (gogoproto.moretags) = "yaml:\"token_in_max_amounts\"",
    (gogoproto.nullable) = false
  ];
}

```

Another big core part of this piece of work is the integration of CosmWasm smart contracts. The process will be somehow similar to the one done for Osmosis, as CosmWasm integrates with it as a Cosmos SDK module called [`wasm`](https://github.com/CosmWasm/wasmd/tree/main/x/wasm). Therefore defining its own set of [messages](https://github.com/CosmWasm/wasmd/tree/main/proto/cosmwasm/wasm/v1) and events. The work done for the Cosmos integration is still valid for all types of events so the focus will turn into the messages part.

Example #3: CosmWasm contract

A message with `type_url`: `/osmosis.wasm.v1beta1.MsgExecuteContract` and value [`MsgExecuteContract`](https://github.com/CosmWasm/wasmd/blob/8ca55b78fcfdca72d813b509ae890ea7e84567ed/proto/cosmwasm/wasm/v1/tx.proto#L74) will trigger the execution of a CosmWasm smart contract.

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

CosmWasm smart contracts also emit events to allow proper indexing of the transactions from smart contracts. Every call to instantiate or execute the contract will be tagged with the info on the contract that was executed and who executed it. The module is always `wasm`, there is also an `action` tag which is auto-added by the Cosmos SDK and has a value of either [`store-code`, `instantiate` or `execute`](https://docs.cosmwasm.com/docs/1.0/smart-contracts/contract-semantics) depending on which message was sent.

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

### Graph Node Osmosis integration

Eventually the Osmosis Firehose gRPC endpoint will be consumed by Graph Node, which in an exact same way as with the Cosmos integration, will use the dedicated Cosmos FirehoseBlockStream, Cosmos specific types, and filtering for triggers, ultimately reaching subgraph code.

Based on the previous section, to properly process transaction data, the subgraph runtime environment has to be extended with a way to decode protobuf bytes encoded information inside messages `value` payload.

These will be provided in form of Typescript decoding libraries that will automatically generate boilerplate code and the required handling methods to successfully decode messages based on their protobuf definitions. These libraries will be made available to subgraph users, enabling them to decode messages directly in the subgraph mapping functions.

The three types of handlers are kept the same as in the Cosmos Hub integration. These are:

-   `blockHandlers`: run on every new block appended to the chain. The handler will receive a full block and all its data containing, among other things, all the events and transactions.
-   `eventHandlers`: run on every event, except if there is an event type specified in the handler definition, in that case only events of that type are processed. A light representation of the block the event is contained in is also passed onto the mapping in order to have context of the event within the chain.
-   `transactionHandlers`: run for every transaction executed. The mapping is provided with all the relevant data related to the transaction and a light representation of the block it is contained in, that can be used to acquire context of the transaction within a block and within the chain.

Event and Transaction handlers are a way to process meaningful data from the chain without the need to process a whole block. The data processed by them can also be found in the block handlers, since events and transactions are also part of a block, but removes the need of processing unnecessary data.

### Operational requirements

In order to index Osmosis subgraphs, Indexers will therefore need to run the following:

-   A Osmosis node with augmented Tendermint library.
-   Cosmos Firehose Component(s).
-   A Graph Node with Firehose endpoint configured.

### GraphQL definition

No changes are anticipated to the `schema.graphql`

### Areas for further development

-   CosmWasm has just been integrated into Osmosis and smart contracts still need to be deployed in the mainnet. When this happens there could be changes to the specification that would force updates in the Osmosis integration.
-   This integration is heavily focused on messages. In further development, the existing handlers could be extended to include some more helpful triggers for subgraph authors.

Backwards Compatibility
--------------------------------------------------------------------------------------------------------------------------------

![:curling_stone:](https://emoji.discourse-cdn.com/apple/curling_stone.png?v=12 ":curling_stone:") Similarly to Cosmos, this functionality is all additive, and because of both Firehose related changes and proto decoding in the subgraphs, it may require some changes to the `subgraph.yaml` specification.

* * * * *

Dependencies
----------------------------------------------------------------------------------------------------------

A Multiblockchain refactor - Tendermint library augmentation - Cosmos Firehose implementation - Firehose Graph Node integration

* * * * *

Copyright Waiver
------------------------------------------------------------------------------------------------------------------

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).