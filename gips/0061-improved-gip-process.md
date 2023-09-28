---
GIP: 0061
Title: Improvements to the GIP process
Authors: Pablo Carranza Vélez <pablo@edgeandnode.com>
Created: 2023-09-28
Updated: 2023-09-28
Stage: Draft
Discussions-To: TODO
Category: Process
Depends-On: GIP-0001
Implementations: <This pull request>
---

# Abstract

This GIP proposes changes to the GIP process, designed to simplify the process and clarify the steps to get a GIP implemented and deployed.

It proposes removing the so far unused concept of GRP (Graph Request for Proposals), and reducing the number of possible stages for a GIP. It also introduces clearer instructions on how to get a GIP audited, tested and discussed by the community and the Council.

# Motivation

Since the introduction of the GIP process in GIP-0001, many proposals have been created, published and accepted by the Council. Most of them have been developed by core developers, and we believe the current process can be improved to make it clearer for non-core-developer contributors to propose changes to the protocol, and to understand the steps that are needed to get a proposal accepted and deployed.

This GIP therefore aims to simplify the process and bring some clarity so that it's easier for people who want to improve The Graph to bring their ideas to the community, and implement and deploy them if they are accepted.

# High-Level Description

GIP-0001 has been restructed to better capture the real process that has been occurring when people proposed changes to the protocol. Some small updates have been made, like replacing mentions of Radicle with the GitHub repository, or updating the list of Editors. The full changeset can be seen in the same Pull Request where this GIP is introduced to the repository.

GIP-0000 has also been edited slightly to reflect the changes proposed here.

The main changes proposed to GIP-0001 and the process are described below.

## Clarify the role of the Editors

We believe it's important that the GIP process is as permissionless as possible. As such, we think it's important to clarify that Editors should not be able, through inactivity, to stop a GIP from advancing through the process. We add some text to the Editors section to clarify this and make it explicit.

## Remove the GRP category

To this date, no "Graph Request for Proposals" have been submitted. To simplify the process, we propose removing this category.

## Simplified Stages

The Strawman and Proposal stages have been removed and merged into the Draft stage. In practice, proposals evolve from a simple abstract to a full blown spec, and tend to change throughout the process, and it's hard to draw the line between different stages. Keeping the stage updated in the GIP takes time, so we believe it's simpler to just differentiate between a proposal that is in progress ("Draft"), one that is ready for acceptance/deployment ("Candidate"), and one that has been concluded ("Accepted/Deferred/Withdrawn/Living").

## Self-assigning GIP numbers

In practice, most GIP authors were self-assigning GIP numbers, to not depend on Editor availability as this could leave a GIP without a number for some time (which makes it harder to reference a GIP in public discussion). Here we propose that authors self-assign a GIP number following clear rules, with Editors being able to re-number if needed to avoid conflicts.

## A clearer process after publishing a GIP

The process to get a GIP implemented and accepted by the Council needed some clarification. We had added more detail about the quality assurance process for the different kind of changes, with additional detail for smart contracts changes.

The changes also make it explicit that implementation of a GIP can happen in parallel to the governance process.

We also introduce some mechanisms for authors to advance proposals through the process, using online forms (currently being set up on TypeForm) to:
- Request a speaking slot in one of the periodic community calls
- Request an audit slot
- Request help from core devs to implement a GIP
- Ask the Council to discuss a GIP for acceptance

Hopefully these forms open a direct way for authors to advance a GIP through the process, and the additional instructions provide more clarity as to what needs to happen before a GIP is ready for deployment.

# Risks and Security Considerations

The use of forms to advance proposals through the process, or request audits and other activities, involves a few risks:
- Spam: if the volume of invalid/nonsensical form submissions is too high, it could make it impractical for the Council and core dev contributors filtering them for legitimate submissions to be noticed. If this happens, further changes to the process would be needed to add more validation before sending a submission.
- Single point of failure: if the forms provider goes down, or for whatever reason someone is unable to get the form submission delivered, this could impede them from advancing the proposal or putting forward their request. To avoid the single point of failure, authors are encouraged to use the forum, Discord, or any other public channel to communicate with core devs or The Foundation if they suspect the message was not received.

# Copyright Waiver

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
