# Seaport Improvement Proposals (SIPs)

**Before you initiate a pull request**, please read the [SIP-1][sip-1] process document. Ideas should be thoroughly discussed on [SIP Discussions][sip-discussions].

This repository tracks ongoing improvements to Seaport. It contains:

- The [SIP repository][sip-repository], tracking standards for Seaport clients and applications
- The [process document][sip-1] that governs how standards are published here

For help _implementing_ a SIP, please visit [Seaport Discussions][seaport-discussions].

If you would like to become a SIP Editor, please see [SIP-2][sip-2].

_Special thanks to [EIPs](https://github.com/ethereum/eips) from which this repository was derived._

## Mission

The goal of the SIP project is to document standardized protocols for Seaport clients and applications and to document them in a high-quality and implementable way.

## Preferred Citation Format

The canonical URL for a SIP that has achieved draft status at any point is at <https://github.com/ProjectOpenSea/SIPs>. For example, the canonical URL for SIP-1 is <https://github.com/ProjectOpenSea/SIPs/blob/main/SIPS/sip-1.md>.

Consider any document not published at <https://github.com/ProjectOpenSea/SIPs> as a working paper. Additionally, consider published SIPs with a status of "draft", "review", or "last call" to be incomplete drafts, and note that their specification is likely to be subject to change.

## SIP status terms

- **Idea** - An idea that is pre-draft. This is not tracked within the SIP Repository.
- **Draft** - The first formally tracked stage of a SIP in development. A SIP is merged by a SIP Editor into the SIP repository when properly formatted.
- **Review** - A SIP Author marks a SIP as ready for and requesting Peer Review.
- **Last Call** - This is the final review window for a SIP before moving to FINAL. A SIP editor will assign Last Call status and set a review end date (`last-call-deadline`), typically 14 days later. If this period results in necessary normative changes it will revert the SIP to Review.
- **Final** - This SIP represents the final standard. A Final SIP exists in a state of finality and should only be updated to correct errata and add non-normative clarifications.
- **Stagnant** - Any SIP in Draft or Review if inactive for a period of 6 months or greater is moved to Stagnant. A SIP may be resurrected from this state by Authors or SIP Editors through moving it back to Draft.
- **Withdrawn** - The SIP Author(s) have withdrawn the proposed SIP. This state has finality and can no longer be resurrected using this SIP number. If the idea is pursued at later date it is considered a new proposal.
- **Living** - A special status for SIPs that are designed to be continually updated and not reach a state of finality. This includes most notably SIP-1 and SIP-2.

## SIP Types

SIPs are separated into a number of types, and each has its own list of SIPs.

### Standard Track

Describes any change that affects most or all Seaport implementations, such as a change to the protocol, a change in order validity rules, proposed application standards/conventions, or any change or addition that affects the interoperability of applications using Seaport. Furthermore Standard SIPs can be broken down into the following categories.

#### Core

Improvements requiring a new version, as well as changes that are not necessarily consensus critical but may be relevant to “core dev” discussions.

#### Networking

Includes improvements around how Seaport information or orders are communicated.

#### Interface

Includes improvements around client API/RPC specifications and standards, and also certain language-level standards like method names and contract ABIs. Discussions and feedback should primarily occur before a SIP is submitted to the SIPs repository.

#### SRC

Application-level standards and conventions, including contract standards, name registries, URI schemes, and library/package formats.

#### Meta

Describes a process surrounding Seaport or proposes a change to (or an event in) a process. Process SIPs are like Standards Track SIPs but apply to areas other than the Seaport protocol itself. They may propose an implementation, but not to Seaport's codebase; they often require community consensus; unlike Informational SIPs, they are more than recommendations, and users are typically not free to ignore them. Examples include procedures, guidelines, changes to the decision-making process, and changes to the tools or environment used in Seaport development. Any meta-SIP is also considered a Process SIP.

#### Informational

Describes a Seaport design issue, or provides general guidelines or information to the Seaport community, but does not propose a new feature. Informational SIPs do not necessarily represent Seaport community consensus or a recommendation, so users and implementers are free to ignore Informational SIPs or follow their advice.

## Validation and Automerging

All pull requests in this repository must pass automated checks before they can be automatically merged:

- Spelling is enforced with [CodeSpell](https://github.com/codespell-project/codespell)[^1]
- False positives sometimes occur. When this happens, please submit a PR editing [.codespell-whitelist](https://github.com/ProjectOpenSea/SIPs/blob/main/config/.codespell-whitelist).
- Markdown best practices are checked using [markdownlint](https://github.com/DavidAnson/markdownlint)[^1]

[^1]: https://github.com/ProjectOpenSea/SIPs/blob/main/.github/workflows/ci.yml

[sip-repository]: https://github.com/ProjectOpenSea/SIPs
[sip-discussions]: https://github.com/ProjectOpenSea/SIPs/discussions
[seaport-discussions]: https://github.com/ProjectOpenSea/seaport/discussions
[sip-1]: https://github.com/ProjectOpenSea/SIPs/blob/main/SIPS/sip-1.md
[sip-2]: https://github.com/ProjectOpenSea/SIPs/blob/main/SIPS/sip-2.md
