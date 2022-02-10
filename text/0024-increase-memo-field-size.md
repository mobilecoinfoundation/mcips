- Feature Name: (`Recalculate appropriate memo size`)
- Start Date: (2022-02-02)
- MCIPS PR: [mobilecoinfoundation/mcips#24](https://github.com/mobilecoinfoundation/mcips/pull/24)
- MobileCoin Epic: Encrypted Memos

# Summary
[summary]: #summary

The size of the memo currently is 46 bytes, and we would like to increase the size of the memo to 66 bytes, and standardizing items per bucket in the Oblivious Map used by the fog-view enclave at 8.

# Motivation
[motivation]: #motivation

Current space in the bucket outside of that being used for the memo does not leave enough additional space for tokens and other upcoming features without impacting fog performance by moving memory size into the next bucket. Therefore, since we anticipate this performance hit, we would like to expand the memo appropriately so that we uniformly standardize to the next bucket while leaving space for those features. This will also provide additional space in the memo for storing interesting features for the user such as elliptic curve points, as desired.

We determined that 66 bytes was the largest we could make the memo with minimal negative impact on Fog, and that this was also more than adequate for use-cases that were considered most interesting. 

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

This is an expansion of the memo data filed from the [encrypted memo feature](0003-encrypted-memos.md#alt-use-a-longer-memo-size)

This provides more space in the memo beyond those used for recoverable transaction history so additional details can be added to the memo.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The [e_memo field](0003-encrypted-memos.md#reference-level-explanation
) is an encrypted memo field on every MobileCoin TxOut.

## This is an update to the values from #3:

The `e_memo` field, when present, is exactly 66 bytes long. Exactly 66 bytes of plaintext are selected when the TxOut is constructed, and then encrypted.

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

A 66-byte `memo_payload` consists of:


| Byte range | Item |
| ---------- | ---- |
| 0 - 2      | `memo_type` |
| 2 - 66     | `memo_data` |

The memo payload data can only be interpreted given the memo payload type.

A 66-byte encrypted-memo is computed by AES-256 in counter mode:

```
e_memo = AES256Ctr(memo_payload, memo_aes_key, memo_aes_nonce)
```

The fog-view enclave uses an Oblivious Map to privately map from fog-search-keys (16 byte outputs from a KexRng) to ETxOutRecords. The hashmap divides each bucket into as many KeySize + ValueSize pairs it can, which are consecutive ranges of bytes with no padding at all. These sizes are fixed at compile-time for the enclave.

As seen in the [original proposal](0003-encrypted-memos.md#alt-use-a-longer-memo-size) for encrypted memos, the number of items that we can fit into a bucket is `2048 / (16 + 1 + sizeof(ETxOutRecord))` rounded down to the nearest integer.

With our current 46 byte memos, and an anticipated additional 6 byte Token field, we measure that the size of the `ETxOutRecord` is 213 bytes.

This chart illustrates the breakpoints and what the points of maximum memory utilization are:

| size of `ETxOutRecord` | `2048 / (16 + 1 + sizeof(ETxOutRecord)` |
| ---------------------- | --------------------------------------- |
| 187                    | 10.0392156863 |
| 188                    | 9.99024390244 |
| 207                    | 9.14285714286 |
| 210                    | 9.02202643172 |
| 211                    | 8.98245614035 |
| 213                    | 8.90434782609 |
| 233                    | 8.192         |
| 239                    | 8.0           |

While maintaining 46 byte memos, we would be slightly above the threshold for 9. Therefore we are leaving 26 bytes on the table if we continue to use 46 byte memos. By increasing memo size to 66, we would still be 6 bytes below the threshold for 8, even with additional 6 bytes being used for Tokens. This is consistent with previous policy to leave a small fudge factor in case there is something we overlooked or another need that arises.


## This is an update to #4
### 0x0100 Authenticated Sender Memo

The 64-byte memo data is laid out as follows:

| Byte range | Item |
| ---------- | ---- |
| 0 - 16     | Sender's Address Hash |
| 16 - 48    | Unused bytes          |
| 48 - 64    | HMAC                  |

The HMAC is computed using HMAC-SHA512. 
(HMAC refers to [RFC 2104 HMAC](https://datatracker.ietf.org/doc/html/rfc2104).)
(SHA512 refers to the 512 bit variant of [SHA-2](https://en.wikipedia.org/wiki/SHA-2).)

### 0x0101 Authenticated Sender With Payment Request Id Memo

The 64-byte memo data is laid out as follows:

| Byte range | Item |
| ---------- | ---- |
| 0 - 16     | Sender's Address Hash |
| 16 - 24    | Big-endian bytes of 8-byte payment request id number |
| 24 - 48    | Unused bytes          |
| 48 - 64    | HMAC                  |

The HMAC is computed in the same way as for 0x0100 Authenticated Sender Memo.

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

# Drawbacks
[drawbacks]: #drawbacks

This expansion of the memo field without immediate usecase could leave inadequate quantities of space in the ETxOutRecord for other use cases, and it can be difficult to reclaim this space, resulting in bloat. 

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

## Alt: Use a smaller memo size

Alternatively we could try reducing the memo size by 3 bytes. This would allow the 6 bytes required for tokens to use the 3 additional bytes unused and the 3 bytes removed from the memo field to be provided to the tokens. This change would impact recoverable transaction history memos which although no particular memo type uses all 44 bytes of the memo data field, does require all 44 bytes to be used between the 3 memo types. This is therefore, not recommended.

## Alt: Increase memo size even further

Increasing memo size to the next item per bucket level would impact the Omap performance, and would not be recommended for this reason. 

Increasing memo size to the limit of allowed size for the EtxOutRecord for this bucket level would leave no room for error in requirements for the tokens and other features which are in development. Therefore it is desireable to leave a small fudge factor.

## Alt: Move to a variable length memo size

This would go against our privacy goals because variable length memo fields would leak meta data and also cause issues when using ORAM.

## Rationale

This is a minimal change that aligns and standardizes our number of items per bucket to 8, and reserves as much space as possible for future uses that we didn't anticipate, before starting to impact the performance of Fog / the cost of running Fog given that we anticipate additional space per ETxOutRecord being used for Tokens. This additional space provides flexibility for any requirements from the SDK team for the memo field. More specifically, by increasing the memo size to 64 bytes the memo is big enough to hold a Schnorr-style digital signature (which in some variants is 64 bytes and in some variants is 48 bytes, for 128 bit security). 

# Prior art
[prior-art]: #prior-art

This is an expansion of the existing Encrypted Memo's proposal in line with [anticipated future work](0003-encrypted-memos.md#future-possibilities), and specifically referencing to the [alternative larger memo size proposal](0003-encrypted-memos.md#alt-use-a-longer-memo-size). 

In terms of other memo fields being resized, I was not able to find examples that have resized their memos since implementation, but there have been proposals for stellar to [increase the memo size](https://galactictalk.org/d/433-stellar-should-have-a-big-memo-or-data).

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- There are no unresolved questions which should be resolved by this RFC.

# Future possibilities
[future-possibilities]: #future-possibilities

This could be changed again in the future as oram performance changes, and as we add more standardized memo fields.