- Feature Name: Block-Version-based Protocol Evolution
- Start Date: 2022-02-05
- MCIP PR: [mobilecoinfoundation/mcips#0026](https://github.com/mobilecoinfoundation/mcips/pull/0026)
- MobileCoin Epic: None

# Summary
[summary]: #summary

Since inception, MobileCoin has not yet made any changes to the ledger format.

This document proposes a general approach to rolling out changes to the ledger format
efficiently while carefully maintaining backwards compatibility.

# Motivation
[motivation]: #motivation

Conventional web applications today are usually developed using data interchange
formats that support schema evolution with forward and backwards compatibility.
A good example of this is protobuf -- protobuf allows that new fields can be
added to schema, and new programs will be able to deserialize old payloads, and
old programs will be able to deserialize the new payloads (ignoring the new fields).

As long as changes are strictly additive then they can be rolled out without
breaking old clients. Eventually the old clients are sunsetted and only the new
clients are supported.

In blockchains, this is made more complicated for two reasons:
* Older data payloads exist forever in the blockchain and clients have to be able to handle them forever
* Frequently, clients and nodes need to agree about the hash of of a piece of data, so that
  clients that sync the blockchain compute the correct block id, and clients that submit merkle proofs
  of membership compute correct proofs.

Because of the need for everyone to compute hashes correctly, it may not be enough for
an old client to successfully deserialize a new payload by ignoring the fields it doesn't
understand -- if those fields are ignored, then it will compute a different hash for
this object from the new clients.

To make this concrete, this means that if we want to roll out changes that add new
features to `TxOut`'s, the new clients and old clients probably cannot coexist the way
they might in a more conventional application.

If a new client sends a `TxOut` to a user using an old client, which contains a `TxOut`
feature that they don't recognize, then:
1. They may fail to compute the correct block id
2. If they submit a transaction spending the TxOut, the merkle proof of membership
   may be wrong if the hash doesn't incorporate the new field.
3. Even if they are a fog client, and (new) fog is building the proof of membership for them,
   if the client strips out the fields it doesn't understand that fog gives them, then consensus
   will likely reject the transaction. (There are some protobuf implementations that support "preserving"
   unknown fields, but PROST, which is the only rust implementation which is enclave-compatible, does not do this.)

This proposal describes a way to navigate these difficulties:
* Schema evolution can be accompanied by bumps to `block_version` (which has never been bumped before)
* Clients can be built that support multiple values of `block_version`
* The clients all switch to using a newer `block_version` at the same time, preventing breakage.

Thus, new transactions do not begin happening immediately when new clients are shipped -- instead, backwards-compatible
clients are shipped, and new behavior is triggered everywhere at once after a suitable deprecation window for the
old clients. Transaction validation enforces that new fields do not appear (which could break the old clients) until
the appropriate time.

This means we can adhere to a support model like the following:
* Clients (e.g. for individuals or merchants) are shipped and supported for `X` days
* Users are expected to update their software at least every `X` days
* Clients are never broken as long as they update on schedule

If we don't accept some proposal like this, then (we argue) we are forced into a world
where clients sometimes break prematurely.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The `block_version` is an existing field in the MobileCoin block header. It currently always has the value 1.

When new fields are added to the blockchain, e.g. to the `TxOut` object, the following principles should be followed:
  * The changes must be additive (in the sense of protobuf)
  * A new value of `block_version` must be announced, (in the MCIP that introduced the changes)
  * The new fields are only accepted by consensus when the `block_version` on chain has incremented
  * Before the block version can be increased (and the new feature activated), clients must be shipped which are `block_version` aware, and should support at least two block versions,
    an "old" version and a "new" version.
  * When building transactions, clients will learn the "current" block version either from the ledger, or from Fog, and will
    build "old" or "new" transactions as appropriate. (The transaction builder will be modified to make this easy.)
  * The enclaves will support a run-time determination of `block_version`, and transaction validation will be `block_version` aware.
  * A mechanism will be described which triggers the increase of the block version. This mechanism may vary from MCIP to MCIP but should
    be governed by the consensus of the node operators.

We also propose a `block_version` increment for the `0003-encrypted-memos` MCIP, and propose
a `block_version` increment for the (nascent) `0025-confidential-token-ids` MCIP.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The `block_version` number already exists in the MobileCoin block header, and as a constant in `mc-transaction-core`.
It has never been modified.

There is a constant called `BLOCK_VERSION` in `mc-transaction-core` which has value 1.
This constant will be renamed `MAX_BLOCK_VERSION`, and the meaning of the constant will be, the largest value of
`block_version` which this version of `mc-transaction-core` has support for.

This proposal largely is about establishing norms for server and client implementations, particularly, that they
should both be `block_version` aware and support multiple block versions when an upgrade occurs.

* The `block_version` of the blockchain is the `block_version` of the latest block.
* The block version of a block can never be less than the `block_version` of its parent.
* We propose that the encrypted memo field from `0003-encrypted-memos` MCIP should be associated to a bump to `block_version = 2`.
  That is, consensus will reject `Tx` that contain any `TxOut` that have memos until `block_version = 2`.
* We propose that the `masked_token_id` field from the  `0025-confidential-token-ids` MCIP should be associated to a bump to `block_version = 3`.
  That is, consensus will reject `Tx` that contain any `TxOut` that have this field until `block_version = 3`.
* We propose that `block_version = 3` may be triggered by the acceptance of the first minting transaction to the blockchain.
* We do not at this point have a concrete proposal for how to trigger `block_version = 2`.

Summary of block versions:

------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
| Block Version | New Field                 | Description                                                                                                                                                  |
| ------------- | ---------                 | -----------                                                                                                                                                  |
| 1             |    -                      | Status quo                                                                                                                                                   |
| 2             | `TxOut::e_memo`           | This is the first block version at which `e_memo` field will be accepted, and it will be required going forwards. This represents the changes in [MCIP #3](https://github.com/mobilecoinfoundation/mcips/pull/0003).    |
| 3             | `Amount::masked_token_id` | This is the first block version at which `masked_token_id` will be accepted, and it will be required going forward. This represents the changes in [MCIP #25](https://github.com/mobilecoinfoundation/mcips/pull/0025). |
------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

We'll attempt to describe code changes that would be needed in mobilecoin software to implement this:

The current block version can be obtained from the `mc-ledger-db` LMDB ledger interface by getting
the block header of the latest block, so clients like `mobilecoind` and `full-service` can already access this number.

The changes that are required are:
* Fog should start telling mobile clients about the latest block version, similarly to how it currently advertises the
"highest known block count". Both fog-view and fog-ledger should do this.
* The standard transaction builder should become `block_version` aware. Clients should pass the latest block version to it.
* Transaction validation should become `block_version` aware.
* The consensus enclave should support dynamic `block_version`.

Mobile clients SHOULD use the `latest_block_version` number to indicate to the users that an upgrade is required. The constant
`mc_transaction_core::MAX_BLOCK_VERSION` will indicate the largest block version that this version of the `TransactionBuilder` object supports.

Desktop clients, which use the lmdb database `mc-ledger-db`, already work by comparing the block version of new blocks to the
`mc_transaction_core::BLOCK_VERSION` constant. They should continue doing this, but the constant should be renamed `MAX_BLOCK_VERSION`.

We do not at this time propose that the `Tx` object itself should contain a `block_version`.
Instead, transactions are valid if they conform to the rules for the current block version.

# Drawbacks
[drawbacks]: #drawbacks

This adds complexity to the transaction validation code, which now must support multiple versions of the rules.

This adds complexity to the transaction building code, which also must support multiple versions of the transactions.

This adds complexity to off-line transaction building workflows, which must target a block version. (However these workflows
currently work by moving a copy of the ledger to the offline machine, which will carry the block version.)

This MCIP proposes ad-hoc mechanisms to trigger increases to the block version.

This MCIP proposes that `0003-encrypted-memos` requires a bump to block version, but does not propose a mechanism to trigger
the bump. If a suitable mechanism is not proposed, then as an alternative there could be one bump to block version for both the encrypted memos
changes and the confidential token ids change. Another alternative may be to implement `0007-Protocol-version-statement-type` MCIP.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

* An alternative upgrade narrative for encrypted memos looks something like this:
  * First, a client update is released so that clients can read memos, but do not write them. (Upgrade readers before writers.)
  * After `X` days all clients can read memos. Deploy an enclave which allows memos to be written, but does not require it.
    (Clients in the first step have to trust this enclave.)
  * Release clients that write memos. Gradually all the clients update, and for a while there is a mix of behaviors.
  * After `X` days all clients are writing memos. We can update the enclave to require memos.
    (Clients in the second step have to trust this enclave.)
  
  This would work for memos, but it requires two `X` day cycles rather than one, because every client is either
  memo-writing or not memo-writing. Another way it could work is, clients look at the `MRENCLAVE` they get when they
  attest to decide whether to write memos or not. But this creates a lot of complexity around configuring the clients
  correctly (for dev networks, testnet, and mainnet), and it impacts off-line wallet workflows because currently they
  can build transactions without having this attestation measurement.
  
  For more complex changes like confidential token ids, it's not clear that this would work. Outputs from the new transactions
  would not be valid to use in the old transactions, because the new outputs have masked token id, and so some sort of proof
  is required to show the enclave that all the token ids are matching apporpriately.
  
  Clients that receive such
  outputs have to switch to writing the new style transactions, but they also can't switch to doing this before an enclave
  is deployed that is willing to accept those transactions. In the period of time where there is a mix of behaviors, the clients
  that didn't update yet are broken.
  
  The difference between this case and the memos case is that even though in terms of protobuf schema, the confidential token ids
  change is additive, in terms of semantics, it isn't -- we are moving from a world where we know in the clear that every output
  has token id MOB (the only token), to a world where no one knows the token id of any output. That is, the network is trying to
  forget / conceal some information that wasn't previously. Outputs that have memos can just drop into the transaction math
  without changes, but outputs that have masked token id cannot, because their semantic has changed and we have less information
  about the output than before.

* If we wish to evolve the protocol in ways like this, where new outputs would not be spendable
  using the old transaction proofs, then the "rolling release" of upgrades to the transactions cannot always work, because it
  implies that for a period of time, new clients and old clients are coexisting, and so new TxOut's and old clients are coexisting,
  and that leads to the breakage in some scenarios.

* If such a "rolling release" strategy cannot work, then we must instead do something where all the clients cut over at once
  to the new behavior. There must be a source of truth for this cutover, and the logical place for this is in the blockchain.
  The `block_version` value seems a perfectly suitable thing to use for this purpose.

* An alternative may be to use feature flags instead of a version number, so that the software supports turning on
  individual blockchain features in any order.
  It seems to the author that this has many drawbacks:
  * More complexity, since some features may depend on other features.
  * More testing surface area
  * More code in more places
  
  Since in real-life, features will turn on in some scheduled order, it will simplify the code if the code assumes the
  order the features will turn on. We may be able to simplify the code though by adding predicates like
  ```
  fn feature_memos_is_supported(block_version: u32) -> bool { block_version >= 2 }
  fn feature_masked_token_id_is_supported(block_version: u32) -> bool { block_version >= 3 }
  ```
  and using these in the transaction validation and construction routines, which will get us most of the benefits
  of a feature-flag based system from an engineering point of view.

* Another alternative might have been to completely scrap `mc-crypto-digestible`. Instead, when we need to compute
  canonical hashes, use either a canonical serialization format such as Libra Canonical Serialization, or, describe and
  implement canonical protobuf serializers, or a protobuf canonizer routine. Then, clients always compute hashes of
  serialized objects, preserving unknown fields.
  * One reason not to do this is practical, it would be a pretty massive change to the system as it currently exists.
  * Another reason not to do it is philosophical. Even if a client computes the correct hash, it may not be wise for them
    to assume that e.g. they know the balance of the user if there are fields in the blockchain that they don't understand.
    Some changes like `masked_token_id` fundamentally change the meaning of `TxOut` and clients need to update to interpret
    `TxOut` correctly.
  * It seems that this would resolve issues like adding memos, but it's not clear that this would let us navigate changes
    to the transaction math like confidential token ids. Even if old clients can correctly hash new objects, it doesn't
    necessarily mean that they can spend new TxOut's successfully using the old math, if we make significant enough changes.

* A benefit of using `block_version` to trigger clients to cut over to new behavior is, we can deliver new features to all
  clients in one `X` day cycle rather than two `X` day cycles. And this doesn't create the same level of complexity in the
  system as, the clients' behavior being different depending on what `MRENCLAVE` they find during attestation. Using a hard
  cut-over like this means we never have to consider the possibility of a client spending new TxOut's using old math.

# Prior art
[prior-art]: #prior-art

This proposal relates to several earlier MCIP proposals:

* [0007-Protocol-version-statement-type](https://github.com/mobilecoinfoundation/mcips/pull/0007)
* [0008-SCP-based-hard-forks](https://github.com/mobilecoinfoundation/mcips/pull/0008)
* [0009-Dual-rule-blocks](https://github.com/mobilecoinfoundation/mcips/pull/0009)

In particular, `0007-protocol-version-statement-type` proposes a way that node operators
could trigger `block_version` to increase by voting for this and reaching consensus on it using SCP itself.

This proposal would resolve some of the drawbacks in this proposal around ad-hoc mechanisms, and may strengthen this proposal.
The main concern would be the risk and complexity around adding new ballot types to SCP as envisioned in the proposal.

The `0008-SCP-based-hard-forks` proposal would also normalize the process around triggering increases to the `block_version`.

The `0009-Dual-rule-blocks` proposal envisions that nodes may accept both "new" transactions and "old" transactions simultaneously
for a period of time. However, as we have seen in this proposal, this will lead to breakage of old clients in some
situations, for both the `0025-confidential-token-ids` changes and the `0003-encrypted-memos` changes.
This proposal can be seen as modifying `0009-Dual-rule-blocks` to avoid this breakage.

Many of these issues were anticipated when we designed `mc-crypto-digestible`, a hashing framework which
makes it easy to do protobuf-style schema evolution in rust without changing the hashes of old structures.
(The interested reader may refer to the README of that crate.)
However, `mc-crypto-digestible` does not help old clients compute hashes of new structures (because they
ignore fields they don't know about, and don't pass them to `mc-crypto-digestible`). 

The [documentation for Libra Canonical Serialization](https://docs.rs/libra-canonical-serialization/latest/libra_canonical_serialization/#background)
touches on these issues and the motivation as well.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

* Is there an elegant way to trigger the encrypted-memos version bump?
  It would be nice not to have to do these two changes simultaneously.
* Should we attempt `0007-Protocol-version-statement-type` before attempting this?
* Should we split this MCIP into a "process" part and an "implementation" part?
  
# Future possibilities
[future-possibilities]: #future-possibilities

It may be possible to avoid incrementing `block_version` if new fields are not added
to the hash, e.g. if they are marked `digestible(omit)`. However, this means that they
are light-weight in the chain and could conceiveably by changed after the block is signed,
which is usually undesirable.

It may be possible to specify that a different set of fields in `TxOut` is hashed for purpose
of computing merkle proofs of membership. For instance, some fields like the encrypted memo
and the encrypted fog hint do not play a role in transaction validation when a `TxOut` is spent.
This would also lead to efficiency gains because the memo and the fog hint are large,
and if `fog-ledger` doesn't have to send them back when buliding merkle proofs and mixins for the
clients, that saves the client bandwidth and avoids sending them data they wouldn't need anymore,
and it also may make it easier to build the oblivious merkle proof service.
