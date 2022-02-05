- Feature Name: Recoverable Transaction History
- Start Date: (2021-06-18)
- RFC PR: [mobilecoinfoundation/mcips#4](https://github.com/mobilecoinfoundation/mcips/pull/4)
- MobileCoin Epic: None

This MCIP describes a memo which has had its size adjusted from 46 to 66 in MCIP #24. The byte tables have been updated to reflect this change.

# Summary
[summary]: #summary

We propose to specify three memo types that support a "recoverable transaction history" feature.

(See [mobilecoinfoundation/mcips#3](https://github.com/mobilecoinfoundation/mcips/pull/3) for a
description of memos and memo types.)

When a sender sends money to a recipient, the "sender memo" is attached to the TxOut that the
recipient recieves, and permits them to identify the sender. The "destination memo" is attached
to the change TxOut that the sender receives, and records details of the recipient and the amount sent.

In a mobile chat application, the use of these memos in payments permits the app to recover transaction
history given only the private keys of the user, by fetching all of the user's TxOut's from Fog and then
decoding and validating the memos.

Some additional ancillary parts of the proposal:
- Change subaddress is standardized to subaddress index 1.
- Clients should always produce a change output even if the change is zero, in order to have a place to put
  the destination memo.
- A standard 16-byte hash of a public address is specified, which may be useful elsewhere.
- A memo containing payment request id numbers is also introduced.

# Motivation
[motivation]: #motivation

One of the major goals of MobileCoin is to provide a best-in-class payments experience on mobile devices.

There are two aspects of this user experience that we hope to simplify and improve:

- In a mobile chat app, typically we want the app to be able to automatically associate payments with the contact
  that sent them to us. The existing mechanism for this uses the `TxConfirmationNumber`. The sender both sends
  a payment in the MobileCoin network, and sends a corresponding `TxConfirmationNumber` over the chat channel.
  Then when the recipient gets the TxOut from Fog, the app can match it to the confirmation number and
  know which chat to show the payment in. A difficulty here is that if the chat message is dropped, then we
  will never be able to associate the payment, which is a negative user experience. If instead we could use a
  memo on the payment to identify the sender, this eliminates that failure mode, and makes the payment appear
  faster since we only have to wait for one thing to happen rather than two things.
- In a payments app, it may be very important to preserve the transaction history of the user -- who they sent
  money to and who they recieved money from. The user may need
  this for budgeting, or to comply with reporting requirements, or to resolve a dispute. In bitcoin and ethereum,
  you can always recover this data from the chain if
  you have your public key. In privacy coins, typically you cannot, and so it falls to the app to help the user
  preserve this data. However, the only way the app can do this reliably is to back the data up in the cloud somehow, which
  has its own costs and complexities. Another way this can be done is to store the data in encrypted memos on the
  blockchain. If this data is in the blockchain memos, then all the user needs to get their
  history is their private keys, since they get all these memos with the TxOut's from Fog.
  If these memos are standardized, then
  when a user exports their private keys from one app to another, their transaction history comes with them.

Standard memos that preserve information about the sender and recipient of the transaction give developers more
tools to build a great user experience, and any users who don't want these memos can ignore them and use clients
configured to use the "Unused" memo everywhere, so this is a win for everyone.

Additionally, this proposal includes a memo which has a payment request id number. This has uses in several
payment flows.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

In the typical scenario, when a sender sends money to a recipient, there is one transaction output
that is created for the recipient, and a *change output* that the sender sends back to themselves.
The value of the change output is whatever is needed to balance the transaction, so that sum of the
value of the inputs equals the sum of the value of the outputs and fees.

To implement recoverable transaction history, we can store some information in the "normal" output memo,
which is read by the recipient, and some information in the change output memo, which is read by the sender.

The normal output contains a "sender" memo which preserves information about the sender, for the recipient.
The change output contains a "destination" memo which preserves information about the recipient, and the
amount that was sent to the recipient, and the fee.

## Sender memo

The sender memo, with type bytes `0x0100`, includes a hash of the sender's public address,
and an HMAC value. This is used to inform the recipient of who the sender is in a verifiable way.

The key to this HMAC is the result of a key exchange between the sender's
spend key, and the recipient's view key. It can be constructed (and verified) by either the sender, OR the
recipient.

This key exchange creates a deniability property. When the recipient verifies the HMAC, they know the sender must have
created this HMAC, since they know that they themselves didn't. However, a third party who doens't trust the recipient,
cannot be certain that the recipient did not create the HMAC themselves. Signal messages have a similar deniability
property.

Note that the sender can use any address to identify themself, so long as they know the private spend key of that address.

The sender memo with payment request id, with type bytes `0x0101`, is the same as `0x0100` except that the schema includes a
payment request id in the payload under the HMAC. (The payment request id is a 64-bit number which identifies a payment request.
This proposal does not specify how payment requests work, but is compatible with many realistic proposals for that.)

To prevent a malicious party from confusing the recipient, a client should not associate a payment to a sender automatically
unless the HMAC check passes.

## Destination memo

The destination memo, with type bytes `0x0200`, includes a hash of the recipient's address (if more than one recipient, then
an arbitrary one). It also includes a count of the number of recipients, and the total outlay and fee paid.

If there is more than one recipient, only the address hash of one of them can be stored in the space available.

The destination memo does not have a MAC, however it can be validated by confirming that the associated TxOut went to the
change subaddress for this account.

To prevent a malicious party from confusing the sender, a client should ignore this memo unless it is on a TxOut
that went to the change subaddress. 

## Change subaddress

The "change subaddress" is a new standard subaddress, like the default subaddress. It should be used when sending change outputs.
It is specified to be subaddress index 1.

The change subaddress should be kept secret, so that only authentic 0x0200 Destination memos will pass validation.

To fully support recoverable transaction history, clients should use 0x0100 or 0x0101 Authenticated sender memos with all
outputs that they send to another party, and 0x0200 Destination memos on all change outputs, which are all sent to the change
subaddress. Change outputs should be sent *even* in the case that the change value is zero, in order that there will be a destination memo.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

A 16-byte *short address hash* is specified:

```
address_hash = public_address.digest32(“mc-address-hash”)[0..16]
```

Here `digest32` refers to the merlin-based hash computed using `mc-crypto-digestible` crate.

The *change subaddress* is specified as the subaddress of index 1. It should not
be used except for change outputs, and this subaddress should not be shared with third parties.

All transactions should have exactly *one* change output, even if the change value is zero.

Three new memo types are specified:

| Memo type bytes | Name                                              |
| -----------     | -----------                                       |
| 0x0100          | Authenticated Sender Memo                         |
| 0x0101          | Authenticated Sender With Payment Request Id Memo |
| 0x0200          | Destination Memo                                  |

## 0x0100 Authenticated Sender Memo

The 64-byte memo data is laid out as follows:

| Byte range | Item |
| ---------- | ---- |
| 0 - 16     | Sender's Address Hash |
| 16 - 28    | Unused bytes          |
| 28 - 44    | HMAC                  |
| 44 - 64    | Unused bytes          |

The HMAC is computed using HMAC-SHA512. 
(HMAC refers to [RFC 2104 HMAC](https://datatracker.ietf.org/doc/html/rfc2104).)
(SHA512 refers to the 512 bit variant of [SHA-2](https://en.wikipedia.org/wiki/SHA-2).)

The key to this HMAC is the 32-byte shared secret computed from Ristretto key exchange
using the Sender's spend key, and the Recipient's view key. (For the sender, we multiply
the senders private spend key corresponding to the subaddress they wish to identify using,
by the view public key from the recipient's public address. For the recipient, we multiply
the view private key of the subaddress that recieved the payment by the spend key
from the public address that the sender used to identify themselves.)

The text to this HMAC is the concatenation of:
- The domain separation string "mc-memo-mac"
- The 32-byte public key of the transaction output to which this memo is attached
- The memo type bytes
- The first 28 bytes of the memo data (all of the data except the HMAC bytes).

The HMAC output is truncated to 16 bytes.

This HMAC text is computed in the same way for all category 0x01 memos.

Clients should interpret this memo as indicating who sent them this payment,
once it has been validated. If the address hash is unknown, or the validation fails,
then they should not associate this payment with a sender.

## 0x0101 Authenticated Sender With Payment Request Id Memo

The 64-byte memo data is laid out as follows:

| Byte range | Item |
| ---------- | ---- |
| 0 - 16     | Sender's Address Hash |
| 16 - 24    | Big-endian bytes of 8-byte payment request id number |
| 24 - 28    | Unused bytes          |
| 28 - 44    | HMAC                  |
| 44 - 64    | Unused bytes          |


The HMAC is computed in the same way as for 0x0100 Authenticated Sender Memo.

Clients may interpret this memo in the same way as the 0x0100 Authenticated Sender Memo,
and additionally conclude that the sender wished to associate this payment to a specific
payment request.

There is an additional way that the client can handle this memo which we wish to call out:

In some payment flows, Bob would like to request payment from Alice, but Bob does not
yet have Alice's public address. We propose that as long as Bob chooses a (secure) uniformly random
64-bit payment request id and includes this in the payment request that he sends to Alice,
Bob's client may conclude that a payment came from Alice based on matching the value in
the payment request id field, even if he does not check the HMAC (he cannot do that
if he doesn't yet have Alice's public address). Later, if Bob's device is lost, he may
lose track of these payment request id numbers, but can still associate this payment to Alice
by validating the memo along the normal pathway if at that later time she has entered his contacts.

In such a case, it is acceptable to interpret the memo and access the payment request id
number, even if we haven't validated the memo by confirming the HMAC.

## 0x0200 Destination Memo

The 64-byte memo data is laid out as follows:

| Byte range | Item |
| ---------- | ---- |
| 0 - 16     | Recipient's Address Hash |
| 16 - 17    | The number of recipients, as an unsigned 8-bit number |
| 17 - 24    | Big-endian bytes of fee amount, as an unsigned 56-bit number |
| 24 - 32    | Big-endian bytes of the total outlay amount, as an unsigned 64-bit number |
| 32 - 64    | Unused bytes                  |

Here, the "total outlay" means the total amount that this transaction is deducting
from the Sender's balance. It is the sum of the value of all non-change outputs,
and the fee.

The destination memo imposes some limits:
* If the number of recipients exceeds 255 then the memo cannot be used. Since there are only 16 outputs
  per transaction at most at time of writing, this is a safe assumption.
* If there is more than one recipient, only one address hash can be recorded.
  The client should choose one of them.
* It is assumed that the fee, when represented as a 64-bit number in big endian bytes, has
  a high order byte which is 0. If the fee is larger than this, it implies that on the order
  of 1% of the total supply of MOB are being spent in the fee. This is considered implausible,
  but technically possible.
  In this case, the Destination memo cannot be used.
* It is assumed that the total outlay, when represented as a 64-bit number, does not overflow.
  It is technically possible for this value to overflow. In that case, the Destination memo cannot
  be used.

The 0x0200 Destination Memo does not have a MAC. Instead, it is ONLY used by the sender
on a change output, which is sent to the (secret) change subaddress.

Clients should only consider this memo valid if the transaction output that it is attached
to matches the *change subaddress*. This prevents a malicious party from confusing the user
using malicious destination memos.

Clients should interpret this memo as recording that they sent a certain amount to a certain
party (at a time determined by the timestamp associated to the change output).

## New Memo type categories

Besides category 0x00 (Unvalidated) which contains only the Unused memo at time of writing, we now have:

| Memo category byte | Name        | Validation |
| -----------        | ----------- | ---------- |
| 0x01               | HMAC        | Last 16 bytes of memo data are HMAC-SHA512 computed as documented earlier |
| 0x02               | Change      | Output must match to the change subaddress for this memo to be valid |

# Drawbacks
[drawbacks]: #drawbacks

One drawback of 16 byte address hashes is that an attacker can likely find a collision by hashing about 2^64 addresses,
which is feasible within our threat model.

However, to obtain an exploitable collision, this collision has to involve an address which
belongs to a real person. This requires hashing about 2^128 addresses, which is similar in difficulty to brute-forcing
our other cryptographic primitives.

Additionally, finding a hash collision won't enable the attacker to bypass the HMAC check, it would just mean that the
client has multiple hits for an address hash and must check HMAC against several addresses.

Using a longer hash for the address hashes would require more space in the memo, and likely wouldn't help anything.

One drawback of using an address hash is that if a later MCIP proposes to add new fields to the public address, the hash will change.
This will require a more complicated versioning story for public addresses.

However, the only way to get an appropriate identifier for public address for our purpose is likely hashing them.
The future MCIP that introduces new fields to the public address may perhaps specify that the address hash will not include the new fields,
to avoid this issue.

One drawback of requiring that the change subaddress is secret in an MCIP like this is that a client may concievably have already revealed
the change subaddress to someone. However, it turns out that in practice, clients were already using the change subaddress this way, and
we are just standardizing existing convention.

One drawback of the Authenticated Sender memo design is that it doesn't have a full digital signature from the Sender, and instead uses
this somewhat weaker HMAC-over-key-exchange mechanism. However, the HMAC mechanism results in a significantly shorter signature, and it
achieves our deniability goal, which a full digital signature would not.

One drawback of the Authenticated Sender With Payment Request Id memo design is that the payment request id is limited to 8 bytes. However,
we believe that this will be adequate in practice.

One drawback of the Destination Memo design is that it imposes numerous limits on the total outlay, fee, number of recipients, etc.
However, we believe that this is more than adequate for the most important payments use-cases.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

## Alt: Do nothing, and storing transaction history is an app-level concern

Each app is certainly capable of storing the transaction history of the user, and could back it up in an encrypted form in the cloud.

The benefits of having a standardized, on-chain way of doing this with encrypted memos are:
* Depending on the nature of the app, the app provider may be unable or unwilling to provide that
* If you move from app to app, the history comes with you
* Cross-app interactions will work. The status quo is that the sender sends a `TxConfirmationNumber` over a secure channel to the recipient,
  but if the sender and recipient are on different apps, then there is no such channel. Similarly, if an exchange is running mobilecoind,
  then it is impractical for them to send a `TxConfirmationNumber` over a chat channel to the recipient, for all of the numerous chat apps
  that their users are using. After this change, the exchange can simply use an `AuthenticatedSender` memo to identify themselves to the
  recipient, if they desire, so that the user's client can associate all payments from the exchange into one interaction. The exchange can
  identify themselves in this memo using whatever subaddress makes the most sense for that recipient.

## Rationale

This is likely the simplest proposal that achieves our goals regarding recoverable transaction history,
and will provide us with more tools to craft an excellent user experience on mobile devices.

# Prior art
[prior-art]: #prior-art

Memo types in Stellar: https://developers.stellar.org/docs/glossary/transactions/#memo

Payment ids in Monero: https://www.getmonero.org/resources/moneropedia/paymentid.html

Signal on Deniability: https://signal.org/blog/simplifying-otr-deniability/

We are not aware of privacy coins that have standardized the use of memos for transaction history in this manner.

The author would like to thank Koe for their work on a draft version of this proposal,
and for many contributions that helped shape the proposal.

The author would like to thank Kyle Fleming for many contributions
that helped shape the proposal.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- There are no unresolved questions which should be resolved by this RFC.

# Future possibilities
[future-possibilities]: #future-possibilities

We hope that future MCIPs will propose additional useful memo types to further enrich the recoverable transaction history.
