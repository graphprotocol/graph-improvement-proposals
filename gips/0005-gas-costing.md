---
GIP: 0005
Title: Gas costing for handlers
Authors: Leonardo Yvens<leo@edgeandnode.com> and Zac Burns <zac@edgeandnode.com>
Created: 2021-05-04
Stage: Candidate
Implementations: https://github.com/graphprotocol/graph-node/pull/2414
---

## Abstract

This GIP proposes that the computational cost of a handler be measured in units of gas and limited to a maximum cost. If a handler attempts to consume more gas than the maximum, the subgraph will fail with a deterministic error. This is necessary so that it is possible to prove that a subgraph cannot be indexed past the block which triggers such a handler.

## Motivation

Subgraph handlers that run for too long are an issue for operators, since they excessively consume resources, and also a issue for protocol security since they make the subgraph impossible to sync in a reasonable time, if it can be synced at all. Graph Node currently implements a timeout for handlers, which is useful to prevent excessive resource consumption, however timeouts are not deterministic. For protocol security, it is necessary to deterministically measure the cost of a handler. This GIP proposes gas costing for handlers, similar to the gas costing for transactions that exists in blockchain protocols.

## Detailed Specification

Gas is defined as the unit of execution cost. As a reference, one unit of gas should correspond to 0.1ns of execution time, making for 10 billion gas in a second. Close correlation with execution time is not a primary goal of gas costing, at least not in this first iteration. Still gas needs to have some correlation to execution time in order to be useful.

The maximum gas cost for a handler should be sufficient for any reasonable subgraph, and be limited by the cost a Fisherman or Arbitrator would be willing to incur in order to check an attestation. The proposed limit is 1000 seconds of execution time, which corresponds to 10 trillion gas.

The costs incurred by executing a handler can be categorized as either WASM execution costs or host function execution costs. WASM execution costs must be measured by injecting a callback at the block, see the documentation for this technique in the [pwasm-utils crate](https://docs.rs/pwasm-utils/0.17.1/pwasm_utils/fn.inject_gas_counter.html). For the gas costing of individual WASM instructions, refer to the implementation in [this commit](https://github.com/graphprotocol/graph-node/blob/30720a8f1fb074b71fe76729d4e0dc4ba3c3e955/runtime/wasm/src/gas_rules.rs).

The cost of most host functions is calculated as a base cost added to a cost per byte of input. To
account for host functions that have non-linear complexity, the input bytes are first combined
through a complexity function and then multiplied by the cost per byte. To calculate the size of the
input bytes, a  `gas_size_of` function exists for each data type that is an input to a host
function, see implementations of `GasSizeOf` in the reference implementation for details. Note that
`gas_size_of` is an approximation of the true in-memory size of the data, the actual size may vary
between platforms and implementations but still this approximation should be sufficient for gas
costing purposes.

#### Constants


##### Table 1:

| Constant               | Value (in gas)                       | Description                                                  |
| ---------------------- | ------------------------------------ | ------------------------------------------------------------ |
| GAS_PER_SECOND         | 10_000_000_000                       | Relates time and gas units.                                  |
| MAX_GAS_PER_HANDLER    | 3600 * GAS_PER_SECOND                | The limit is one hour worth of gas.                          |
| HOST_EXPORT_GAS        | 10_000                               | Cost of the callback that measures gas.                      |
| DEFAULT_BASE_COST      | 100_000                              | Base cost of most host functions.                            |
| DEFAULT_GAS_PER_BYTE   | 1_000                                | Equivalent to 10MB/s, so 36GB in the 1 hour limit.           |
| BIG_MATH_GAS_PER_BYTE  | 100                                  | Equivalent to 100MB/s.                                       |
| ETHEREUM_CALL          | 25_000_000_000                       | Gas cost of a contract call. Allows for 400 contract calls.  |
| CREATE_DATA_SOURCE     | MAX_GAS_PER_HANDLER / 100_000        | Cost of  `dataSource.create`.                                |
| LOG_GAS                | MAX_GAS_PER_HANDLER / 100_000        | Base cost of `log.log`.                                      |
| STORE_SET_BASE_COST    | MAX_GAS_PER_HANDLER / 250_000        | Base cost of `store.set`.                                    |
| STORE_SET_GAS_PER_BYTE | MAX_GAS_PER_HANDLER / 1_000_000_000  | Allows 1GB of entity data for `store.set`.                   |
| STORE_GET_BASE_COST    | MAX_GAS_PER_HANDLER / 10_000_000     | Base cost of `store.get`.                                    |
| STORE_GET_GAS_PER_BYTE | MAX_GAS_PER_HANDLER / 1_000_000_000  | Allows 10GB of entity data for `store.get`.                  |

#### Costs for host functions

This section specifies the costs attributed to each host function.

Most host functions have linear complexity on a single parameter `N` and are costed as:

````
DEFAULT_BASE_COST + gas_size_of(N)*DEFAULT_GAS_PER_BYTE
````

Host functions that differ from this are listed in the table below:

| Host function               | Cost                                                         |
| --------------------------- | ------------------------------------------------------------ |
| abort                       | DEFAULT_BASE_COST                                            |
| store.set(key, data)        | STORE_SET_BASE_COST + STORE_SET_GAS_PER_BYTE * (gas_size_of(key) + gas_size_of(data)) |
| store.remove(key)           | STORE_SET_BASE_COST + STORE_SET_GAS_PER_BYTE * gas_size_of(key) |
| store.get(key) -> data      | STORE_GET_BASE_COST + STORE_GET_GAS_PER_BYTE * (gas_size_of(key) + gas_size_of(data)) |
| ethereum.call               | ETHEREUM_CALL                                                |
| dataSource.create           | CREATE_DATA_SOURCE                                           |
| log.log(m)                  | LOG_GAS + DEFAULT_GAS_PER_BYTE * gas_size_of(m)              |
| bigInt.plus(x, y)           | DEFAULT_BASE_COST + BIG_MATH_GAS_PER_BYTE * max(gas_size_of(x), gas_size_of(y)) |
| bigInt.minus(x, y)          | DEFAULT_BASE_COST + BIG_MATH_GAS_PER_BYTE * max(gas_size_of(x), gas_size_of(y)) |
| bigInt.times(x, y)          | DEFAULT_BASE_COST + BIG_MATH_GAS_PER_BYTE * gas_size_of(x) * gas_size_of(y) |
| bigInt.divided_by(x, y)     | DEFAULT_BASE_COST + BIG_MATH_GAS_PER_BYTE * gas_size_of(x) * gas_size_of(y) |
| bigInt.mod(x, y)            | DEFAULT_BASE_COST + BIG_MATH_GAS_PER_BYTE * gas_size_of(x) * gas_size_of(y) |
| bigInt.pow(x, exp)          | DEFAULT_BASE_COST + BIG_MATH_GAS_PER_BYTE * gas_size_of(x) ^ exp |
| bigInt.bit_or(x, y)         | DEFAULT_BASE_COST + BIG_MATH_GAS_PER_BYTE * max(gas_size_of(x), gas_size_of(y)) |
| bigInt.bit_and(x, y)        | DEFAULT_BASE_COST + BIG_MATH_GAS_PER_BYTE * min(gas_size_of(x), gas_size_of(y)) |
| bigDecimal.plus(x, y)       | DEFAULT_BASE_COST + BIG_MATH_GAS_PER_BYTE * (gas_size_of(x) + gas_size_of(y)) |
| bigDecimal.minus(x, y)      | DEFAULT_BASE_COST + BIG_MATH_GAS_PER_BYTE * (gas_size_of(x) + gas_size_of(y)) |
| bigDecimal.times(x, y)      | DEFAULT_BASE_COST + BIG_MATH_GAS_PER_BYTE * gas_size_of(x) * gas_size_of(y) |
| bigDecimal.divided_by(x, y) | DEFAULT_BASE_COST + BIG_MATH_GAS_PER_BYTE * gas_size_of(x) * gas_size_of(y) |
| bigDecimal.equals(x, y)     | DEFAULT_BASE_COST + BIG_MATH_GAS_PER_BYTE * min(gas_size_of(x), gas_size_of(y)) |
