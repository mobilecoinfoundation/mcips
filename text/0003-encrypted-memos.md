- Feature Name: Encrypted Memos
- Start Date: (2021-06-18)
- MCIP PR: [mobilecoinfoundation/mcips#3](https://github.com/mobilecoinfoundation/mcips/pull/3)

# Summary
[summary]: #summary

We propose to extend mobilecoin transaction outputs to include an "encrypted memo" field of a
fixed, 46 byte length. These memo fields have a standard encryption scheme and can be read by
the sender and recipient, but no one else.

These memos are intended to hold data, rather than human-readable text. (Although that is one possible use.)
We propose an extensible method for specifying a schema for the plaintext data in the memo.
We propose that memo types are standardized via MCIP proposals, so that clients can interoperate
seamlessly.

# Motivation
[motivation]: #motivation

Memo fields, which are uninterpretted by consensus and pass-through to the blockchain, offer a way of attaching metadata permanently to transaction outputs. There are many use-cases for payments that are simplified by the ability to do this, and many blockchains support memos.

In some blockchains, memos are simply human-readable text displayed directly to the user.
However, one major consideration is how memos affect the privacy of transactions.
In order to ensure output uniformity, it is desirable to have these memos encrypted, such
that only the sender and recipient can read them.

Additionally, the use-cases we are most interested in are not human-readable text, but rather,
specific data about the payment which is encoded according to a schema. This includes data
that helps a user figure out who a payment is from or what it is for.

In order to make the memo feature maximally useful, we would like:

1. A standard encryption scheme for memos, so that all encrypted memos are decrypted in the same way regardless of schema.
2. A standard way of determining the schema, so that clients can interoperate seamlessly.
3. The memos are passed through both Consensus and Fog uninterpreted and unchanged. Only Alice and Bob can read the memo.
4. The memos do not substantially impact the performance of MobileCoin Consensus or Fog.

We determined that 46 bytes was the largest we could make the memo before it will start to negatively impact Fog,
and that this was also more than adequate for use-cases that were considered most interesting.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The `e_memo` (short for "encrypted memo") is a new fixed-size field on every MobileCoin TxOut.

The `e_memo` data is encrypted using AES, with a key specific to the TxOut. Only the sender and recipient can decrypt the memo.

The plaintext bytes have the following interpretation:
- First 2 bytes indicate "memo type".
- Remaining bytes indicate the "memo data". These can only be interepretted in the context of the memo type, and their semantics and interpretation should be specified in an MCIP.

When the memo type bytes are both zero, this indicates an "Unused" memo, regardless of the contents of the memo data bytes.

Additional memo types / schemas can be specified by MCIPs, and their assigned memo type bytes shall be unique, so that all clients conforming to the MCIPs
may unambiguously interpret the contents of memos. This promotes interoperability among clients.

Memo category 0x00 consists of unvalidated memos. (Memos with no defined validation mechanism.)

To ease the release of the change, there will be an interim period during which `e_memo` field is optional.
After a period of time, memos will become mandatory and consensus will be modified to reject any transaction
containing a `TxOut` that doesn't have a memo. This will enforce output uniformity, which improves the privacy of users.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The `e_memo` field is an optional field on every MobileCoin TxOut, starting with BLOCK_VERSION = 1.

After an interim period, a release will be made that makes memos mandatory.
(We will update this document to reflect when the interim period has ended.)

The release that makes memos mandatory will result in a new MRENCLAVE value for consensus, but not a new
`BLOCK_VERSION` as it is not a breaking change to the ledger format.

The `e_memo` field, when present, is exactly 46 bytes long. Exactly 46 bytes of plaintext are selected when the TxOut is constructed, and then encrypted.

Encryption is performed using AES with a secret key derived from the TxOut shared secret.

A 48-byte memo-okm value is defined as:

```
memo_okm = 
	hkdf-sha512("mc-memo-okm",  tx_out_shared_secret_bytes).expand("", 48)
```

Here okm means "output key material", see RFC5869 HKDF. (Specifically: The input key material to hkdf is the canonical bytes of the `tx_out_shared_secret`. The salt is the string “mc-memo-okm”. The info-string is empty, when expanding to a 48 byte secret with hkdf.) (SHA512 refers here to the 512-bit variant of SHA-2 hashing standard.)

The first 32 bytes are interpreted as an AES key, and remaining 16 bytes as an AES nonce.

```
memo_aes_key = memo_okm[0..32]
memo_aes_nonce = memo_okm[32..48]
```

A 46-byte `memo_payload` consists of:


| Byte range | Item |
| ---------- | ---- |
| 0 - 2      | `memo_type` |
| 2 - 46     | `memo_data` |

The memo payload data can only be interpreted given the memo payload type.

A 46-byte encrypted-memo is computed by AES-256 in counter mode:

```
e_memo = AES256Ctr(memo_payload, memo_aes_key, memo_aes_nonce)
```

Because only the sender and recipient can know the TxOut shared secret, only the sender and recipient can decrypt the memo.

The first byte of the `memo_type` indicates a "memo type category", and the second byte indicates a type within the category.

Each memo type may specify additional steps for the recipient to validate the `memo_data` as necessary.

Memo type categories should typically consist of memos with the same or similar validation, which may also be extensions of one another.
This is meant to help promote reuse of the memo validation code in implementations.

Additional memo categories and types can be specified by MCIPs, and their assigned memo type bytes shall be unique, so that all clients conforming to the MCIPs
may unambiguously interpret the contents of memos. This promotes interoperability among clients.

Memo category 0x00 consists of unvalidated memos. (Memos with no defined validation mechanism.)

Memo type 0x0000 indicates the Unused memo. Its memo data shall not be interpretted by the client.

# Drawbacks
[drawbacks]: #drawbacks

One drawback of this approach is that it commits us to a short, fixed size payload rather than a variable-length one. However, it would
be impossible to reconcile variable-length memo fields with our privacy goals, since we would give up on output uniformity. It is also impossible
to serve variable-length payloads via Oblivious RAM -- the only way to do this is to pad up to some maximum.

Another drawback of this approach is that instead of using a well-developed serialization format like protobuf, we are creating a byte-oriented protocol
for reading and writing the memos, which is more complicated to implement. 

Another drawback of this proposal is that it creates a ledger format change, which means that clients must be updated so that they will hash the new data correctly.
But, it seems to be an unavoidable consequence -- there is no way that we can add new features to the blockchain like this without a ledger format change, if we want
this new data to be under the hash, and we generally want to put all transaction data under the hash to prevent tampering.

Another drawback is that, the additional data about TxOut's may be sensitive, and some users would not want it to be stored in the blockchain forever, even if it is encrypted.
However, the memos are completely optional, and those users can use clients configured not to make use of the memos.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

## Alt: Use a longer memo size

The justification for 46 bytes comes from an analysis of the impact that increasing this size has on the fog-view service.

The fog-view enclave uses an Oblivious Map to privately map from fog-search-keys (16 byte outputs from a KexRng) to ETxOutRecords.

This is implemented as a special form of Cuckoo hash table using buckets, where each bucket is a value served by an underlying Oblivious RAM.

After Circuit ORAM is implemented (scheduled for this quarter), we believe the optimal value size will be 2048. So that will be the bucket size in the hash map.

The hashmap divides each bucket into as many `KeySize + ValueSize` pairs it can, which are consecutive ranges of bytes with no padding at all.
These sizes are fixed at compile-time for the enclave.

(See in code: https://github.com/mobilecoinofficial/mc-oblivious/blob/2b620343e27f652d5efeda5af248901436f06832/mc-oblivious-map/src/lib.rs#L187)

However, the `ETxOutStore` `in fog-view-enclave-impl` is meant to allow that some `ETxOutRecord`'s could be shorter than others. This would happen if we made a ledger format
change and added a field, then the older `ETxOutRecord`'s are shorter. To achieve this, the value that it stores in the Oblivious Map is encoded, so that the first byte to say how many bytes
less than the maximum the payload is, and rest of the bytes are the bytes of the `ETxOutRecord`.

(See in code: https://github.com/mobilecoinfoundation/fog/blob/817d151c1035aaa32458dd281fd10cbd33848ca5/fog/view/enclave/impl/src/e_tx_out_store.rs#L108)

The upshot is that the number items that we can fit into a bucket will be `2048 / (16 + 1 + sizeof(ETxOutRecord))` rounded down to the nearest integer.

When this number gets smaller, it impacts memory utilization both because we can't fit as many items in a bucket, but also because, when the buckets can
hold fewer items, the Cuckoo hashing algorithm requires more cuckoo steps to achieve the same memory utilization as before, which means we either accept
more steps and more Oblivious RAM operations to support a single query, or we accept a further reduction in memory utilization. It is difficult to accurately
predict the magnitude of this effect using a pencil and paper, and in practice we measure it empirically.

With 46 byte memos, we measure that the size of an `ETxOutRecord` is 207 bytes:

(See in code: https://github.com/mobilecoinfoundation/fog/pull/122/files#r688751880)

This chart illustrates the breakpoints and what the points of maximum memory utilization are:

| size of `ETxOutRecord` | `2048 / (16 + 1 + sizeof(ETxOutRecord)` |
| ---------------------- | --------------------------------------- |
| 187                    | 10.0392156863 |
| 188                    | 9.99024390244 |
| 207                    | 9.14285714286 |
| 210                    | 9.02202643172 |
| 211                    | 8.98245614035 |
| 239                    | 8.0           |

Before this revision, the size of `ETxOutRecord` is 159 bytes (and this does not include protobuf
overhead for representing the `e_memo` field.)

With 26 byte memos, we would have the size of `ETxOutRecord` at 187 and hit the 10 threshold.
But with this size, we would not be able to use multiple cryptographic primitives
with 128-bit security in a single memo, which is what we would like to be able to do in applications.

In fact, a parallel change which is planned for release in 1.2 is reducing the size of an `ETxOutRecord`
by about 28 bytes. (This is the change that omits the compressed commitment from the amount data in a `TxOut`
that fog-view returns to the user, since the user can reproduce this anyway.) This means that the first 28 bytes of memo
are not adding any burden to Fog relative to the status quo.

An `ETxOutRecord` size of 207 is actually 3 bytes below the threshold for 9. So, we are leaving 3 bytes
on the table if we choose 46 byte memos.

We could therefore justify increasing the `e_memo` by a few bytes, so that there are a few more bytes for future use in the memo.
(It seems likely that we made a slight error when planning the change in calculating how much smaller the `ETxOutRecord` would get
due to the optimization that omits the compressed commitment.)

OTOH, it seems okay to leave a small fudge factor in case there is something we overlooked
in the enclave code or some other need that arises.

## Alt: Just use protobuf

Typically, it is better from an engineering point of view to use an existing serialization format rather than create a new format for data.
For example we could have a protobuf enum over the possible memo types.

The main justification to avoid protobuf is that we want to make as effective use as possible of the space of the memo, and using an off-the-shelf approach like protobuf would create overheads that would prevent us from storing all the data that we want to in 46 bytes.

Nothing stops a memo type from being introduced that uses protobuf to encode its memo data.

## Alt: Use AES-GCM

Typically, it is better to use an AEAD than to use a stream cipher like AES in counter mode. For example, we could have selected AES-GCM, and then we have
some additional resistance to an adversary tampering with the data.

However, this adds an additional 16 bytes for the MAC to the encrypted memo, and would prevent us from storing all the data that we want to store in the memo
without making the memo larger.

Additionally, the authentication provided by that MAC is fairly weak, since it only ensures that the memo came from someone who knew the shared secret.
In some memos, we actually want to authenticate that the sender knows the private keys corresponding to some MobileCoin public address.
For this reason, we leave the selection of a MAC and what is being MAC'ed to the specification of the memo type, and only the encryption part is standardized.

One concern when we MAC and then encrypt, rather than encrypting and then applying a MAC, is that some form of the Vaudenay attack may occur. However, because
these memos are fixed length, and an incorrect length is an immediate deserialization error, this turns out to be a non-issue.

Additionally, the key used to encrypt here is derived in a one-off manner from the transaction output's shared secret, and is not used for anything else, so
that aspect of the attack scenario also doesn't apply.

## Alt: Memo type bytes are not encrypted

Another option would be that the memo type bytes are not encrypted, and instead each memo type specifies both encryption and MAC together, and so could potentially
use an AEAD.

The main drawback of this is that it leaks the memo type to the world, and weakens output uniformity.

If in the future we desired to pivot to a strategy like this, one approach would be to add additional `e_memo` fields in the protobuf encoding of a TxOut and make it
implicitly an enum. This would compress the memo type bytes into the protobuf tag when the TxOut is encoded. So, this proposal does not rule that out as a future direction.
However, lacking any compelling reason to choose this alternative, we prefer to encrypt the memo type bytes.

A second drawback of this approach is even if we did this and we used an AEAD to encrypt, in several intended use-cases we still would need to have an extra MAC value.
For example: In one desired memo type, the sender sends a hash of their public address, and then a MAC value over a key which depends on their identity.
The recipient needs to look up the public address based on the hash, and then they can derive the key that is appropriate for the MAC and confirm that their identity
is tied to this memo.

Clearly, the address hash needs to be encrypted somehow, to meet the privacy goal. And the key for that cannot depend on the sender's identity, because the recipient
doesn't know that before they decrypt the memo.
If that encryption is an AEAD mode, then we have one MAC for that implicitly. But then we still need an additional MAC that depends on a key that is tied to the sender to meet the
authenticity goal of the memo. So, the use-case really needs that encryption is happening with a different key than authentication.
Any version of this that uses an AEAD results in a redundant MAC and doesn't fit in 46 bytes if we use primitives secure at a 128 bit security level.

## Rationale

This is likely the simplest proposal that supports the memo types that we are most interested in, and reserves as much space as possible for future uses that we didn't anticipate, before starting to impact the performance of Fog / the cost of running Fog.

# Prior art
[prior-art]: #prior-art

ZCash on their 512-byte encrypted memo: https://electriccoin.co/blog/encrypted-memo-field/

Memo types in Stellar, with space for 28 bytes of text, or a 32 byte hash: https://developers.stellar.org/docs/glossary/transactions/#memo

Bitcoin and Ethereum do not have memos.

Monero discussion on removing "tx_extra": https://github.com/monero-project/monero/issues/6668
which is unstructured data attached to transactions, not of a fixed size or encrypted in a standard way.

The author would like to thank Koe for their work on a draft version of this proposal,
and for many contributions that helped shape the proposal.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- There are no unresolved questions which should be resolved by this MCIP.

# Future possibilities
[future-possibilities]: #future-possibilities

If compelling use-cases were identified, we could adopt a larger memo size than 46 bytes in the future. After more engineering work has been done to improve the performance of the oblivious RAMs used in Fog, we may be able to justify the potential performance impact of having larger memos.

We hope that future MCIPs will propose additional useful memo types for standardization.
