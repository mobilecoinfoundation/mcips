```
  MCIP: ?
  Title: MCIP Process
  Authors: koe <ukoe@protonmail.com>
  Status: Draft (NOT FOR USE)
  Type: Process
  Created: ?
  Replaces: [MCIP-0000](0000-mobilecoin-rfcs.md)
```

## Table Of Contents

- [Abstract](#Abstract)
- [Motivation](#Motivation)
- [MCIP workflow](#MCIP-workflow)
    - [Merging Draft MCIPs](#merging-draft-mcips)
    - [MCIP Editors](#MCIP-Editors)
    - [MCIP Editor Responsibilities](#MCIP-Editor-Responsibilities)
    - [Transferring MCIP Ownership](#Transferring-MCIP-Ownership)
- [MCIP format and structure](#MCIP-format-and-structure)
    - [MCIP header preamble](#MCIP-header-preamble)
    - [Auxiliary Files](#Auxiliary-Files)
- [MCIP types](#MCIP-types)
- [MCIP status field](#MCIP-status-field)
    - [Progression to Active status](#Progression-to-Active-status)
    - [Rationale](#Rationale)
- [MCIP README](#MCIP-README)
- [MCIP licensing](#MCIP-licensing)
- [Rationale](#Rationale)
- [Backward compatibility](#Backward-compatibility)
- [Reference implementation](#Reference-implementation)
- [References](#References)



## Abstract

A MobileCoin Improvement Proposal (MCIP) is a design document providing information to the MobileCoin community, or describing a new feature for MobileCoin or its processes or environment. The MCIP should provide a concise technical specification of the feature and a rationale for the feature.

We intend MCIPs to be the primary mechanisms for proposing new features, for collecting community input on an issue, and for documenting the design decisions that have gone into MobileCoin. The MCIP author is responsible for building consensus within the community and documenting dissenting opinions.

Because the MCIPs are maintained as text files in a versioned repository, their revision history is the historical record of the feature proposal.

This MCIP is adapted from [BIP-0002](https://github.com/bitcoin/bips/blob/master/bip-0002.mediawiki), with modifications.



## Motivation

For MobileCoin to be successful in the long run, it is important for design decisions to be well-documented, and for the community and ecosystem to have the power to both contribute new ideas and provide feedback on ideas before they are implemented.

This MCIP specifies a concrete process for proposing new ideas, getting feedback on those ideas, and documenting those ideas clearly for future reference (whether or not they are actually implemented). It replaces [MCIP-0000](0000-mobilecoin-rfcs.md), which was adapted from the [Rust](https://github.com/rust-lang/rfcs) RFC process.

The Rust process is optimized for an ecosystem where development and 'protocol evolution' are owned and driven by specific development teams. This MCIP is optimized for a more decentralized ecosystem with heterogenous participants, motivations, and efforts, where protocol evolution and discussion is not owned by any particular group.



## MCIP workflow

The MCIP process begins with a new idea for MobileCoin. Each potential MCIP must have a champion (or champions) -- someone who writes the MCIP using the style and format described below, shepherds the discussions in the appropriate forums, and attempts to build community consensus around the idea. The MCIP champion(s) (a.k.a. author(s)) should first attempt to ascertain whether the idea is MCIP-able.

Small enhancements or patches to a particular piece of software often don't require standardization between multiple projects. These don't need an MCIP and should be injected into the relevant project-specific development workflow.

The author should see if an idea has been considered before, and if so, what issues arose in its progression. After investigating past work (note that past ideas may not be discoverable via any search engine - human interactions may be necessary), they should collect community and developer feedback on their idea. Importantly, the author should only proceed if they are confident their idea is applicable to the entire community. Just because an idea sounds good to the author does not mean it will work for most people in most areas where MobileCoin is used.

Once a champion has fleshed out the core concepts of their proposal, they should submit a Draft MCIP as a pull request to the [MCIPs git repository](https://github.com/mobilecoinfoundation/mcips). This pull request should then be referenced in early public discussions around the proposal.

While in 'pull request' form, the author should solicit feedback from the community, and address requests for formatting and quality improvements. Where possible, important discussions should either be summarized or linked-to in comments on the pull request.

This Draft must be written in MCIP style as described below, and named with an alias such as "MCIP-johndoe-infiniteMobileCoins" until an editor has assigned it an MCIP number (authors MUST NOT self-assign MCIP numbers).

It is highly recommended that a single MCIP contain a single key proposal or new idea. If in doubt, split your MCIP into several well-focused ones.


### Merging Draft MCIPs

The MCIP process does not seek to gatekeep proposals. Any Draft can be merged once its author(s) and an MCIP editor decide it can be merged. Merging a Draft does not mean the proposal is in its final form (i.e. a form that can be recommended for widespread use). Intead, a merged Draft is a proposal from the author that may be iterated on, and whose merits may be continuously assessed by the community and project developers.

Once an MCIP Draft is ready, an MCIP editor will:

1. Assign an MCIP number in the pull request.
1. Merge the pull request.
1. List the MCIP in the [MCIPs README](https://github.com/mobilecoinfoundation/mcips/README.md)

After a Draft has been merged, the MCIP author may submit new pull requests to update their Draft, until the MCIP reaches Proposed or Active status.

The MCIP editors will not unreasonably prevent a Draft or Draft update from being merged. Reasons for rejecting pull requests include:

- duplication of effort
- disregard for formatting rules
- being too unfocused or too broad
- being technically unsound
- not providing proper motivation or addressing backwards compatibility
- not in keeping with the MobileCoin philosophy (a private cryptocurrency for everyone)

For an MCIP to be accepted, it must meet certain minimum criteria.

1. It must be a clear and complete description of the proposed enhancement.
1. The enhancement must represent a net improvement.
1. The proposed implementation, if applicable, must be solid and must not complicate the protocol unduly.

If an editor rejects an MCIP, they must clearly enumerate the reasons. Rejections that seem unfair or unwarranted may be appealed with the following series of escalations:

1. Request that the MCIP editor group review the rejection. The author is expected to address the stated reasons for rejection.
1. Petition the MobileCoin Foundation Technical Committee for a review of the rejection and a review of the MCIP editor group's integrity/efficacy.
1. The author's last resort is hosting their proposal outside of the MCIP repository.


### MCIP Editors

The current MCIP editors are:

- [[[TODO]]] (contact at: [[[TODO]]])
- [[[TODO]]] (contact at: [[[TODO]]])


### MCIP Editor Responsibilities

The MCIP editors are appointed by the MobileCoin Foundation's Technical Committee to manage the MCIP repository and MCIP submission process. They must have access to all major arenas where MCIPs may be discussed.

When a Draft MCIP is submitted as a pull request to the MCIP repository, an editor reads the MCIP and checks the following:

- The MCIP should be ready: sound and complete. The ideas must make technical sense, even if they don't seem likely to be accepted.
- The title should accurately describe the content.
- The author must have requested comments in the MCF discord channel \#mcip ([invite link here](https://discord.gg/EUg9cW3PHR)).
- Motivation and backward compatibility (when applicable) must be addressed.
- The header must be appropriately constructed. Draft MCIPs should have the status `Draft (NOT FOR USE)`.

If the MCIP isn't ready, an editor will request that the author revise their pull request, with specific instructions.

The MCIP editors are intended to fulfill administrative and editorial responsibilities. The MCIP editors monitor MCIP changes, and update MCIP headers as appropriate.


### Transferring MCIP Ownership

It may become necessary to transfer ownership of an MCIP to a new champion. We prefer retaining the original author as a co-author of the transferred MCIP, but that's really up to the original author. A good reason to transfer ownership is because the original author no longer has the time or interest in updating it or following through with the MCIP process, or has become unreachable. A bad reason to transfer ownership is because you don't agree with the direction of the MCIP. We try to build consensus around an MCIP, but if that's not possible, you can always submit a competing MCIP.

If you are interested in assuming ownership of an MCIP, send a message asking to take over, addressed to both the original author and an MCIP editor. If the original author doesn't respond to email in a timely manner, an MCIP editor will make a unilateral decision (this may be reversed if the author resurfaces).



## MCIP format and structure

MCIPs should be written in Markdown format.

Each MCIP should have the following parts:

- **Preamble**: A header containing metadata about the MCIP (see [MCIP header preamble](#MCIP-header-preamble)).
- **Abstract**: A short (\~200 word) description of the technical issue being addressed.
- **Motivation**: Explain why the proposal is desirable. MCIPs that want to change the MobileCoin protocol (either the consensus protocol or network protocol) should clearly explain why the existing protocol is inadequate to address the problem that the MCIP solves.
- **Specification**: The specification describes the syntax and semantics of any new feature. The specification should be detailed enough to allow competing, interoperable implementations for any of the current MobileCoin platforms.
- **Rationale**: The rationale adds context to the specification by describing why particular design decisions were made, and important aspects of the design (especially related to security, when relevant). It should describe alternate designs that were considered, and related work. The rationale should discuss important objections or concerns raised during discussion.
- **Backwards compatibility**: If an MICP introduces backwards incompatibilities, describe them and their severity. The MCIP must explain how the author proposes to deal with these incompatibilities.
- **Reference implementation**: A reference implementation may be required for an MCIP to be given 'Proposed' or 'Active' status. The implementation must include test code and documentation appropriate for the MobileCoin protocol. Typically these will be represented by a link to a PR in a different repository, or a standalone repository implementing the proposal.
- **References**: A list of important references for the MCIP. These may include duplicates of hyperlinked sources from the body of the MCIP.


### MCIP header preamble

Each MCIP must begin with a header preamble. The headers must appear in the following order. Headers marked with "-" are optional and are described below. All other headers are required.

```
  MCIP: <MCIP number, or "?" before being assigned>
- Layer: <Consensus | Peer Services | API/RPC | Applications>
  Title: <MCIP title; maximum 44 characters>
  Authors: <list of authors' names and email addresses>
- Discussions-To: non-standard discussion location
  Status: <Draft | Withdrawn | Deferred | Rejected |
           Proposed | Inactive |
           Active | Replaced | Obsolete>
  Type: <Standards Track | Informational | Process>
  Created: <date MCIP number assigned, in ISO 8601 (yyyy-mm-dd) format>
- Requires: <MCIP number(s)>
- Replaces: <MCIP number>
- Superseded-By: <MCIP number>
```

The `Layer` header (only for Standards Track MCIPs) documents which layer of MobileCoin the MCIP applies to.
See [BIP-0123](https://github.com/bitcoin/bips/blob/master/bip-0123.mediawiki) for definitions of the various MCIP layers (note: we consider BIP-0123 'soft forks' and 'hard forks' to both qualify as 'hard forks').

The `Authors` header lists the names and email addresses of all the authors/owners of the MCIP. The format of the `Authors` header must be

```
  Authors: Random J. User <address@dom.ain>,
           Writer Two <addr@dom.ain>
```

While an MCIP is being discussed, a `Discussions-To` header may indicate a special mailing list or URL where the MCIP is being discussed. No `Discussions-To` header is necessary if the MCIP is being discussed privately with the author, or in the MobileCoin Foundation discord server's \#mcip channel ([invite link here](https://discord.gg/EUg9cW3PHR)).

The `Type` header specifies the type of MCIP: Standards Track, Informational, or Process.

The `Created` header records the date (in ISO 8601 yyyy-mm-dd format) that the MCIP was assigned a number.

MCIPs may have a `Requires` header, indicating the MCIP numbers that this MCIP depends on.

MCIPs may also have a `Superseded-By` header indicating that an MCIP has been rendered obsolete by a later document. The value is the number of the MCIP that replaces the current document. The newer MCIP must have a `Replaces` header containing the number of the MCIP that it renders obsolete.


### Auxiliary Files

MCIPs may include auxiliary files such as diagrams. Auxiliary files should be included in a subdirectory for that MCIP.



## MCIP types

There are three kinds of MCIP.

- **Standards Track**: Describes any change that affects most or all MobileCoin implementations, such as a change to the network protocol, a change in block or transaction validity rules, or any change or addition that affects the interoperability of applications using MobileCoin. Standards Track MCIPs consist of two parts, a design document and a reference implementation.
- **Informational**: Describes a MobileCoin design issue, or provides general guidelines or information to the MobileCoin community, but does not propose a new feature. Informational MCIPs do not necessarily represent a MobileCoin community consensus or recommendation, so users and implementors are free to ignore Informational MCIPs or follow their advice.
- **Process**: Describes a process surrounding MobileCoin, or proposes a change to (or an event in) a process. Process MCIPs are like Standards Track MCIPs but apply to areas other than the MobileCoin protocol itself. They may propose an implementation, but not to MobileCoin's reference codebase. They often require community consensus. Unlike Informational MCIPs, they are more than recommendations, and users are typically not free to ignore them. Examples include procedures, guidelines, hard fork coordination notices, changes to the decision-making process, and changes to the tools or environment used in MobileCoin development. Any meta-MCIP is also considered a Process MCIP.



## MCIP status field

An MCIP may have the following statuses:

- **Draft**: It's just a Draft, and content may change at the author's discretion. A Draft header should say `Draft (NOT FOR USE)` to emphasize the proposal's non-final status.
    - **Withdrawn**: A Draft withdrawn by the author.
    - **Deferred**: A Draft paused by its author.
    - **Rejected**: A Draft inactive for three years with unaddressed public criticism.
- **Proposed**: A finished MCIP. The document is considered complete by its author, there is sufficient consensus, there is an adequate reference implementation (where appropriate), and there is a plan for the proposal to reach Active status.
- **Inactive**: A Draft or Proposed MCIP with no activity for three years.
- **Active**: An MCIP with objectively verifiable real-world use (see below).
    - **Obsolete**: An Active MCIP that should no longer be used. Changing an MCIP to this status must have objectively verifiable and/or discussed reasons.
    - **Replaced**: An Active MCIP that should no longer be used because it was replaced by another MCIP.

An MCIP may change status from Draft to Proposed when it achieves rough consensus in the MobileCoin Foundation discord server's \#mcip channel ([invite link here](https://discord.gg/EUg9cW3PHR)). Such a proposal is said to have rough consensus if it has been open to discussion for at least one month, and no person maintains any unaddressed substantiated objections to it. Addressed or obstructive objections may be ignored/overruled by general agreement that they have been sufficiently addressed, but clear reasoning must be given in such circumstances.


### Progression to Active status

A Consensus-layer Standards Track MCIP must be accompanied by a hard fork of the MobileCoin blockchain. Hard forks always require adoption from the entire MobileCoin economy. MCIPs dependent on hard forks may only have 'Active' status after the hard fork has been executed. In the case of competing MobileCoin blockchains (e.g. chains that split from the same ancestor block), the chain supported by the MobileCoin Foundation should be considered canonical for our purposes.

[[[better criteria?]]]Peer services MCIPs should be observed to be adopted by at least 1% of public listening nodes for one month (either validator or watcher nodes).

API/RPC and application layer MCIPs must be implemented by at least two independent and compatible software applications.

Software authors are encouraged to publish summaries of what MCIPs their software supports, to aid in verification of status changes.

These criteria are considered objective ways to observe the de facto adoption of MCIPs, and are not to be used as reasons to oppose or reject an MCIP. Should an MCIP become actually and unambiguously adopted despite not meeting the criteria outlined here, it should still be updated to Active status.


### Rationale

#### What is the ideal percentage of listening nodes needed to adopt peer services proposals?

This is unknown, and set rather arbitrarily at this time (this is a rationale provided in BIP-0002). For a random selection of peers to have at least one other peer implementing the extension, 13% or more would be necessary, but nodes could continue to scan the network for such peers with perhaps some reasonable success. Furthermore, service bits exist to help identification upfront.

- Why is it necessary for at least two software projects to release an implementation of API/RPC and Application layer MCIPs, before they become Active?

1. If there is only one implementation of a specification, there is no other program for which a standard interface is used with or needed.
1. Even if there are only two projects rather than more, some standard coordination between them exists.

#### What if an MCIP is proposed that only makes sense for a single specific project?

The MCIP process exists for standardization between independent projects. If something only affects one project, it should be done through that project's own internal processes, and never be proposed as an MCIP in the first place.



## MCIP README

The MCIP README is the 'front page' of the MCIPs repository. It should say three things.

1. "MCIP process in effect: MCIP-mcipProcess"
1. "**Disclaimer**: MCIPs with 'Draft' status are just drafts, and should not be considered 'recommended for use' until they reach 'Proposed' or 'Active' status."
1. A table of MCIPs with the following format (taking this MCIP as an example, with number MCIP-0000):

Number | Layer | Title | Owner | Type | Status 
-------|-------|-------|-------|------|---------
0000   |       | MCIP Process | koe | Process | Draft



## MCIP licensing

The MCIP repository, and by extension all merged MCIPs, have the [MIT license](LICENSE). If an MCIP is disingenuously licensed prior to being merged, it will be rejected immediately.



## Rationale

#### Why is it called a 'MobileCoin Improvement Proposal' instead of a 'MobileCoin RFC'?

1. The MCIP process shares many similarities with the IETF RFC process (and RFC processes found in other projects such as Rust). One essential difference is the focus in the MCIP process (inherited from BIP-0002) on consensus-building and proposal progress. MCIPs have champions who are expected to advocate for real-world use of their proposals. These champions must actively obtain feedback from the MobileCoin community and project developers, and improve their proposals until consensus is reached. Proposals are given 'statuses' that track a proposal throughout its lifetime - from Draft (being worked on) to Proposal (mature) to Active (in use) to, perhaps, Obsolete (superceded or deprecated). In some cases, proposals cannot make progress without a reference implementation.
1. Unlike RFC processes designed for the workflow in monolothic projects (e.g. as found in Rust or within software companies), the MCIP process does not assume a proposal can either have a 'accepted and implemented' or 'rejected and ignored' state. An MCIP tries to have a positive (and definite/real) impact on a potentially diverse and heterogenous ecosystem, where success may mean only a small component of that ecosystem adopts the MCIP's suggestions.
1. As mentioned, the MCIP process is based of the BIP-0002 process, so it is appropriate to use a similar naming scheme.

#### Why are there MCIP editors? Doesn't this add unnecessary bureaucracy and increase the risk of gatekeeping?

1. It is necessary to define quality and format standards for MCIPs, which must be managed/enforced by some person(s).
1. For transparency, clarity, and accountability, it is better if MCIP editors are publicly and explicitly known, instead of relying on an ambiguous group of repository maintainers whose members can change invisibly. In practice, the concept of an 'MCIP editor group' is equivalent to writing down and publishing a list of active repository maintainers (those maintainers who are expected to manage the MCIP process).
1. Writing down MCIP editors makes it easier to contact, and interact with, whoever is managing the MCIP process.

#### What are the differences from BIP-0002?

1. **Licensing**: The MCIP repository is all MIT, in contrast to the BIP repository where each proposal can have different licenses.
1. **Editors**: Editors are appointed by the MobileCoin Foundation, which owns the MCIP repository. There is expected to be more than one editor, and there is an explicit appeals process for authors whose MCIPs are rejected.
1. **Header**: The header was slightly simplified (comments, license, and post history were removed).
1. **Motivation**: The Motivation section is now above the specification, so readers have better context before they encounter technical details.
1. **Statuses**: The Final/Active status is just 'Active'.
1. **Comments**: Comments were removed.
1. Various edits to improve readability.



## Backward compatibility

This proposal makes a number of structural changes to the MCIPs repository compared to [MCIP-0000](0000-mobilecoin-rfcs.md), which is the current active proposal process. Since only two proposals (MCIP-0000 and MCIP-0001) were created under that process, we believe those changes are not prohibitively disruptive.



## Reference implementation

Not applicable.



## References

- [BIP-0002](https://github.com/bitcoin/bips/blob/master/bip-0002.mediawiki)
