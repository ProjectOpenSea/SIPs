---
sip: 2
title: SIP Editor Handbook
description: Handy reference for SIP editors and those who want to become one
author: Pooja Ranjan (@poojaranjan), Pandapip1 (@Pandapip1)
discussions-to:
status: Living
type: Informational
created: 2022-12-21
requires: 1
---

## Abstract

A Seaport Improvement Proposal (SIP) is a design document providing information to the Seaport community, or describing a new feature for Seaport or its processes or environment. The SIP standardization process is a mechanism for proposing new features, for collecting community technical input on an issue, and for documenting the design decisions that have gone into Seaport. Because improvement proposals are key components of Seaport, it is important that they are well reviewed before reaching `Final` status. SIPs are stored in text files in a versioned repository which is monitored by the SIP editors.

This SIP describes the recommended process for becoming a SIP editor.

## Specification

The key words “MUST”, “MUST NOT”, “REQUIRED”, “SHALL”, “SHALL NOT”, “SHOULD”, “SHOULD NOT”, “RECOMMENDED”, “MAY”, and “OPTIONAL” in this document are to be interpreted as described in RFC 2119.

### Application and Onboarding Process

Anyone having a good understanding of the SIP standardization and network upgrade process, intermediate level experience on the core and/or application side of the Seaport blockchain, and willingness to contribute to the process management MAY apply to become a SIP editor. Potential SIP editors SHOULD have the following skills:

- Good communication skills
- Ability to handle contentious discourse
- 1-5 spare hours per week

The best available resource to understand the SIP process is [SIP-1](./sip-1.md). Anyone desirous of becoming a SIP editor MUST understand this document. Afterwards, participating in the SIP process by commenting on and suggesting improvements to PRs and issues will familiarize the procedure, and is RECOMMENDED. The contributions of newer editors SHALL be monitored by other SIP editors.

Anyone meeting the above requirements MAY make a pull request adding themselves as a SIP editor and adding themselves to the editor list in [SIP-1](./sip-1.md). If the existing SIP editors approve, the author SHALL become a full SIP editor. This SHALL notify the editor of relevant new proposals submitted in the SIPs repository, and they SHALL review and merge pull requests.

### Special Merging Rules for this SIP

This SIP MUST have the same rules regarding changes as [SIP-1](./sip-1.md).

## Rationale

- "6 months" was chosen as the cutoff for denoting `Stagnant` SIPs terminally-`Stagnant` arbitrarily.
- This SIP requires special merging rules for the same reason [SIP-1](./sip-1.md) does.

## Acknowledgements

Special thank you to authors Pooja Ranjan and Pandapip1 of the original [EIP-5069](https://eips.ethereum.org/EIPS/eip-5069) this document is derived from.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
