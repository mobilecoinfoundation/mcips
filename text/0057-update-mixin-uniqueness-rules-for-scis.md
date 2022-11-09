- Feature Name: update-mixin-uniqueness-rules-for-scis
- Start Date: 2022-11-03
- MCIP PR: [mobilecoinfoundation/mcips#0057](https://github.com/mobilecoinfoundation/mcips/pull/0057)
- Tracking Issue: [mobilecoinfoundation/mobilecoin#2813](https://github.com/mobilecoinfoundation/mobilecoin/issues/2813)

# Summary
[summary]: #summary

Update the transaction validation rules around  "DuplicateRingElements" error,
relaxing them when MCIP #31 Signed Contingent Inputs are present in the transaction.

# Motivation
[motivation]: #motivation

Currently, the consensus enclave enforces the following rule: a ring element cannot appear in two different inputs in the transaction.

If this rule is violated, the transaction fails with a "DuplicateRingElements" error.

The rationale for this rule is
* It is generally possible for clients to sample the mixin ring elements without replacement,
  removing all the real inputs, and so ensure that duplicate ring elements do not occur.
* Doing so may make it more difficult for an adversary to statistically analyze the rings.

This only justifies why this rule should be a **best practice for clients**, but not why
it should be enforced at the level of the consensus network. In fact, this rule is not needed
to protect the integrity of the cryptocurrency -- removing it doesn't allow double spends
or imbalanced transactions. It simply helps to prevent clients from using bad ring sampling
code in production.

After MCIP #31, the rationale for the rule is no longer universally applicable, because
there may be multiple parties involved in building a transaction. Consider the following scenario:

1. Alice creates a Signed Contingent Input. Alice's client must choose a ring of mixins at this time.
   By chance, Alice chooses one of Bob's outputs as a mixin.
1. Bob tries to create a transaction that incorporates this SCI. Bob's client adds this SCI to the
   transaction builder, and to pay for it, performs input selection, also adding some of Bob's outputs.
1. The consensus network rejects Bob's transaction with a "DuplicateRingElements" error.
   This is because Bob's real input is the same as one of Alice's mixins.

This scenario currently causes testing failures in continuous integration and continuous deployment
for block version 3 revisions. It is not a very easy issue to fix:

* Alice's client cannot work around this because Alice has no idea in advance who will match their SCI.
  The design goal is that Alice's client shouldn't need to care.
* Bob's client cannot work around this because if, for example, Bob only has one output with significant
  value, and it is selected by Alice's client as a mixin, then there is no way that Bob's client can make
  input selection work, even if Bob does have enough funds to satisfy Alice's offer. This will ultimately
  mean giving up and surfacing a strange and hard to understand error to the user.

We take the point of view that, the ground has shifted and the rationale for the original consensus rule has been
undermined. It is no longer the case that

> It is generally possible for clients to sample the mixin ring elements without replacement,
> removing all the real inputs, and so ensure that duplicate ring elements do not occur.

We propose that, the original rule applies to any inputs without rules, and a lesser rule
applies to any inputs with rules -- we only check that ring elements are unique within a ring,
when the input contains rules. 

Input rules and SCI's are new in block version 3, which is not yet released. This change does not allow any transactions that would have been invalid in block version 2 to be valid -- it only changes how this historical rule will apply to new transactions that involve block version 3 features.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The current rule (as of block version 2) is that:

* Ring elements cannot appear twice across all inputs in the transaction.

The proposed rule is:

* Ring elements cannot appear twice across all inputs without rules in the transaction.
* For each input with rules, the elements of its ring cannot appear twice in that ring.

This change only impacts transactions that involve SCIs.

# Drawbacks
[drawbacks]: #drawbacks

The main reason not to do this is fear that this will somehow weaken the security
of rings.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

One alternative is, simply do nothing. How likely are these collisions anyways?

* In continuous integration testing, we generally test with fairly small ledgers.
  It seems that out of 40 swap transactions, one of them fails this way about 1/3 of the time.
* In production, the mainnet ledger has on the order of millions of TxOuts. The chance that
  two randomly chosen TxOut's collide is about 1/1000, and with 11 mixins for each input,
  this chance will get a little higher. So I would estimate that somewhat less than 1% of
  swap attempts will fail in this way.

Another alternative is, clients could handle this situation by making a self-payment.
They could spend the input they wanted to select, which was covered by the SCI they
are trying to match against, and then attempt to match the SCI again after that transaction
settles.

This has some drawbacks:

* Harms the user experience, which now requires an additional transaction fee and waiting time
  to accomplish what they were trying to accomplish.
* Increases the complexity of client code
* Not much different in terms of ring analysis.

To this last point: If an adversary who can see the rings sees that

* An SCI was published and a particular TxOut was one of its ring elements
* A Tx was subsequently submitted using that same TxOut as one of its ring elements (but not using the SCI)
* One of the outputs of that Tx was subsequently used immediately in a Tx that also consumed this SCI

then the adversary can probably infer that this TxOut was used to consume the SCI.

On the other hand, if we followed this proposal and allowed this to happen in one transaction,
there is still a plausible alternative that Alice happened to choose a mixin which is the same
as a mixin chosen by Bob. (This will happen with nontrivial probability due to the birthday paradox.)

Finally, the main concern around this change is that allowing this could somehow weaken the security
of rings. 

* Ultimately, the security of rings derives from the algorithm used to sample rings, and not
  this particular check in the consensus network.
* Even if there is an overlap of one ring element between an SCI and the rest of the rings,
  this is no worse than having a ring size of 10 instead of 11 for the SCI, which is only a
  marginal change. And it is still considerably better than that, because the adversary has no
  obvious way to determine if this ring element is the true input of the SCI, the other ring,
  or neither.
* Even in quite small ledgers, the chance that there are considerably more than one or two
  collisions is very low, while the chance that there is an overlap of one or two is nontrivial.
* The right way to test if ring sampling implementations are working correctly is to unit test
  them. Collect many samples and make assertions about their statistics and frequency of
  collisions. Testing by enforcing hard rules at the level of the consensus network
  is insufficient to ensure security and correctness of the implementation.

Another concern which was raised in review is that the stauts quo allows an active attack along
the following lines:

* Adversary creates an SCI, selecting as a mixin a TxOut that is conjectured to belong to the victim.
* Victim tries to match against this SCI, building a transaction using it
* Adversary observes the `DuplicateRingElement` error and gets nontrivial statistical insight into the
  victim's TxOut set.

This proposal mitigates this concern.

# Prior art
[prior-art]: #prior-art

[MCIP 17](https://github.com/mobilecoinfoundation/mcips/pull/17) discussed alternative ring selection strategies.

To our knowledge, no other blockchains using ring signatures have a feature like MCIP 31,
so we don't expect to find prior art around this.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

At this time, we aren't suggesting any specific changes to how ring sampling should work
in the presence of SCIs. We hope that future MCIPs that introduce new ring sampling procedures
will touch on this issue.

# Future possibilities
[future-possibilities]: #future-possibilities

In the longer term, we hope that MobileCoin will move away from using ring signatures. If instead
we use ZK-SNARKs to conceal merkle proofs and play the role of ring signatures in our current
transaction math, then there would be no need for rings or ring elements at all, and this issue
becomes completely moot.
