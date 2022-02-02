# MobileCoin Improvement Proposals (MCIPs)

[MobileCoin Improvement Proposals]: #mcips

Many changes, including bug fixes and documentation improvements can be
implemented and reviewed via the normal GitHub pull request workflow.

Some changes though are "substantial", and we ask that these be put through a
bit of a design process and produce a consensus among the MobileCoin community and
the teams.

The "MCIP" (MobileCoin Improvement Proposal) process is intended to provide a consistent
and controlled path for new features to enter the ecosystem, so that all
stakeholders can be confident about the direction the protocol is evolving in.


## Table of Contents
[Table of Contents]: #table-of-contents

  - [Opening](#mcips)
  - [Table of Contents]
  - [When you need to follow this process]
  - [Team-specific guidelines]
  - [Before creating an MCIP]
  - [What the process is]
  - [The MCIP life-cycle]
  - [Reviewing MCIPs]
  - [Implementing an MCIP]
  - [MCIP Postponement]
  - [Help this is all too informal!]
  - [License]
  - [Contributions]


## When you need to follow this process
[When you need to follow this process]: #when-you-need-to-follow-this-process

You need to follow this process if you intend to make "substantial" changes to the
MobileCoin Foundation codebase, or the MCIP process itself. What constitutes a
"substantial" change is evolving based on community norms and varies depending
on what part of the ecosystem you are proposing to change, but may include the
following.

  - Any semantic or syntactic change to the APIs which are not a bugfix.
  - Removing features.
  - Changes to the interface between the clients and servers.

Some changes do not require an MCIP:

  - Rephrasing, reorganizing, refactoring, or otherwise "changing shape does
    not change meaning".
  - Additions that strictly improve objective, numerical quality criteria
    (warning removal, speedup, better platform coverage, more parallelism, trap
    more errors, etc.)
  - Additions only likely to be _noticed by_ other developers-of-mobilecoin,
    invisible to users-of-mobilecoin.

If you submit a pull request to implement a new feature without going through
the MCIP process, it may be closed with a polite request to submit an MCIP first.


### Team specific guidelines
[Team-specific guidelines]: #team-specific-guidelines

For more details on when an MCIP is required for the following areas, please see
the foundation's team-specific guidelines for:

  - [consensus](consensus.md),
  - [fog](fog.md),


## Before creating an MCIP
[Before creating an MCIP]: #before-creating-an-mcip

A hastily-proposed MCIP can hurt its chances of acceptance. Low quality
proposals, proposals for previously-rejected features, or those that don't fit
into the near-term roadmap, may be quickly rejected, which can be demotivating
for the unprepared contributor. Laying some groundwork ahead of the MCIP can
make the process smoother.

Although there is no single way to prepare for submitting an MCIP, it is
generally a good idea to pursue feedback from other project developers
beforehand, to ascertain that the MCIP may be desirable; having a consistent
impact on the project requires concerted effort toward consensus-building.

As a rule of thumb, receiving encouraging feedback from long-standing project
developers, and particularly members of the relevant team is a good
indication that the MCIP is worth pursuing.


## What the process is
[What the process is]: #what-the-process-is

In short, to get a major feature added to MobileCoin, one must first get the MCIP
merged into the MCIP repository as a markdown file. At that point the MCIP is
"active" and may be implemented with the goal of eventual inclusion into MobileCoin.

  - Fork the MCIP repo [MCIP repository]
  - Copy `0000-template.md` to `text/0000-my-feature.md` (where "my-feature" is
    descriptive). Don't assign an MCIP number yet; This is going to be the PR
    number and we'll rename the file accordingly if the MCIP is accepted.
  - Fill in the MCIP. Put care into the details: MCIPs that do not present
    convincing motivation, demonstrate lack of understanding of the design's
    impact, or are disingenuous about the drawbacks or alternatives tend to
    be poorly-received.
  - Submit a pull request. As a pull request the MCIP will receive design
    feedback from the larger community, and the author should be prepared to
    revise it in response.
  - Now that your MCIP has an open pull request, use the issue number of the PR
    to update your `0000-` prefix to that number.
  - Each pull request will be labeled with the most relevant team, which
    will lead to its being triaged by that team in a future meeting and assigned
    to a member of the team.
  - Build consensus and integrate feedback. MCIPs that have broad support are
    much more likely to make progress than those that don't receive any
    comments. Feel free to reach out to the MCIP assignee in particular to get
    help identifying stakeholders and obstacles.
  - The team will discuss the MCIP pull request, as much as possible in the
    comment thread of the pull request itself. Offline discussion will be
    summarized on the pull request comment thread.
  - MCIPs rarely go through this process unchanged, especially as alternatives
    and drawbacks are shown. You can make edits, big and small, to the MCIP to
    clarify or change the design, but make changes as new commits to the pull
    request, and leave a comment on the pull request explaining your changes.
    Specifically, do not squash or rebase commits after they are visible on the
    pull request.
  - At some point, a member of the team will propose a "motion for final
    comment period" (FCP), along with a *disposition* for the MCIP (merge, close,
    or postpone).
    - This step is taken when enough of the tradeoffs have been discussed that
    the team is in a position to make a decision. That does not require
    consensus amongst all participants in the MCIP thread (which is usually
    impossible). However, the argument supporting the disposition on the MCIP
    needs to have already been clearly articulated, and there should not be a
    strong consensus *against* that position outside of the team. Team
    members use their best judgment in taking this step, and the FCP itself
    ensures there is ample time and notification for stakeholders to push back
    if it is made prematurely.
    - For MCIPs with lengthy discussion, the motion to FCP is usually preceded by
      a *summary comment* trying to lay out the current state of the discussion
      and major tradeoffs/points of disagreement.
    - Before actually entering FCP, *all* members of the team must sign off;
    this is often the point at which many team members first review the MCIP
    in full depth.
  - The FCP lasts 1 business days.
  - In most cases, the FCP period is quiet, and the MCIP is either merged or
    closed. However, sometimes substantial new arguments or ideas are raised,
    the FCP is canceled, and the MCIP goes back into development mode.

## The MCIP life-cycle
[The MCIP life-cycle]: #the-MCIP-life-cycle

Once an MCIP becomes "active" then authors may implement it and submit the
feature as a pull request to the relevant repo. Being "active" is not a rubber
stamp, and in particular still does not mean the feature will ultimately be
merged; it does mean that in principle all the major stakeholders have agreed
to the feature and are amenable to merging it.

Furthermore, the fact that a given MCIP has been accepted and is "active"
implies nothing about what priority is assigned to its implementation, nor does
it imply anything about whether a developer has been assigned the task of
implementing the feature. While it is not *necessary* that the author of the
MCIP also write the implementation, it is by far the most effective way to see
an MCIP through to completion: authors should not expect that other project
developers will take on responsibility for implementing their accepted feature.

Modifications to "active" MCIPs can be done in follow-up pull requests. We
strive to write each MCIP in a manner that it will reflect the final design of
the feature; but the nature of the process means that we cannot expect every
merged MCIP to actually reflect what the end result will be at the time of the
next major release.

In general, once accepted, MCIPs should not be substantially changed. Only very
minor changes should be submitted as amendments. More substantial changes
should be new MCIPs, with a note added to the original MCIP. Exactly what counts
as a "very minor change" is up to the team to decide; check
[Team-specific guidelines] for more details.


## Reviewing MCIPs
[Reviewing MCIPs]: #reviewing-MCIPs

While the MCIP pull request is up, the team may schedule meetings with the
author and/or relevant stakeholders to discuss the issues in greater detail,
and in some cases the topic may be discussed at a team meeting. In either
case a summary from the meeting will be posted back to the MCIP pull request.

A team makes final decisions about MCIPs after the benefits and drawbacks
are well understood. These decisions can be made at any time, but the team
will regularly issue decisions. When a decision is made, the MCIP pull request
will either be merged or closed. In either case, if the reasoning is not clear
from the discussion in thread, the team will add a comment describing the
rationale for the decision.


## Implementing an MCIP
[Implementing an MCIP]: #implementing-an-MCIP

Some accepted MCIPs represent vital features that need to be implemented right
away. Other accepted MCIPs can represent features that can wait until some
arbitrary developer feels like doing the work. Every accepted MCIP has an
associated epic tracking its implementation in a repository; thus that
associated epic can be assigned a priority via the triage process that the
team uses for all issues in a repository.

The author of an MCIP is not obligated to implement it. Of course, the MCIP
author (like any other developer) is welcome to post an implementation for
review after the MCIP has been accepted.

If you are interested in working on the implementation for an "active" MCIP, but
cannot determine if someone else is already working on it, feel free to ask
(e.g. by leaving a comment on the associated issue).


## MCIP Postponement
[MCIP Postponement]: #MCIP-postponement

Some MCIP pull requests are tagged with the "postponed" label when they are
closed (as part of the rejection process). An MCIP closed with "postponed" is
marked as such because we want neither to think about evaluating the proposal
nor about implementing the described feature until some time in the future, and
we believe that we can afford to wait until then to do so. Historically,
"postponed" was used to postpone features until after 1.0. Postponed pull
requests may be re-opened when the time is right. We don't have any formal
process for that, you should ask members of the relevant team.

Usually an MCIP pull request marked as "postponed" has already passed an
informal first round of evaluation, namely the round of "do we think we would
ever possibly consider making this change, as outlined in the MCIP pull request,
or some semi-obvious variation of it." (When the answer to the latter question
is "no", then the appropriate response is to close the MCIP, not postpone it.)


### Help this is all too informal!
[Help this is all too informal!]: #help-this-is-all-too-informal

The process is intended to be as lightweight as reasonable for the present
circumstances. As usual, we are trying to let the process be driven by
consensus and community norms, not impose more structure than necessary.


## License
[License]: #license

This repository is licensed as:

* MIT license ([LICENSE](LICENSE) or https://opensource.org/licenses/MIT)

### Contributions
[Contributions]: #contributions

Unless you explicitly state otherwise, any contribution intentionally submitted for inclusion in the work by you shall be licensed as above, without any additional terms or conditions.


[MCIP repository]: https://github.com/mobilecoinfoundation/MCIPs
