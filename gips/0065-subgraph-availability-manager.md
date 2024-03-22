---
GIP: 0065
Title: Subgraph Availability Manager
Authors: Miguel de Elias <miguel@edgeandnode.com>
Created: 2024-01-10
Stage: Draft
Discussions-To: https://forum.thegraph.com/t/gip-0065-subgraph-availability-manager/5501
Category: Protocol Interfaces
Implementations: https://github.com/graphprotocol/contracts/pull/882
Audits: https://github.com/graphprotocol/contracts/blob/main/packages/contracts/audits/OpenZeppelin/2024-02-subgraph-availability-manager-and-minimum-allocation-duration-removal.pdf
---

# Abstract

This GIP proposes increasing the decentralization in The Graph by introducing a new smart contract `SubgraphAvailabilityManager`.

# Motivation

The Subgraph Availability Oracle (SAO) plays a crucial role in The Graph's ecosystem. Its main function is to verify the availability and validity of subgraph files. Currently there is a single instance of the SAO in operation executing open-source code accessible at [The Graph's Subgraph Oracle repository](https://github.com/graphprotocol/subgraph-oracle). This SAO instance runs at five-minute intervals, performing a series of checks to assess the state of subgraph files. In instances where the SAO identifies unavailability or invalidity in a subgraph's files, it sends a transaction to the `RewardsManager` contract to deny indexing rewards for that subgraph. By default, subgraphs are presumed to be available, and typically, no initial action or vote by the SAO is necessary when a new subgraph is added to the network.

The proposed change aims to enhance the decentralization of the protocol by introducing five separate oracles operators and requiring a minimum of three votes among them before any decisive action is taken. All oracles operators will be bound by the **Subgraph Availability Oracle Operator Charter** included in this GIP which outlines their roles, responsibilities, and operational guidelines.

# High-Level Description

The new smart contract `SubgraphAvailabilityManager` will be exclusively deployed to the Arbitrum One network and will function as a mediator between five distinct oracle operators and the `RewardsManager` contract. It will feature a vote aggregation system and enforce an execution threshold requiring three votes to alter the state of a subgraph deployment within the `RewardsManager` contract. Additionally, it will incorporate a `voteTimeLimit` timeframe to monitor current votes, after which they expire and become invalid.

It's important to note that this system operates effectively only if a minimum of three oracles are honest and running. The system will be able to report subgraph availability whenever three or more of the oracle instances are running correctly. According to the Charter guidelines, all oracles are expected to operate in compliance with established standards. In the event that any oracles are found not to be performing up to these standards, measures will be taken to replace them, ensuring the system's integrity and continuous effective operation.

The `SubgraphAvailabilityManager` contract will be governed by the council. While the contract itself will not be upgradeable, a new contract can be deployed, and the council can set the `RewardsManager` oracle address to the new deployment. The council will have the authority to add or remove oracles operators and update the `voteTimeLimit` timeframe.

# Detailed Specification

The number of oracle operators will be strictly enforced as five. Upon contract initialization, five valid accounts must be provided, and each oracle must maintain their index position, crucial for passing it as a parameter when casting votes.

Vote aggregation will be separately tracked for positive or negative votes using the following data structures:

```solidity
mapping(uint256 currentNonce => mapping(bytes32 subgraphDeploymentID => uint256[NUM_ORACLES] timestamps)) public lastDenyVote;
mapping(uint256 currentNonce => mapping(bytes32 subgraphDeploymentID => uint256[NUM_ORACLES] timestamps)) public lastAllowVote;
```

The initial mapping key corresponds to a public variable `currentNonce`. This nonce serves as a mechanism to invalidate all votes whenever a configuration is altered—be it updating the oracle operators list or modifying the `voteTimeLimit` parameter. The inner mapping stores the `subgraphDeploymentId` against the timestamp of each oracle's vote, using an array denoting the oracle index.

The `executionThreshold` will be set to three during contract construction. Whenever an oracle casts a vote, the `SubgraphAvailabilityManager` contract will query the corresponding vote aggregation storage to verify if enough votes have been registered. Upon the third vote a transaction to `RewardsManager` is triggered to make the subgraph rewards state change.

A `voteTimeLimit` parameter will be used to invalidate old votes, initially set to ten minutes. This duration is slightly longer than the current subgraph availability oracle configuration, which runs every five minutes, allowing time for oracles to cast their votes and accounting for potential delays in execution. This minimizes the need for frequent transactions. When comparing oracle votes, the `SubgraphAvailabilityManager` will use the `voteTimeLimit` to ensure the last recorded vote occurred within this timeframe. Votes older than the limit will not be considered valid, necessitating a new vote from the oracle if required. It's crucial to note that any adjustments to `voteTimeLimit` must always exceed the oracle's execution period to prevent the premature invalidation of votes before all oracles have participated. However, this timeframe should be kept as brief as possible, since an excessively long `voteTimeLimit` undermines its intended purpose. The council can modify this timeframe if desired but it must be set with consideration to the oracle’s iteration period to avoid repeated voting or expired votes.

# Rationale and Alternatives

The presented proposal serves as a temporary solution; we will continue our efforts to explore additional methods for further decentralization or even the complete elimination of the subgraph availability oracle. We believe this implementation provides a short-term fix that significantly improves decentralization compared to the previous 1-of-1 approach.

---

*This section contains the body of the Subgraph Availability Oracle Operator Charter.*

# Subgraph Availability Oracle Operator Charter

## 1. Role of the Subgraph Availability Oracle.

The primary role of the Subgraph Availability Oracle in The Graph Network is to ensure the availability and validity of subgraph manifests. Oracles are responsible for verifying subgraphs availability and conducting validity checks.

Oracles operate as part of the `SubgraphAvailabilityManager` smart contract, contributing to the decentralized decision-making process by casting votes regarding the availability and validity of subgraph files.

## 2. Appointment and Governance of Subgraph Availability Oracles Operators

Oracle operators are appointed by The Graph Council, which retains the authority to add or remove oracle operators, ensuring compliance with the guidelines and responsibilities outlined in this charter. For this initial phase, the Oracle operators will be Core Developers Teams, although future extensions to include other community members are not discounted.

The number of oracle operators is strictly limited to five.

## 3. Responsibility and Integrity

Operators must ensure their oracles remain operational, acknowledging that while occasional outages are part of normal operations, efforts to minimize these are crucial. Operators should communicate their infrastructure choices with one another to prevent risks associated with common dependencies, thereby enhancing the network's overall resilience. Active participation in a group chat to discuss operational challenges and coordinate on mitigating risks is required. Operators who are unresponsive or consistently underperform may be recommended for replacement by the technical advisory board or fellow operators to the Graph Council.

## 4. Subgraph Availability Oracle Software and Versioning

Subgraph Availability Oracles operators must use the software available at [The Graph's Subgraph Oracle repository](https://github.com/graphprotocol/subgraph-oracle). This ensures uniformity and accuracy in their assessments across the network. A specific commit or release version from this repository will be designated as the required version for all oracles. This version is subject to change over time as updates and improvements are made.

Oracle operators are responsible for reaching consensus on the specific commit and configuration to be used. To facilitate this process, a group chat channel will be established for operators to communicate and reach consensus. Once agreed upon, operators must align their operations with these standards by posting a transaction with the details to a `DataEdge` smart contract within no more than three business days. A dedicated subgraph will track these declarations, flagging any deviations for prompt resolution to maintain network consistency.

**Note:** In case a security fix is needed, operators should treat this with the highest priority. Oracles are required to be updated as soon as possible and no later than 24 hours after consensus.

The transaction to the `DataEdge` should follow the format:

```json
{
    "commit-hash": "XXXXX",
    "config": {
        "ipfs-concurrency": "4",
        "ipfs-timeout": "10000",
        "min-signal": "100",
        "period": "300",
        "grace_period": "0",
        "supported-networks": "mainnet,gnosis,xdai,eip155:1,eip155:100,avalanche,eip155:43114,arbitrum-one,eip155:42161,celo,eip155:42220,fantom,eip155:250,matic,eip155:137",
        "supported_data_source_kinds": "ethereum,ethereum/contract,file/ipfs,substreams,file/arweave",
        "subgraph": "https://api.thegraph.com/subgraphs/name/graphprotocol/graph-network-arbitrum",
        "subgraph_availability_manager_contract": "CONTRACT_ADDRESS",
        "oracle_index": "ORACLE_INDEX",
    }
}

```

It is strongly recommended that Oracle operators diversify their sources for `IPFS_NODE_URL` and `RPC_URL` configurations. Utilizing varied sources for these critical components not only enhances the robustness and reliability of the network but also mitigates potential risks associated with single points of failure.

## 5. Testnet

In addition to their operations on Arbitrum One, all oracle operators are required to run a corresponding version on the Arbitrum Sepolia testnet. This setup is necessary to create a testing environment that closely mirrors the production environment, facilitating thorough testing and verification before deployment to production. Part of this requirement involves sending a transaction with the commit hash and configuration to the `DataEdge` smart contract on the testnet, following the format mentioned in the section above.

## 6. Security

Oracle operators are expected to follow best practices in security to protect their systems from unauthorized access and potential vulnerabilities.

## 7. Conflict Resolution and Recourse

Given that all oracle operators are mandated to run the same software version, uniformity in voting outcomes is expected. Discrepancies in oracle votes are therefore an area of concern and warrant investigation. In instances where voting differences occur or concerns are raised to the foundation or the core developers, the Technical Advisory Board and the Subgraph Availability Oracles operators will appoint an investigator to look into the issue. This investigation aims to determine whether the discrepancy was due to an honest failure or a result of malicious intent. The investigators decision will be final.

If an oracle operator is found to be acting maliciously, the Graph Council will nominate a new operator to replace them. In scenarios where an oracle consistently deviates from the expected voting pattern due to honest errors, The Graph Council will evaluate its performance. Persistent non-compliance with the charter's guidelines, even if unintentional, may lead to the replacement of the oracle operator to maintain the integrity and reliability of the network.

*This concludes the body of the Subgraph Availability Oracle Operator charter.*

---

# Copyright Waiver

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).