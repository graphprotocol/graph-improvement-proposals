---
GIP: <0055>
Title: EVM Firehose implementation
Authors: Adam Fuller, Alexandre Bourget
Created: 2023-06-15
Updated: 2023-06-15
Stage: Draft
---

# Abstract

This GIP proposes the development of a generally EVM-compatible Firehose, which will process, store and serve the more limited block data available from any EVM-compatible RPC.

# Motivation

This will unblock Firehose and Substreams usage for all EVM-compatible chains, which may also act as "lead generation" for full Firehose integrations.

# High Level Description

The EVM RPC is limited, but it does surface a lot of useful data for indexing:

- Block with transactions `getBlockByNumber`
- Receipts (including logs) `getReceiptByHash`
- Traces (if the [trace module](https://openethereum.github.io/JSONRPC-trace-module) is supported)

This information could be collated into a "Light" block, which has some (but not all) of the data which is made available in a "Full" Firehose block. We can then update the Ethereum Firehose Block schema to accommodate different "types" of blocks:

- Full blocks: the current Firehose block
- Light blocks: blocks sourced from an EVM-compatible client
- Light blocks with traces: blocks sourced from an EVM-compatible client with `trace` support

> Ideally Light blocks and Full blocks are compatible, i.e. Light block data is a subset of Full block data

It will then be possible to create a generic "EVM Firehose", which relies on the EVM RPC to populate Light blocks via polling. This process will need to handle re-orgs when populating blocks at the chain head. This process could be massively parallelised for finalised blocks.

While polling an EVM RPC is not performant relative to a traditional firehose integration, historical syncing is a "one-off", and Graph Node has demonstrated the ability to keep up with the chain head while fetching complete blocks from fast-moving EVM RPCs.

# Detailed Specification

TODO

# Backwards Compatibility

- Graph Node and other consuming services may need to be updated to be "schema aware" when it comes to Firehose blocks.
- The Firehose block schema may need to be updated to enable easy migration from Light to Full blocks.

# Dependencies

N/A

# Risks and Security Considerations

- Performance

# Rationale and Alternatives

We could continue to rely purely on client-level integrations.

# Copyright Waiver

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
