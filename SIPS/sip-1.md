---
sip: 1
title: SIP Purpose and Guidelines
status: Living
type: Meta
author: Martin Becze <mb@ethereum.org>, Hudson Jameson <hudson@ethereum.org>, et al.
created: 2022-12-21
---

## What is a SIP?

SIP stands for Seaport Improvement Proposal. A SIP is a design document providing information to the Seaport community, or describing a new feature for Seaport or its processes or environment. The SIP should provide a concise technical specification of the feature and a rationale for the feature. The SIP author is responsible for building consensus within the community and documenting dissenting opinions.

## SIP Rationale

We intend SIPs to be the primary mechanisms for proposing new features, for collecting community technical input on an issue, and for documenting the design decisions that have gone into Seaport. Because the SIPs are maintained as text files in a versioned repository, their revision history is the historical record of the feature proposal.

For Seaport implementers, SIPs are a convenient way to track the progress of their implementation. Ideally each implementation maintainer would list the SIPs that they have implemented. This will give end users a convenient way to know the current status of a given implementation or library.

## SIP Types

There are three types of SIP:

- A **Standards Track SIP** describes any change that affects most or all Seaport implementations, such as—a change to the protocol, proposed application standards/conventions, or any change or addition that affects the interoperability of applications using Seaport. Standards Track SIPs consist of three parts—a design document, an implementation, and (if warranted) an update to the Seaport codebase. Furthermore, Standards Track SIPs can be broken down into the following categories:

  - **Core**: improvements requiring a version upgrade, as well as changes that are not necessarily critical but may be relevant to “core dev” discussions.
  - **Networking**: includes improvements around [the P2P specification](./sip-4.md), as well as proposed improvements to network protocol specifications.
  - **Interface**: includes improvements around client API/RPC specifications and standards, and also certain language-level standards like method names and contract ABIs. Discussions and feedback should primarily occur before a SIP is submitted to the SIPs repository.
  - **ERC**: application-level standards and conventions, including contract standards, name registries, URI schemes, and library/package formats.

- A **Meta SIP** describes a process surrounding Seaport or proposes a change to (or an event in) a process. Process SIPs are like Standards Track SIPs but apply to areas other than the Seaport protocol itself. They may propose an implementation, but not to Seaport's codebase; they often require community consensus; unlike Informational SIPs, they are more than recommendations, and users are typically not free to ignore them. Examples include procedures, guidelines, changes to the decision-making process, and changes to the tools or environment used in Seaport development. Any meta-SIP is also considered a Process SIP.

- An **Informational SIP** describes an Seaport design issue, or provides general guidelines or information to the Seaport community, but does not propose a new feature. Informational SIPs do not necessarily represent Seaport community consensus or a recommendation, so users and implementers are free to ignore Informational SIPs or follow their advice.

It is highly recommended that a single SIP contain a single key proposal or new idea. The more focused the SIP, the more successful it tends to be. A change to one client doesn't require a SIP; a change that affects multiple clients, or defines a standard for multiple apps to use, does.

A SIP must meet certain minimum criteria. It must be a clear and complete description of the proposed enhancement. The enhancement must represent a net improvement. The proposed implementation, if applicable, must be solid and must not complicate the protocol unduly.

## SIP Work Flow

### Shepherding a SIP

Parties involved in the process are you, the champion or _SIP author_, the [_SIP editors_](#sip-editors), and the [Seaport community](https://github.com/ProjectOpenSea/seaport/discussions).

Before you begin writing a formal SIP, you should vet your idea. Ask the Seaport community first if an idea is original to avoid wasting time on something that will be rejected based on prior research. It is thus recommended to open a discussion thread on [the Seaport Discussions forum](https://github.com/ProjectOpenSea/seaport/discussions) to do this.

Once the idea has been vetted, your next responsibility will be to present (by means of a SIP) the idea to the reviewers and all interested parties, invite editors, developers, and the community to give feedback on the aforementioned channels. You should try and gauge whether the interest in your SIP is commensurate with both the work involved in implementing it and how many parties will have to conform to it. For example, the work required for implementing a Core SIP will be much greater than for a SRC and the SIP will need sufficient interest from the Seaport team and community. Negative community feedback will be taken into consideration and may prevent your SIP from moving past the Draft stage.

### Core SIPs

For Core SIPs, given that they require implementations to be considered **Final** (see "SIPs Process" below), you will need to either provide an implementation or convince core Seaport team members to implement your SIP.

The best way to get implementers to review your SIP is to present it in a [Seaport Discussions topic](https://github.com/ProjectOpenSea/seaport/discussions)

_In short, your role as the champion is to write the SIP using the style and format described below, shepherd the discussions in the appropriate forums, and build community consensus around the idea._

### SIP Process

The following is the standardization process for all SIPs in all tracks:

![SIP Status Diagram](../assets/sip-1/process.jpg)

**Idea** - An idea that is pre-draft. This is not tracked within the SIP Repository.

**Draft** - The first formally tracked stage of a SIP in development. A SIP is merged by a SIP Editor into the SIP repository when properly formatted.

**Review** - A SIP Author marks a SIP as ready for and requesting Peer Review.

**Last Call** - This is the final review window for a SIP before moving to `Final`. A SIP editor will assign `Last Call` status and set a review end date (`last-call-deadline`), typically 14 days later.

If this period results in necessary normative changes it will revert the SIP to `Review`.

**Final** - This SIP represents the final standard. A Final SIP exists in a state of finality and should only be updated to correct errata and add non-normative clarifications.

**Stagnant** - Any SIP in `Draft` or `Review` or `Last Call` if inactive for a period of 6 months or greater is moved to `Stagnant`. A SIP may be resurrected from this state by Authors or SIP Editors through moving it back to `Draft` or its earlier status. If not resurrected, a proposal may stay forever in this status.

> _SIP Authors are notified of any change to the status of their SIP_

**Withdrawn** - The SIP Author(s) have withdrawn the proposed SIP. This state has finality and can no longer be resurrected using this SIP number. If the idea is pursued at later date it is considered a new proposal.

**Living** - A special status for SIPs that are designed to be continually updated and not reach a state of finality. This includes most notably SIP-1 and SIP-2.

## What belongs in a successful SIP?

Each SIP should have the following parts:

- Preamble - RFC 822 style headers containing metadata about the SIP, including the SIP number, a short descriptive title (limited to a maximum of 44 characters), a description (limited to a maximum of 140 characters), and the author details. Irrespective of the category, the title and description should not include SIP number. See [below](./sip-1.md#sip-header-preamble) for details.
- Abstract - Abstract is a multi-sentence (short paragraph) technical summary. This should be a very terse and human-readable version of the specification section. Someone should be able to read only the abstract to get the gist of what this specification does.
- Motivation _(optional)_ - A motivation section is critical for SIPs that want to change the Seaport protocol. It should clearly explain why the existing protocol specification is inadequate to address the problem that the SIP solves. This section may be omitted if the motivation is evident.
- Specification - The technical specification should describe the syntax and semantics of any new feature. The specification should be detailed enough to allow competing, interoperable implementations for any of the current Seaport platforms.
- Rationale - The rationale fleshes out the specification by describing what motivated the design and why particular design decisions were made. It should describe alternate designs that were considered and related work, e.g. how the feature is supported in other languages. The rationale should discuss important objections or concerns raised during discussion around the SIP.
- Backwards Compatibility _(optional)_ - All SIPs that introduce backwards incompatibilities must include a section describing these incompatibilities and their consequences. The SIP must explain how the author proposes to deal with these incompatibilities. This section may be omitted if the proposal does not introduce any backwards incompatibilities, but this section must be included if backward incompatibilities exist.
- Test Cases _(optional)_ - Test cases for an implementation are mandatory for SIPs that are affecting consensus changes. Tests should either be inlined in the SIP as data (such as input/expected output pairs, or included in `../assets/sip-###/<filename>`. This section may be omitted for non-Core proposals.
- Reference Implementation _(optional)_ - An optional section that contains a reference/example implementation that people can use to assist in understanding or implementing this specification. This section may be omitted for all SIPs.
- Security Considerations - All SIPs must contain a section that discusses the security implications/considerations relevant to the proposed change. Include information that might be important for security discussions, surfaces risks and can be used throughout the life-cycle of the proposal. E.g. include security-relevant design decisions, concerns, important discussions, implementation-specific guidance and pitfalls, an outline of threats and risks and how they are being addressed. SIP submissions missing the "Security Considerations" section will be rejected. A SIP cannot proceed to status "Final" without a Security Considerations discussion deemed sufficient by the reviewers.
- Copyright Waiver - All SIPs must be in the public domain. The copyright waiver MUST link to the license file and use the following wording: `Copyright and related rights waived via [CC0](../LICENSE.md).`

## SIP Formats and Templates

SIPs should be written in [markdown](https://github.com/adam-p/markdown-here/wiki/Markdown-Cheatsheet) format. There is a [template](../sip-template.md) to follow.

## SIP Header Preamble

Each SIP must begin with an [RFC 822](https://www.ietf.org/rfc/rfc822.txt) style header preamble, preceded and followed by three hyphens (`---`). This header is also termed ["front matter" by Jekyll](https://jekyllrb.com/docs/front-matter/). The headers must appear in the following order.

`sip`: _SIP number_ (this is determined by the SIP editor)

`title`: _The SIP title is a few words, not a complete sentence_

`description`: _Description is one full (short) sentence_

`author`: _The list of the author's or authors' name(s) and/or username(s), or name(s) and email(s). Details are below._

`discussions-to`: _The url pointing to the official discussion thread_

`status`: _Draft, Review, Last Call, Final, Stagnant, Withdrawn, Living_

`last-call-deadline`: _The date last call period ends on_ (Optional field, only needed when status is `Last Call`)

`type`: _One of `Standards Track`, `Meta`, or `Informational`_

`category`: _One of `Core`, `Networking`, `Interface`, or `ERC`_ (Optional field, only needed for `Standards Track` SIPs)

`created`: _Date the SIP was created on_

`requires`: _SIP number(s)_ (Optional field)

`withdrawal-reason`: _A sentence explaining why the SIP was withdrawn._ (Optional field, only needed when status is `Withdrawn`)

Headers that permit lists must separate elements with commas.

Headers requiring dates will always do so in the format of ISO 8601 (yyyy-mm-dd).

### `author` header

The `author` header lists the names, email addresses or usernames of the authors/owners of the SIP. Those who prefer anonymity may use a username only, or a first name and a username. The format of the `author` header value must be:

> Random J. User &lt;address@dom.ain&gt;

or

> Random J. User (@username)

if the email address or GitHub username is included, and

> Random J. User

if the email address is not given.

It is not possible to use both an email and a GitHub username at the same time. If important to include both, one could include their name twice, once with the GitHub username, and once with the email.

At least one author must use a GitHub username, in order to get notified on change requests and have the capability to approve or reject them.

### `discussions-to` header

While an SIP is a draft, a `discussions-to` header will indicate the URL where the SIP is being discussed.

The preferred discussion URL is a topic on [Seaport Discussions](https://github.com/ProjectOpenSea/seaport/discussions). The URL cannot point to Github pull requests, any URL which is ephemeral, and any URL which can get locked over time (i.e. Reddit topics).

### `type` header

The `type` header specifies the type of SIP: Standards Track, Meta, or Informational. If the track is Standards please include the subcategory (core, networking, interface, or SRC).

### `category` header

The `category` header specifies the SIP's category. This is required for standards-track SIPs only.

### `created` header

The `created` header records the date that the SIP was assigned a number. Both headers should be in yyyy-mm-dd format, e.g. 2001-08-14.

### `requires` header

SIPs may have a `requires` header, indicating the SIP numbers that this SIP depends on. If such a dependency exists, this field is required.

A `requires` dependency is created when the current SIP cannot be understood or implemented without a concept or technical element from another SIP. Merely mentioning another SIP does not necessarily create such a dependency.

## Linking to External Resources

Other than the specific exceptions listed below, links to external resources **SHOULD NOT** be included. External resources may disappear, move, or change unexpectedly.

The process governing permitted external resources is described in [SIP-3](./sip-3.md).

### Seaport Repository

Links to the Seaport repository may be included using normal markdown syntax, such as:

```markdown
[Seaport.sol](https://github.com/ProjectOpenSea/seaport/blob/f402dac8b3faabdb8420d31d46759f47c9d74b7d/contracts/Seaport.sol)
```

Which renders as:

[Seaport.sol](https://github.com/ProjectOpenSea/seaport/blob/f402dac8b3faabdb8420d31d46759f47c9d74b7d/contracts/Seaport.sol)

Permitted Seaport GitHub URLs must anchor to a specific commit, and so must match this regular expression:

```regex
^https://github.com/ProjectOpenSea/seaport/blob/[0-9a-f]{40}/.*$
```

### Digital Object Identifier System

Links qualified with a Digital Object Identifier (DOI) may be included using the following syntax:

````markdown
This is a sentence with a footnote.[^1]

[^1]:
    ```csl-json
    {
      "type": "article",
      "id": 1,
      "author": [
        {
          "family": "Jameson",
          "given": "Hudson"
        }
      ],
      "DOI": "00.0000/a00000-000-0000-y",
      "title": "An Interesting Article",
      "original-date": {
        "date-parts": [
          [2022, 12, 31]
        ]
      },
      "URL": "https://sly-hub.invalid/00.0000/a00000-000-0000-y",
      "custom": {
        "additional-urls": [
          "https://example.com/an-interesting-article.pdf"
        ]
      }
    }
    ```
````

Which renders to:

This is a sentence with a footnote.[^1]

[^1]:
    ```csl-json
    {
      "type": "article",
      "id": 1,
      "author": [
        {
          "family": "Jameson",
          "given": "Hudson"
        }
      ],
      "DOI": "00.0000/a00000-000-0000-y",
      "title": "An Interesting Article",
      "original-date": {
        "date-parts": [
          [2022, 12, 31]
        ]
      },
      "URL": "https://sly-hub.invalid/00.0000/a00000-000-0000-y",
      "custom": {
        "additional-urls": [
          "https://example.com/an-interesting-article.pdf"
        ]
      }
    }
    ```

See the [Citation Style Language Schema](https://resource.citationstyles.org/schema/v1.0/input/json/csl-data.json) for the supported fields. In addition to passing validation against that schema, references must include a DOI and at least one URL.

The top-level URL field must resolve to a copy of the referenced document which can be viewed at zero cost. Values under `additional-urls` must also resolve to a copy of the referenced document, but may charge a fee.

## Linking to other SIPs

References to other SIPs should follow the format `SIP-N` where `N` is the SIP number you are referring to. Each SIP that is referenced in a SIP **MUST** be accompanied by a relative markdown link the first time it is referenced, and **MAY** be accompanied by a link on subsequent references. The link **MUST** always be done via relative paths so that the links work in this GitHub repository, forks of this repository, the main SIPs site, mirrors of the main SIP site, etc. For example, you would link to this SIP as `./sip-1.md`.

## Auxiliary Files

Images, diagrams and auxiliary files should be included in a subdirectory of the `assets` folder for that SIP as follows: `assets/sip-N` (where **N** is to be replaced with the SIP number). When linking to an image in the SIP, use relative links such as `../assets/sip-1/image.png`.

## Transferring SIP Ownership

It occasionally becomes necessary to transfer ownership of SIPs to a new champion. In general, we'd like to retain the original author as a co-author of the transferred SIP, but that's really up to the original author. A good reason to transfer ownership is because the original author no longer has the time or interest in updating it or following through with the SIP process, or has fallen off the face of the 'net (i.e. is unreachable or isn't responding to email). A bad reason to transfer ownership is because you don't agree with the direction of the SIP. We try to build consensus around an SIP, but if that's not possible, you can always submit a competing SIP.

If you are interested in assuming ownership of an SIP, send a message asking to take over, addressed to both the original author and the SIP editor. If the original author doesn't respond to the email in a timely manner, the SIP editor will make a unilateral decision (it's not like such decisions can't be reversed :)).

## SIP Editors

The current SIP editors are

- 0age (@0age)
- Kartik (@Slokh)

If you would like to become an SIP editor, please see [SIP-2](./sip-2.md).

## SIP Editor Responsibilities

For each new SIP that comes in, an editor does the following:

- Read the SIP to check if it is ready: sound and complete. The ideas must make technical sense, even if they don't seem likely to get to final status.
- The title should accurately describe the content.
- Check the SIP for language (spelling, grammar, sentence structure, etc.), markup (GitHub flavored Markdown), code style

If the SIP isn't ready, the editor will send it back to the author for revision, with specific instructions.

Once the SIP is ready for the repository, the SIP editor will:

- Assign a SIP number (generally the PR number, but the decision is with the editors)
- Merge the corresponding [pull request](https://github.com/ProjectOpenSea/SIPs/pulls)
- Send a message back to the SIP author with the next step.

Many SIPs are written and maintained by developers with write access to the Seaport codebase. The SIP editors monitor SIP changes, and correct any structure, grammar, spelling, or markup mistakes we see.

The editors don't pass judgment on SIPs. We merely do the administrative & editorial part.

## Style Guide

### Titles

The `title` field in the preamble:

- Should not include the word "standard" or any variation thereof; and
- Should not include the SIP's number.

### Descriptions

The `description` field in the preamble:

- Should not include the word "standard" or any variation thereof; and
- Should not include the SIP's number.

### SIP numbers

When referring to an SIP by number, it should be written in the hyphenated form `SIP-X` where `X` is the SIP's assigned number.

### RFC 2119 and RFC 8174

SIPs are encouraged to follow [RFC 2119](https://www.ietf.org/rfc/rfc2119.html) and [RFC 8174](https://www.ietf.org/rfc/rfc8174.html) for terminology and to insert the following at the beginning of the Specification section:

> The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119 and RFC 8174.

## History

Special thank you to authors Martin Becze, Hudson Jameson, et al. of the original [EIP-1](https://eips.ethereum.org/EIPS/eip-1) this document is based on.

This document was derived heavily from [Bitcoin's BIP-0001](https://github.com/bitcoin/bips) written by Amir Taaki which in turn was derived from [Python's PEP-0001](https://peps.python.org/). In many places text was simply copied and modified. Although the PEP-0001 text was written by Barry Warsaw, Jeremy Hylton, and David Goodger, they are not responsible for its use in the Seaport Improvement Process, and should not be bothered with technical questions specific to Seaport or the SIP. Please direct all comments to the SIP editors.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
