- Feature Name: Fog-Compatible-Gift-Codes
- Start Date: Mar 21
- MCIP PR: [mobilecoinfoundation/mcips#0032](https://github.com/mobilecoinfoundation/mcips/pull/0032)
- Tracking Issue: None

# Summary
[summary]: #summary

We propose an alternate mechanism of function for the "gift codes" feature, which is compatible with
Mobile wallets using MobileCoin Fog.

The payload of the new mechanism is similar in size and the process of redeeming it is less complex,
compared to the existing mechanism.

# Motivation
[motivation]: #motivation

Full service supports a "gift codes" feature. (Called "Tranfer code" [in sources](https://github.com/mobilecoinfoundation/mobilecoin/blob/7077b418fab65e05a05e4fca1d272e3dddd2e392/api/proto/printable.proto#L28).)

The idea of a gift code is that you can give money to
someone without knowing their public address before hand, creating a payload that can be handed to
anyone, which then funds their account. This could be used for gift cards for example.

The existing mechanism for gift codes is as follows:
* The person who creates a gift code, creates an entirely new set of account keys, and funds these account keys with an on-chain transaction.
* The gift code payload contains these private keys.
* To receive a gift code, the recipient drains the funds from this temporary account into their main account.

This mechanism is not very robust in the sense that, if an "attacker" intercepts the gift code, then they can race the
recipient to drain the funds. Indeed, even the sender is capable of racing the recipient to take back the funds.
However, the design goal of the feature is that the sender of the gift code wants whoever
obtains the payload to be able to use it, so these races are not considered a design flaw.
This feature has been used for example to implement the "red envelopes" feature in Mixin
messenger group chats -- one or more gift codes are transmitted in an encrypted group chat, and whoever clicks on it first
gets a festive reward.

While this functionality works when all the participants are using a desktop wallet like full-service, it turns out that this doesn't
work when the recipient is on a mobile device using a Fog account. The reason is that they are unable to perform a balance
check of the temporary account, since the outputs were not indexed by Fog.

This could be worked around, by making the person who creates the temporary account create it as a Fog account.
However, there could be several deployments of Fog, and this basically means that the creator of the gift code needs to know
what software the recipient is using, which is counter to the goal of the feature.

In later versions of transfer payloads, the `TxOut` public key was included.

```
/// Message encoding a private key and a UTXO, for the purpose of
/// giving someone access to an output. This would most likely be
/// used for gift cards.
message TransferPayload {
    /// [Deprecated] The root entropy, allowing the recipient to spend the money.
    /// This has been replaced by a BIP39 entropy.
    bytes root_entropy = 1 [deprecated=true];

    /// The public key of the UTXO to spend. This is an optimization, meaning
    /// the recipient does not need to scan the entire ledger.
    external.CompressedRistretto tx_out_public_key = 2;

    /// Any additional text explaining the gift
    string memo = 3;

    /// BIP39 entropy, allowing the recipient to spend the money.
    /// When deriving an AccountKey from this entropy, account_index is always 0.
    bytes bip39_entropy = 4;
}
```

It is possible to use the untrusted fog ledger endpoint to lookup `TxOut` by their public key,
but this is not private, and the fog operator can see which `TxOut` you looked up. This puts the
gift codes outside the normal threat-model for Fog transfers.

We propose to fix this in a quite simple way:

Instead of transfering private keys to an entire account, we just transfer ownership of a specific `TxOut`.
We give them private keys and secrets connected to this `TxOut`, but which don't reveal anything about our account private keys.

* To enable someone to spend a TxOut, we just have to give them the `onetime_private_key` of the `TxOut`.
* To enable someone to see the unblinded amount of the TxOut, we just have to give them the `shared_secret` of the `TxOut`.
* To enable someone to find the TxOut in the ledger (even a Fog user), we just have to tell them the global index in the blockchain.

Given this data, a Fog user who receives the payload can ask Fog ledger for a merkle proof of membership for the `TxOut`.
They can get the entire `TxOut` this way, match it against the shared secret and so unblind the amount commitment, and given the one-time
private key, they can spend it as normal using the transaction builder.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

A `TxOutGiftCode` is a payload which confers ownership of an individual `TxOut` in the blockchain,
and is a little less than 80 bytes in size.

* To enable someone to spend a TxOut, we just have to give them the `onetime_private_key` of the `TxOut`.
* To enable someone to see the unblinded amount of the TxOut, we just have to give them the `shared_secret` of the `TxOut`.
* To enable someone to find the TxOut in the ledger (even a Fog user), we just have to tell them the global index in the blockchain.

When a Fog wallet receives such a payload, they can first contact the Fog Ledger service with the global index, which returns to them a merkle proof of
membership, and a copy of the `TxOut` as it appears in the ledger. (Wallets that have a local `ledger_db` can just search the ledger directly.)

The wallet can then attempt to unmask the amount using the `shared_secret` of the `TxOut`. Assuming this succeeds, they know the value and blinding factor
of the `TxOut`. Given the one-time private key, they can compute the `KeyImage` and check if it is already spent. Finally, given all this data and a Merkle proof,
they can spend the Tx as normal with the `TransactionBuilder`.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

A `TxOutGiftCode` conforms to the following protobuf schema, using types from [`external.proto`](https://github.com/mobilecoinfoundation/mobilecoin/blob/924abcff4937fced12cdd951aad598dbdbbfab05/api/proto/external.proto):

```
message TxOutGiftCode {
    // The global index of the TxOut which is gifted
    fixed64 global_index = 1;
    // The one-time private key which can be used to spend this TxOut
    RistrettoPrivate onetime_private_key = 2;
    // The shared secret which can be used to unblind the amount of this TxOut
    CompressedRistretto shared_secret = 3;
}
```

As described, a Fog wallet can look up a `TxOut` from its global index, and then
attempt to unblind the amount using the shared secret.

To add an input to the transaction builder, normally we construct an [`InputCredentials` object](https://github.com/mobilecoinfoundation/mobilecoin/blob/924abcff4937fced12cdd951aad598dbdbbfab05/transaction/std/src/input_credentials.rs#L11):

```
pub struct InputCredentials {
    /// A "ring" containing "mixins" and the one "real" TxOut to be spent.
    pub ring: Vec<TxOut>,

    /// Proof that each TxOut in `ring` is in the ledger.
    pub membership_proofs: Vec<TxOutMembershipProof>,

    /// Index in `ring` of the "real" output being spent.
    pub real_index: usize,

    /// Private key for the "real" output being spent.
    pub onetime_private_key: RistrettoPrivate,

    /// Public key of the transaction that created the "real" output being
    /// spent.
    pub real_output_public_key: RistrettoPublic,

    /// View private key for the address this input was sent to
    pub view_private_key: RistrettoPrivate,
}
```

All the clients are capable of sampling rings, obtaining membership proofs, and the
`TxOutGiftCode` contains the one-time private key.

However, the `InputCredentials` appears to require the view private key, which the recipient
of the `TxOutGiftCode` does not have.

Fortunately, it turns out that they don't really need it, it is only used in the `TransactionBuilder`
to construct the `shared_secret`:

https://github.com/mobilecoinfoundation/mobilecoin/blob/924abcff4937fced12cdd951aad598dbdbbfab05/transaction/std/src/transaction_builder.rs#L410

We propose either to
* Modify `InputCredentials` so that it includes the `shared_secret` instead of the `view_private_key`.
  Since clients usually compute the `shared_secret` as soon as they receive an input as a by-product of view-key matching,
  this may actually be more efficient and prevent computing elliptic curve operations again unnecessarily.
* Modify `InputCredentials` so that it holds either the `shared_secret` OR the `view_private_key`, for backwards
  compatibility.

Clients will then be able to go from a `TxOutGiftCode` to an `InputCredentials` very easily.

# Drawbacks
[drawbacks]: #drawbacks

In our opinion, there is no reason why we should not do this, and this version of gift codes is superior to the existing version.
The existing version is very complicated and forces all the clients to view-key scan the ledger with a new set of keys in order
to process a gift code. The "fixed" version that returns the `TxOut` public key avoids the view key scanning issue, but it still
isn't really compatible with Fog's privacy model. It is much simpler to just give away ownership of an existing `TxOut`, and
give the fog recipient a global `TxOut` index.

This also makes creating gift codes instantaneous and not require on-chain events and fees.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

There are a few minor technical variations we considered like, not revealing the shared secret, and revealing
the amount and blinding factor directly instead, but this is more bytes on the wire, and it may be useful for
the recipient to be able to read the memo anyways.

# Prior art
[prior-art]: #prior-art

None that we are aware of.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

None at this time.

# Future possibilities
[future-possibilities]: #future-possibilities

None at this time.
