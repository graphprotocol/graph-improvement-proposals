---
GIP: "0080"
Title: Optimistic EBO
Authors: "mono <0xmono@defi.sucks>", "shaito <shaito@defi.sucks>"
Created: 2024-10-23
Updated: 2024-10-23
Stage: Draft
Discussions-To: 
---

# **Abstract**

This GIP proposes using an optimistic oracle to translate epochs from any chain to Arbitrum One. This approach replaces the current Epoch Block Oracle (EBO) design with a new optimistic version, called Optimistic EBO, which synchronizes the clock across indexed chains supported by The Graph.

# **Motivation**

The first version of the EBO, introduced in [GIP-0038](https://forum.thegraph.com/t/gip-0038-epoch-block-oracle/3323), prioritized simplicity but introduced a potential single point of failure. With the protocol's [migration to Arbitrum](https://forum.thegraph.com/t/gip-0043-activate-indexing-rewards-in-the-graph-network-on-arbitrum-one/3999) and the upcoming [Horizon update](https://forum.thegraph.com/t/gip-0066-introducing-graph-horizon-a-data-services-protocol/5989), which will enable the integration of arbitrary data services, The Graph should continue its progressive decentralization efforts by further decentralizing the Epoch Block Oracle.

# **Prior Art**

Several approaches have been proposed to improve the oracle process within The Graph protocol:

- [Submitting recent blocks for POIs](https://forum.thegraph.com/t/require-that-more-recent-pois-are-submitted-in-order-to-collect-indexing-rewards-multi-blockhain-pois/2500)
- [On-chain epoch oracle](https://hackmd.io/KHLuc4haR5uCnmpMu0Ce7Q)

> Before adopting the optimistic approach, expanding the number of EBO operators to 3-5 Council-authorized organizations was considered. However, the design proposed in this GIP provides greater decentralization, reducing the potential for coordinated corruption and the reliance on the Council as a single point of failure.
> 

## Current status of Epoch Block Oracle

The EBO defines the source of truth for the block number corresponding to a new epoch in each indexed chain. It fetches the block for each epoch on each chain and records it on-chain on the protocol chain (Arbitrum One). An epoch is a period determined by the [EpochManager](https://github.com/graphprotocol/contracts/blob/main/contracts/epochs/EpochManager.sol), which lasts approximately one day but is measured in blocks.

Indexers use the EBO subgraph to get the block number for each chain and use it to build the POI (Proof Of Indexing), which is published when closing allocations to claim rewards.

If a dispute arises, an indexer who publishes a POI for the wrong block may be disputed by a fisherman and penalized by the Arbitrator.

The EBO is composed of 3 parts:

- A Rust process that reads information from RPCs.
- A [Data Edge](https://github.com/graphprotocol/block-oracle/blob/main/packages/contracts/contracts/DataEdge.sol) contract where updates get posted (compressed and encoded).
- A [Subgraph](https://github.com/graphprotocol/block-oracle/tree/main/packages/subgraph) that indexes the Data Edge allowing indexers to easily craft their POI.

### Current design

The current system is run by a single Operator, which is the largest trust assumption of the EBO.

![EBO Designs (18).png](../assets//gip-0080/Optimistic%20EBO%20Designs.png)

### The Risks

- **Single Operator risk:** The current system is run by a single operator, which is the most prominent trust assumption of the EBO. There have been discussions around increasing the committee of validators/proposers.
- **Liveness risks:** If the owner fails to update the EBO, indexers won’t have a reference block to build their POIs, preventing them from closing their allocations or risk slashing if they register a POI for an incorrect block. The system must be redundant to mitigate this risk.
- **Fork risks:** There might be uncertainty around the correct fork choice when updating the EBO since it is not a fork oracle. For future verifiability efforts, it will likely be necessary to include blockhash in addition to the block number in the EBO’s output, making this even more important.

# **High-Level Description**

## Optimistic EBO Solution Considerations

The goal of this solution is to decentralize the EBO while keeping the structure open for future modifications.

Key improvements:

- **Optimistic operators:** Implementing an optimistic oracle will allow EBO operations to be open and permissionless, improving decentralization, redundancy, and scalability.
- **Timestamp comparison:** Instead of querying each RPC when detecting a new epoch, a delay is introduced to compare timestamps, reducing time sensitivity and improving the precision of fork selection.
- **Escalating bond mechanism**: The modularity of the optimistic oracle allows for an escalating bond dispute system, reducing Arbitrator intervention and enabling lower initial stakes without compromising security.
    - The Arbitrator multisig will act as the ultimate source of truth in the case of disputes in this initial phase, with potential future improvements.

### Optimistic Design

![image.png](../assets//gip-0080/Optimistic%20EBO.png)

### **Optimistic operators**

The optimistic approach allows off-chain agents to propose updates to the Optimistic EBO, which other participants can dispute by staking tokens or using alternative resolution mechanisms to align incentives and ensure accuracy.

For this, we will use [Prophet](https://github.com/defi-wonderland/prophet-modules/tree/dev), an optimistic, modular, and public-good oracle that abstracts the complexity of the proposal and dispute process while offering the flexibility to customize its functionality as needed. For more details see [Prophet documentation](https://docs.prophet.tech/content/modules/).
~~~~
The process would work as follows: 

- Any staked actor can propose an update for the EBO, which is published in a contract and remains open for a set period to allow for potential disputes.
- Any staked actor can also challenge the proposed update. If a dispute arises, the initial proposal is discarded, and a new one can be submitted.
- If a dispute occurs, the resolution mechanism is triggered, and the winner of the dispute will receive the counterparty’s bond as a reward.
- If no disputes are raised within the designated time window, or if the dispute is resolved in favor of the proposer, the update is considered valid.

Operators can update the block number of one chain at a time, allowing for ongoing disputes and resolutions for each updated chain. This improves the system's scalability.

The [Detailed Specification](https://www.notion.so/DRAFT_I-GIP-0XYZ-Optimistic-EBO-74a0b9df7008464e9dc1215c46920213?pvs=21) section will provide further details on how contracts and off-chain agents interact.

### Timestamp Comparison

Implementing a slight delay in timestamp comparison allows for safer EBO operations, significantly reducing risks related to RPC liveness and fork choice (finality) in exchange for a slight they delay in reward distribution:

- **Check the timestamp from the first block of the new epoch in the protocol chain (Arbitrum One).**
- **Check the block number for each indexed chain at that timestamp.**

The following methods can be used to prove that a specific block number corresponds to a claimed timestamp:

- Check the timestamps of two consecutive blocks $t_N$ and $t_{N+1}$. If the timestamp value from the protocol chain $t_0$ lives in the range $[t_N,t_{N+1}]$, then $N$ is the first block corresponding to the new epoch.
- Check a single block timestamp $t_N$ and an average block measure defined per chain $\Delta_i$. If the timestamp value from the protocol chain $t_0$ lives in the range $[t_N,t_N+\Delta_i]$, then $N$ is the first block corresponding to the new epoch.

### Escalating bond

To reduce the number of calls to The Graph's Arbitrator, we will implement an escalating bond system:

- Anyone can dispute while the dispute window is open and if staked. To dispute, you need to post a bond.
    - The resolution system is triggered once the participants have completed the maximum number of rounds.
    - If the disputant “overtakes” the Min Bond, the answer is considered incorrect until the proposer responds. The proposer must overtake the disputant's Bond to escalate the answer further. This process can be repeated up to N times.
- After each dispute, the original update gets deleted and can be answered again, even before resolution. This option makes the process quicker for the requester. This answer can be disputed using the same method as described above.
    - An honest proposer will be rewarded more for the won resolution than the original update.
    - Disputers are incentivized to post the new answer, as they have already computed the query.
- The payment from the requester will end up going to the proposer with the correct answer

The “overtake” mechanism improves resolution time and reduces Arbitrator calls, while the second answer (before resolution) improves the answering time to the request. 

![EBO Designs (6).png](../assets/gip-0080/Optimistic%20EBO%20Designs%20(1).png)

**Source of Truth**

If the escalating bond mechanism cannot resolve a dispute, the Arbitrator will be the final instance and ultimate source of truth, though this could change in the future.

There are other possible sources of truth for the protocol to verify the timestamp of a block on-chain:

- **Canonical bridges:** This is the most secure option since it shares the chain’s security, but on optimistic chains, it can introduce significant delays due to the mainnet withdrawal period
- **Third-party bridges:** Less secure but varying in their levels of trust.
    - **Light client-based bridges:** These use light clients to update block headers with timestamps, but no battle-tested solutions exist yet.
    - **Bridge aggregators:** Combine results from multiple bridges, like [Hashi](https://github.com/gnosis/hashi) or [LayerZero](https://docs.layerzero.network/v2/home/modular-security/security-stack-dvns) for EVMs, although it requires additional development
    - Other bridges require case-by-case analysis.

> Using an Arbitrator (a Council of qualified individuals selected by the governance) is the simplest way to launch the system at this early stage and manage all the chains with a single resolution mechanism. This approach allows for greater decentralization in the future, as the source of truth can be easily migrated.
> 

**Some considerations:**

- Implementing multiple resolution systems by chain or type (rollups, L1s, etc) adds complexity.
- The Council should upgrade the system to accommodate future improvements.
- Disputes should be managed per-chain to prevent a single dispute from impacting the entire system.

## **Economic Security considerations**

We recognize that economic security in The Graph presents complex challenges, which various teams are researching. The ideas discussed here are preliminary and may adapt to future protocol developments.

### Parameters

We suggest the following parameter values for the Escalation Bond system:

- **Bond threshold**: The amount of stake that triggers a call to the Arbitrator (virtual bond in the escalation system), set at 100K GRT.
- **Round**: The maximum number of rounds a dispute can escalate, set at 5.
- **Min Bond**: Minimum amount required for five rounds of escalations to reach the Bond threshold, set at 11111 GRT.
- **Dispute Window**: The maximum time a proposer has to respond to a dispute, set at 30 minutes.

These values are based on an economic security analysis that evaluates the potential gains from corruption across various attack scenarios. You can read the complete analysis [here](https://www.notion.so/Economic-security-considerations-1009a4c092c780b6afb1fa8bb8c266be?pvs=21).

**The Arbitrator has the authority to modify these parameters.**

# **Detailed Specification**

## Prophet modules

Interaction with Prophet requires the following modules that we will use/build to support the EBO:

- **RequestModule:** Used to request data in prophet. We will build the EBORequestModule which will allow users to create epoch-block requests to the oracle with the needed data for it. Compatible with partial requests (a subset of all the chains) and will contain the epoch number and the chain to fetch for that epoch.
- **ResponseModule:** Used by data providers to answer requests.
- **DisputeModule:** Provides the framework to automatically solve disputes before going into the resolution module. The [BondEscalationModule](https://github.com/defi-wonderland/prophet-modules/blob/dev/solidity/contracts/modules/dispute/BondEscalationModule.sol) manages disputes by allowing users to bond tokens to vote for the user that is telling the truth. The only modification we need to add is that only one propose per request can be solved through escalation. We can safely remove this.
- **ResolutionModule:** Used as the last resort to solve disputes and the ultimate source of truth. In our case we will be using the [ArbitratorModule](https://github.com/defi-wonderland/prophet-modules/blob/dev/solidity/contracts/modules/resolution/ArbitratorModule.sol) (Council) with a custom periphery contract so that The Graph's Arbitrator can resolve escalated disputes.
- **FinalityModule:** Executes code when a request is fully finished. We will be forking the [CallbackModule](https://github.com/defi-wonderland/prophet-modules/blob/dev/solidity/contracts/modules/finality/CallbackModule.sol) to add the events and functions necessary to be able to index it into the subgraph. This new FinalityModule will behave like the previous DataEdge contract. It will receive finalized requests for each chainId-block and emit events for them for the subgraph to index. Gated functions will also be included to allow The Graph's Arbitrator to amend any errors or publish data needed in case of emergency.
- **AccountingExtension:** Provides an interface to bond the user's tokens. We will support bonding in the Horizon Staking smart contract that is being developed by The Graph. We will be using and modifying the [BondEscalationAccounting](https://github.com/defi-wonderland/prophet-modules/blob/dev/solidity/contracts/extensions/BondEscalationAccounting.sol) module for this because it integrates fully with the BondEscalationModule on disputes.

### **Subgraph**

- **The EBO** subgraph will contain the indexed data generated by the FinalityModule. It will register new data published from prophet and also amendments pushed by the Council.

### **Periphery**

- **EBORequestCreator:** Will simplify and validate the creation of requests plus also allowing The Graph to add rewards to data providers in the future if they wish to. Checks that the requested epoch hasn't been requested before and that it has already started. The EBORequestCreator will also include the accepted chains. The Graph's Arbitrator will have the ability to add or remove them.
- **CouncilArbitrator:** Will be called by The Graph's Arbitrator with the result of a dispute to solve it so that it can be closed.

## Off-Chain Agents

Optimistic EBO allows any off-chain agent to interact with it by creating requests, proposing responses, or disputing responses in a permissionless manner.

### Architecture

The EBO Agent will be implemented as a single and standalone process that runs on a dedicated instance. The primary responsibility of this process is to poll events from the relevant on-chain smart contracts on the Protocol chain (Arbitrum) and respond to these events with specific actions tailored to the flow requirements. Additionally, the agent will interface with blockchains supported by The Graph and perform block computations for each epoch as necessary.
 

> When ready, we will suggest that the Indexers run the instances. There is an intrinsic incentive for them to post EBO updates as they use them to collect indexing rewards. Future changes to issuance and incentives could include other ways to incentivize EBO operation.
> 

### System limitations

The system will not handle requests that escalate to the Arbitrator for final resolution, as this isn’t automated by the off-chain agent and the Arbitrator must resolve the dispute manually. Additionally, once an epoch concludes, the system will not manage any pending requests.

![EBO-high-level.jpeg](../assets/gip-0080/Optimistic%20EBO%20high-level.jpeg)

The development of the EBO Agent will consist of two modules: 

- **DisputeModule:** The DisputeModule will handle all dispute-related activities and will operate in a fully reactive manner based on incoming events. It is important to note that all dispute logic will occur on the protocol chain. Hence, the DisputeModule will interact exclusively with the protocol chain, which in this case is Arbitrum.
- **BlockNumberModule:** The BlockNumberModule will be responsible solely for computing blocks based on timestamps on the blockchains supported by The Graph, including both EVM and non-EVM chains. For EVM chains, RPC calls will be utilized to extract the necessary data. For non-EVM chains, a third-party service.

# **Dependencies**

The [horizon staking contract](https://github.com/graphprotocol/contracts/pull/944), as described in GIP-0066, needs to be deployed to integrate with it.

# **Validation**

### **Testnet**

We propose a trial period on testnet for the new Optimistic EBO implementation, with the participation of volunteer indexers running the new instances. The goal is to test as many connected chains to the system as possible.

### **Internal Reviews**

A team of experienced developers from Wonderland, who have not been previously involved in the project, will carry out the following tasks:

- In-depth analysis of EBO requirements from a security perspective.
- Security review of each contract developed for the EBO use case.
- Implementation of necessary fixes based on relevant findings.
- Testing campaign including property-based testing with Echidna and an integration testing suite.

### **Audit**

The Graph Foundation and core devs will audit the code through an external security company. Based on the relevant findings, Wonderland will implement the necessary fixes.

# Who is Wonderland?

We’re a group of developers, researchers, and data scientists with one thing in common: we all love building cool sh*t. [DeFi sucks](https://defi.sucks/), but we are here to make it better.

Our mission is to discover, partner, and empower innovators in the creation of open, permissionless, and decentralized financial solutions. Our pledge is to stand by our partners, supporting them in every way we can.

We have partnered with some of the most successful and promising protocols in Web3 – including Aztec, Optimism, Everclear, Safe, MakerDAO, Gitcoin, Layer 3, Eigen Layer, Keep3r, Reflexer, and Yearn – to find solutions to complex engineering challenges and help them reach their full potential.

### What will we do?

The Wonderland team will implement the new version of EBO in close collaboration with The Graph's core development teams.

# **Copyright Waiver**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).