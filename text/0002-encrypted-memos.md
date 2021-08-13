- Feature Name: Encrypted Memos
- Start Date: (2021-06-18)
- RFC PR: [mobilecoinfoundation/rfcs#3](https://github.com/mobilecoinfoundation/rfcs/pull/3)
- MobileCoin Epic: None

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

In some blockchains, memos are simply human readable text displayed directly to the user.
However, one major consideration is how memos affect the privacy of transactions.
In order to ensure output uniformity, it is desirable to have these memos encrypted, such
that only the sender and recipient can read them.

Additionally, the use-cases we are most interested in are not human-readable text, but rather,
specific data about the payment which is encoded according to a schema. This includes data
that helps a user figure out who a payment is from or what it is for.

In order to make the memo feature maximally useful, we would like:

(1) A standard encryption scheme for memos, so that all encrypted memos are decrypted in the same way regardless of schema.
(2) A standard way of determining the schema, so that clients can interoperate seamlessly.
(3) The memos are passed through both Consensus and Fog uninterpreted and unchanged. Only Alice and Bob can read the memo.
(4) The memos do not substantially impact the performance of MobileCoin Consensus or Fog.

We determined that 46 bytes was the largest we could make the memo before it will start to negatively impact Fog,
and that this was also more than adequate for use-cases that were considered most interesting.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The `e_memo` (short for "encrypted memo") is a new fixed-size field on every MobileCoin TxOut.

The `e_memo` data is encrypted using AES, with a key specific to the TxOut. Only the sender and recipient can decrypt the memo.

The plaintext bytes have the following interpretation:
- First 2 bytes indicate "memo type".
- Remaining bytes indicate the "memo data". These can only be interepretted in the context of the memo type, and their semantics and interpretation SHOULD be specified in an MCIP.

When the memo type bytes are both zero, this indicates an "Unused" memo, regardless of the contents of the memo data bytes.

Additional memo types / schemas can be specified by MCIPs, and their assigned memo type bytes SHALL unique, so that all clients conforming to the MCIPs
may unambiguously interpret the contents of memos. This promotes interoperability among clients.

Memo category 0x00 consists of unvalidated memos.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The `e_memo` field shall appear on every MobileCoin TxOut, starting with BLOCK_VERSION = 1.

It is exactly 46 bytes long. Exactly 46 bytes of plaintext are selected when the TxOut is constructed, and then encrypted.

Encryption is performed using AES with a secret key derived from the TxOut shared secret.

A 48-byte memo-okm value is defined as:

```
memo_okm = 
	hkdf-sha512("mc-memo-okm",  tx_out_shared_secret_bytes).expand("", 48)
```

Here okm means "output key material", see RFC5869 HKDF. (Specifically: The input key material to hkdf is the canonical bytes of the tx_out_shared_secret. The salt is the string “mc-memo-okm”. The info-string is empty, when expanding to a 48 byte secret with hkdf.)

The first 32 bytes are interpreted as an AES key, and remaining 16 bytes as an AES nonce.

```
memo_aes_key = memo_okm[0..32]
memo_aes_nonce = memo_okm[32..48]
```

A 46-byte memo-payload consists of:

```
Bytes [0-2]: "memo_payload_type"
Bytes [2-46]: "memo_payload_data"
```

The memo payload data can only be interpreted given the memo payload type.

A 46-byte encrypted-memo is computed by AES-256 in counter mode:

```
encrypted_memo = AES256Ctr(memo_payload, memo_aes_key, memo_aes_nonce)
```

Because only the sender and recipient can know the TxOut shared secret, only the sender and recipient can decrypt the memo.

The first byte of the "memo_payload_type" indicates a "memo type category", and the second byte indicates a type within the category.

Each memo type may specify additional steps for the recipient to validate the memo data as necessary.

Memo type categories should typically consist of memos with the same or similar validation, which may also be extensions of one another.
This is meant to help promote reuse of the memo validation code in implementations.

Additional memo categories and types can be specified by MCIPs, and their assigned memo type bytes SHALL unique, so that all clients conforming to the MCIPs
may unambiguously interpret the contents of memos. This promotes interoperability among clients.

Memo category 0x00 consists of unvalidated memos.

Memo type 0x0000 indicates the Unused memo. It's memo data shall not be interpretted by the client.

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

## Alt: Just use protobuf

Typically, it is better from an engineering point of view to use an existing serialization format rather than create a new format for data.
For example we could have a protobuf enum over the possible memo types.

The main justification to avoid protobuf is that we want to make as effective use as possible of the space of the memo, and using an off-the-shelf approach like protobuf would create overheads that would prevent us from storing all the data that we want to in 46 bytes.

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

## Alt: Memo type bytes are not encrypted

Another option would be that the memo type bytes are not encrypted, and instead each memo type specifies both encryption and MAC together, and so could potentially
use an AEAD.

The main drawback of this is that it leaks the memo type to the world, and weakens output uniformity.

If in the future we desired to pivot to a strategy like this, one approach would be to add additional `e_memo` fields in the protobuf encoding of a TxOut and make it
implicitly an enum. This would compress the memo type bytes into the protobuf tag when the TxOut is encoded. So, this proposal does not rule that out as a future direction.
However, lacking any compelling reason to choose this alternative, we prefer to encrypt the memo type bytes.

## Rationale

This is likely the simplest proposal that supports the memo types that we are most interested in, and reserves as much space as possible for future uses that we didn't anticipate, before starting to impact the performance of Fog / the cost of running Fog.

# Prior art
[prior-art]: #prior-art

We are not aware of useful prior art on this, or of any other blockchains that have created standardized encrypted memos.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- There are no unresolved questions which should be resolved by this RFC.

# Future possibilities
[future-possibilities]: #future-possibilities

If compelling use-cases were identified, we could adopt a larger memo size than 46 bytes in the future. After more engineering work has been done to improve the performance of the oblivious RAMs used in Fog, we may be able to justify the potential performance impact of having larger memos.

We hope that future MCIPs will propose additional useful memo types for standardization.
