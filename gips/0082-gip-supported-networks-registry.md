---
GIP: 0082
Title: Establishment of the Supported Networks Registry
Authors: Paul Farnam <paul.farnam@pinax.network>
Created: 2024-12-10
Updated: 2024-12-10
Stage: Draft
Category: Process
Implementations: https://github.com/graphprotocol/networks-registry
---
> **Note:** This GIP is currently a Work in Progress (WiP). It has not yet been finalized and is open for preliminary review and feedback.

## Abstract

This proposal introduces the Supported Networks Registry, a centralized repository for managing and documenting networks within The Graph ecosystem. It aims to streamline the addition and updating of network information, ensuring consistency, reliability, and accessibility for developers and stakeholders.

## Motivation

As The Graph ecosystem expands to support an increasing number of blockchain networks, managing network data consistently becomes more challenging. The absence of a unified, standardized method for documenting and updating network information can lead to inconsistencies and confusion among developers, indexers, and community members.

Establishing a Supported Networks Registry allows The Graph to centralize and standardize network data maintenance. This initiative simplifies chain integration, enhances indexer operations, and provides a seamless experience for developers working across diverse blockchain infrastructures. The registry ensures that The Graph’s ecosystem can scale efficiently, with transparent governance and clear processes for integrating new networks.

## High-Level Description

The Supported Networks Registry is a GitHub-hosted repository containing JSON-based definitions of all supported networks. Each network’s configuration includes fields for unique identifiers, RPC endpoints, explorer URLs, and other relevant metadata. A predefined schema ensures consistency and data integrity, while automated validation scripts prevent malformed or conflicting entries.

Contributors can add or update networks by submitting pull requests. Automated checks verify schema compliance and the uniqueness of identifiers. Continuous Integration/Continuous Deployment (CI/CD) pipelines handle merging, versioning, and publishing updated registry files. Integration libraries in various languages (e.g., TypeScript, Go, Rust) provide developers and indexers with easy access to reliable, up-to-date network data.

## Detailed Specification

**Repository Structure:**

- `schemas/`: Contains the schema for the registry.
- `registry/`: Contains the network JSON files to be edited.
- `public/`: Contains all generated registry versions (auto-generated; not to be edited manually).
- `src/`: Contains scripts to validate network JSON files and generate the resulting registry JSON.

**JSON Schema Requirements:**

Required Fields:
- `networkName`: Unique canonical name of the network.
- `chainId`: A unique identifier (e.g., EVM chain ID) ensuring no duplicates.
- `rpcEndpoints`: A list of valid RPC endpoints.
- `explorerUrls`: A list of block explorer URLs.

Optional Fields:
- `logoUrl`, `supportedFeatures`, aliases, and additional metadata.

**Validation Layers:**

1. Schema Validation: Ensures JSON files conform to the registry schema.
2. Semantic Validation: Checks the uniqueness of `chainId`, verifies URLs, and cross-references external data sources.
3. Registry Validation: Confirms the consolidated registry data’s consistency and completeness.

**Workflow for Updating the Registry:**

1. Add or modify the relevant network JSON file in the `registry/` directory.
2. Optionally, validate changes locally using `bun validate` (see the repository’s README for setup instructions).
3. Optionally, format files using `bun format`.
4. Increment the patch version in `package.json`.
5. Submit a pull request. This will trigger validation checks and indicate any issues with the new definitions.

**Versioning & Publication:**

The registry uses semantic versioning (MAJOR.MINOR.PATCH).
- **Patch Updates**: Non-breaking additions or corrections.
- **Minor Updates**: Backward-compatible enhancements.
- **Major Updates**: Potentially breaking schema changes, requiring broader community agreement.

Upon merging, automated pipelines generate versioned registry files (e.g., `TheGraphNetworksRegistry_v1_2_3.json`) and maintain a master registry file (`TheGraphNetworksRegistry.json`). Integration libraries reference these files to ensure applications access the latest validated network data.

**Governance & Integration:**

- Changes to schemas, governance rules, or critical functionality follow The Graph Improvement Proposal (GIP) process.
- The community-driven approach allows all pull requests and discussions to be public, enabling contributors and core developers to review and comment.
- Integration libraries are maintained in the [networks-registry repository](https://github.com/graphprotocol/networks-registry) for easy access.

## Backward Compatibility

This proposal is backward-compatible. Existing applications and services can continue using current methods until they choose to adopt the new registry. Versioning ensures that older files remain accessible, enabling a smooth transition.

## Dependencies

This proposal depends on the availability and maintenance of the following:

- External chain ID standards (e.g., Ethereum chain lists) for validation.
- Participation from The Graph community, indexers, and developers to maintain accurate and complete metadata.
- Collaboration with stakeholders to converge on standardized naming conventions and metadata fields.

### Risks and Security Considerations

The key risks of this proposal involve invalid or malicious data submissions that could affect applications relying on the registry. To address these concerns, we have implemented the following safeguards:
-  Strict schema and semantic validation checks are in place to ensure that all submitted data meets defined standards.
-	 Before any changes are accepted, every contribution undergoes review and approval. In addition, CI/CD pipelines, version control, and public revision histories provide transparency and maintain trust.
- Should a newly introduced change cause issues, we can fall back to a stable older version.

## Validation

The registry will be initially tested in a testnet environment and validated through:

- Local schema checks and semantic validation.
- Pull request-based reviews by the community and core contributors.
- Continuous integration tests verifying every proposed change.

## Copyright Waiver

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).