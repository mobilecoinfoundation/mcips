- Feature Name: recoverable_scis
- Start Date: 2023-03-02
- MCIP PR: [mobilecoinfoundation/mcips#0062](https://github.com/mobilecoinfoundation/mcips/pull/0062)
- Tracking Issue: (link to tracking issue)

# Summary

[summary]: #summary

This MCIP proposes that inputs to be used in an SCI first be sent to a reserved subaddress `SCI_SUBADDRESS_INDEX`, extending the scheme from [MCIP #36](0036-reserved-subaddresses.md)

This MCIP also proposes that new RTH memos be created to further support inputs used in SCIs, extending the scheme from [MCIP #4](0004-recoverable-transaction-history.md)

# Motivation

[motivation]: #motivation

Similar to the motivations in [MCIP #32](0032-fog-compatible-gift-codes.md), a concern for users is the ability to track things across linked devices using the blockchain (and not separate services). It is a concern wrt txos used as an SCI in that one instance of an account could use a txo as an SCI and before the recipient of said SCI completes a transaction with it (which could take some arbitrary yet non trivial amount of time), another (or even the same, if data is not properly tracked) instance of the account could use that txo as an input in another transaction, therefore unintentionally rendering the SCI cancelled.

Another concern is that because there are multiple types of ways to specify the rules of an SCI (one example being [MCIP #42](0042-partial-fill-rules.md) which extended the SCI rules to include partial fills) one might want to have some record of that information on the blockchain (instead of relying on a secondary service)

# Guide-level explanation

[guide-level-explanation]: #guide-level-explanation

When one wishes to generate an SCI, the app must first make a self payment to the `SCI_SUBADDRESS_INDEX = u64::MAX - 3`, which is now reserved for `TxOut`s used as `SCI`s.
Apps should not select `TxOut`s at this subaddress for inputs in other transactions, and they may choose not to include these in the account balances as they represent the amount of active `SCI`s.

After an SCI is sent, an app can check if it has been claimed based on if this `TxOut` has been spent. Until it has been spent, the app can potentially cancel the `SCI` by sending a self payment with the `TxOut`.

We additionally extend Recoverable Transaction History memos from [MCIP #4](0004-recoverable-transaction-history.md) to handle `SCI`s as well.

| Memo type bytes | Name                  |
| --------------- | --------------------- |
| 0x0301          | SCI generation memo   |
| 0x0302          | SCI cancellation memo |
| 0x0003          | SCI sender memo       |

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

> Why should we _not_ do this?

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
