- Feature Name: Standardized Sender Signature
- Start Date: 2022-03-14
- MCIP PR: [mobilecoinfoundation/mcips#0030](https://github.com/mobilecoinfoundation/mcips/pull/0030)
- MobileCoin Epic: None

# Summary
[summary]: #summary

Enact MCIP #18 (Janus Mitigation), which sets the ephemeral key in the encrypted fog hint
to be the tx private key times the Ristretto base point. The original motivation was to enact
Janus attack mitigation.

We find a secondary use-case to motivate this: it becomes possible for the signer to create
a Schnorr signature over any payload, which a verifier can validate against the keys appearing
in a `TxOut`, without revealing the identity of the signer. We propose to both do #18 and standardize
the construction and validation of this signature.

# Motivation
[motivation]: #motivation

Often in a payments flow, it is necessary for the sender and recipient to exchange additional messages
and metadata about the payment. Frequently, these messages need to be authenticated.

We have a mechanism for doing this in the memo field, which attaches fixed-size payloads to TxOut's.
However, this imposes fairly strict size requirements on the data that can be sent.

Outside of the memo field, we have a notion of "receipts" and "confirmation numbers" which a sender
can use to identify themselves to a recipient. These are hashes of the TxOut shared secret, which
convince the recipient that whoever sent them this number must have access to secrets involved in the
construction of the TxOut. These confirmation numbers have a "deniability" property also shared by signal messages.
This arises because they are based on the TxOut shared secret, which can be formed by the sender OR the recipient.
(For more discussion, see the prior art section.)

However, these confirmation numbers are not useful for convincing a third-party who is not a party to the transaction
that a particular person sent the transaction, because only the transaction recipient is able to validate these numbers.
Even if the recipient gives away their private keys to a third party, this still doesn't work, because the confirmation
numbers have the deniability property, and the recipient could say that anyone had sent them the confirmation number.
(The hmac appearing in the memos in MCIP #4 couldn't have been created by anyone, but could have been created by the recipient.)
There is currently no form of receipt that can be validated by anyone, just from the blockchain.

This is desirable by some app project developers, who would like to store encrypted metadata per TxOut,
but off-chain. For example, they might like to have a database of encrypted metadata, keyed on TxOut's sent
by users of their app, and be able to ensure that only the true sender of the TxOut can post metadata associated
to the TxOut, without learning anything about the sender, recipient, amount, etc. of the TxOut.

After this proposal, a straightforward way to do this is that their app can sign the encrypted metadata using
the tx private key, and their server can check this signature against the public key in the encrypted fog hint.
We can simply use the Schnorrkel-based signature with Ristretto curve points that we already use in fog public addresses.

This might also be useful because it makes the notion of receipts somewhat more robust -- arbitrary metadata
can be signed by the sender, and this form of signature will not have the deniability property.
This happens because it is a traditional digital signature, based on a key that only the sender has.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

This MCIP will:

* Make it possible for the builder of a transaction output to additionally sign a blob
    using the tx private key.
* Make it possible for anyone to validate that signature against the TxOut, without
    knowing the identity of the sender.

# Reference-level explanation
[Reference-level-explanation]: #reference-level-explanation

We propose to

* Make internal changes to the transaction builder, following MCIP #18
  * Instead of being a second independent random private key, the private key used
    for `mc-crypto-box` encryption will be the same as the tx private key for that
    `TxOut`.
  * Also, implement code for a "Janus test" which enables clients to detect the
    Janus attack, as described in MCIP #18.
* Standardize a Schnorrkel-based signature using Ristretto curve points, made using the tx private key,
  and validated using the ephemeral key in the fog hint. This entails (one of):
  * Make the transaction builder take a blob to be signed when `add_output` is called.
    `add_output` produces the signature as an additoinal return value.
  * Make the transaction builder return a "signing context" when `add_output` is called.
    This contains the tx private key and can be used to later sign any number of blobs.
    This object zeroizes the tx private key when it goes out of scope.
    * The idea is, perhaps some of the data that needs to be signed needs to be obtained
      from an endpoint, but the app would rather not wait for that call to return before
      submitting the transaction. This way, the app could build and submit a transaction
      in parallel with obtaining some of the data to be signed, and pipeline calls,
      potentially reducing end to end transaction times. Since it's just a thought experiment,
      we don't know if it would matter in a real app.
* Standardize validation code that validates a blob and its signature against a TxOut.

# Prior art
[prior-art]: #prior-art

To recap on the deniability property:

* Deniable authentication means that, when the sender "signs" a message, the signature is made by a key exchange with the recipient.
  * This means that both the sender AND the recipient are technically capable of producing the signature.
  * The recipient who sees the signature, and knows they didn't produce the signature, concludes the sender must have sent it.
  * However, the recipient cannot convince another person that they did not forge the signature.
  * As a result, if the recipient shows the message to a third party (to embarrass the sender), the sender can deny having written
    the message, and claim that the recipient made the whole thing up.
* A [good intro](https://www.praetorian.com/blog/an-opinionated-series-on-why-signal-protocol-is-well-designed-deniability/)
* A (technical) Signal blog post on the topic: https://signal.org/blog/simplifying-otr-deniability/

This proposal touches on some earlier MCIPS:

* [0004-Recoverable-Transaction-History](https://github.com/mobilecoinfoundation/mcips/pull/0004)
* [0018-Janus-attack-mitigation](https://github.com/mobilecoinfoundation/mcips/pull/0018)

Outside of MobileCoin, there are many blockchain projects that want to store chain-related data off-chain.
This is often motivated by scaling considerations.

Popular strategies include:
* Embed hash of the stored data into the transaction
* Use a layer 2 rollup

This signature approach gives us a third possibility, which is relatively simple to implement,
where we store off-chain data with a signature which ties it back to the chain,
and don't have to embed a hash into the main chain.

That has several advantages:
* Adding a hash to the transaction bloats the chain, and requires a ledger format change.
* More composable. If we want to have multiple off-chain data stores, we cannot have multiple hashes
  because TxOut's must be fixed-size. We could have a little hash tree (so, a hash of hashes), but this
  requires that all the different off-chain data stores know about eachother in order to validate the
  data, which adds a lot of complexity to each one. By relying instead on signatures from the sender,
  off-chain data stores can all be fully independent.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

An alternative worth discussing is:

* Don't implement MCIP #18, don't make the ephemeral private key from the fog hint
  the same as the tx private key. Just make that private key available to form
  digital signatures with.

This meets the motivation of this MCIP, but since this takes us almost all the way
to MCIP #18, we think we should do MCIP #18 also, since it is valuable to mitigate
Janus attacks and that MCIP is well-studied. It costs us almost nothing to start
building transactions in the way described there, and at some point, flip the switch
on clients actually flagging Janus test failures as suspicious.

We do not believe right now that there is a simpler way to create this signature,
as the signer does not know the root of the `public_key` or `target_key` of `TxOut`'s
relative to the ristretto basepoint. The signer does know the root relative to some of
the public subaddress keys of the recipient, but such a signature cannot be verified
by someone who does not know who the recipient is.

The modification from MCIP #18 seems to be the simplest change we can make that makes
a signature like this possible, and that modification already has good motivations.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

None at this time.
  
# Future possibilities
[future-possibilities]: #future-possibilities

It seems likely that we could find other use-cases for this digital signature scheme,
for example, to be able to create receipts that don't have the deniability property,
and which sign additional metadata.
