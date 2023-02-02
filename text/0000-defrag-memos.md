- Feature Name: RTH Defragmentation Memos
- Start Date: (fill me in with today's date, 2023-02-02)
- MCIP PR: [mobilecoinfoundation/mcips#0000](https://github.com/mobilecoinfoundation/mcips/pull/0000)

# Summary
[summary]: #summary

This proposal introduces a new memo type to support [RecoverablTransaction History (RTH)](https://github.com/mobilecoinfoundation/mcips/pull/4). The new memo type will be used by clients to mark `TxOut`s created as part of account defragmentation. This can be used by client applications as a reliable way to exclude defragmentation transactions from appearing as regular transactions.

# Motivation
[motivation]: #motivation

[RTH](https://github.com/mobilecoinfoundation/mcips/pull/4) introduced a standard method of utilizing [encrypted memos](https://github.com/mobilecoinfoundation/mcips/pull/3) for recording transaction history between two users. Not all transactions on the MobileCoin blockchain are between two different users or accounts, however. Defragmenation transactions are transactions that a user sends to themselves in order to consolidate small `TxOut`s into larger ones. This is necessary if a user needs to send a large transaction but cannot satisfy the amount using 16 `TxOuts`s or less.

From the client's perspective, these larger `TxOut`s created by defragmentation appear as `TxOut`s received through normal transactions. This would be confusing to users if displayed this way in a client application. Currently, the only way to differentiate defragmentation transactions from regular ones is to use [RTH]([RecoverablTransaction History (RTH)](https://github.com/mobilecoinfoundation/mcips/pull/4) and check if the sender and recipient are both the user's account.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

This proposal adds one new encrypted memo type.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

> This is the technical portion of the MCIP. Explain the design in sufficient detail that:
>
> - Its interaction with other features is clear.
> - It is reasonably clear how the feature would be implemented.
> - Corner cases are dissected by example.
>
> The section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work.

# Drawbacks
[drawbacks]: #drawbacks

> Why should we *not* do this?

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

> - Why is this design the best in the space of possible designs?
> - What other designs have been considered and what is the rationale for not choosing them?
> - What is the impact of not doing this?

# Prior art
[prior-art]: #prior-art

> Discuss prior art, both the good and the bad, in relation to this proposal.
> A few examples of what this can include are:
>
> - For consensus and fog proposals: Does this feature exist in other systems, and what experience have their community had?
> - For community proposals: Is this done by some other community and what were their experiences with it?
> - For other teams: What lessons can we learn from what other communities have done here?
> - Papers: Are there any published papers or great posts that discuss this? If you have some relevant papers to refer to, this can serve as a more detailed theoretical background.
>
> This section is intended to encourage you as an author to think about the lessons from other systems, provide readers of your MCIP with a fuller picture.
> If there is no prior art, that is fine - your ideas are interesting to us whether they are brand new or if it is an adaptation from other systems.
>
> Note that while precedent set by other systems is some motivation, it does not on its own motivate an MCIP.
> Please also take into consideration that MobileCoin sometimes intentionally diverges from common cryptocurrency features.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

> - What parts of the design do you expect to resolve through the MCIP process before this gets merged?
> - What parts of the design do you expect to resolve through the implementation of this feature before stabilization?
> - What related issues do you consider out of scope for this MCIP that could be addressed in the future independently of the solution that comes out of this MCIP?

# Future possibilities
[future-possibilities]: #future-possibilities

> Think about what the natural extension and evolution of your proposal would
> be and how it would affect the project as a whole in a holistic way. Try to
> use this section as a tool to more fully consider all possible interactions
> with aspects of the project in your proposal. Also consider how this all
> fits into the roadmap for the project and of the relevant team.
>
> This is also a good place to "dump ideas", if they are out of scope for the
> MCIP you are writing but otherwise related.
>
> If you have tried and cannot think of any future possibilities,
> you may simply state that you cannot think of anything.
>
> Note that having something written down in the future-possibilities section
> is not a reason to accept the current or a future MCIP; such notes should be
> in the section on motivation or rationale in this or subsequent MCIPs.
> The section merely provides additional information.

