- Feature Name: RTH Custom Memo Types
- Start Date: 2023-02-03
- MCIP PR: [mobilecoinfoundation/mcips#0060](https://github.com/mobilecoinfoundation/mcips/pull/0060)

# Summary
[summary]: #summary

[MCIP 3](https://github.com/mobilecoinfoundation/mcips/pull/3) added encrypted memo fields (ememos) to `TxOut`s. Since that addition, a number of standard memo types have been added. This proposal adds three new, standard memo types. These new memo types, however, are customizable. This will allow client applications to implement their own uses for the encrypted memo field.

# Motivation
[motivation]: #motivation

The encrypted memo field has nearly limitless uses. However, with only a handful of standard memo types, client applications can only, in practice, use them for a few different purposes. The future possibility that there will be many new, different clients entering the MobileCoin ecosystem must be considered.

Different clients will be designed to fill different roles and will have different use-cases. It is impossible to anticipate all of the different ways that MobileCoin technology and, by extension, the encrypted memo field can be used. Rather than have developers be blocked on the MCIP process and MobileCoin engineering capacity, the tools for them to fully utilize this feature to best suit their application's needs should be provided.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

This proposal adds three new encrypted memo types. The first two build on [Recoverable Transaction History (RTH)](https://github.com/mobilecoinfoundation/mcips/pull/4) (later expanded by [MCIP 24](https://github.com/mobilecoinfoundation/mcips/pull/24)). As such, the information included on the first two new memo types here builds on the background information contained in that proposal.

## Authenticated Sender With Data Memo

This memo type is similar to the Authenticated Sender Memo proposed in [RTH](https://github.com/mobilecoinfoundation/mcips/pull/4). After the encrypted memo fields were expanded in [MCIP 24](https://github.com/mobilecoinfoundation/mcips/pull/24), the Authenticated Sender Memo has 32 bytes of unused space. This memo utilizes that 32 bytes of empty space to store arbitrary data supplied by the client. This memo will have type bytes `0x0103`.

This memo will be sent to a recipient of a transaction in the payload `TxOut`. The accompanying change `TxOut` will contain a `DestinationWithDataMemo`.

## Destination With Data Memo

This memo type is similar to the Destination Memo proposed in [RTH](https://github.com/mobilecoinfoundation/mcips/pull/4). Just like the Authenticated Sender Memo in expanded RTH, the Destination memo has 32 bytes of unused space. This memo utilizes that 32 bytes of empty space to store arbitrary data supplied by the client. This memo will have type bytes `0x0204`.

This memo will be written by the sender of a transaction and attached to the change `TxOut`.

## Custom Memo

This memo type offers clients full use of the ememo field (except for the two type bytes). All of the space available for memo data is open for use by the client. No additional data is stored by the memo builder, allowing clients to implement their own memo types.

This memo type has type byes `0x0300` and stores 64 bytes of arbitrary data supplied by the client. It will be written to both the payload and change `TxOut`s of a transaction.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## 0x0103 Authenticated Sender With Data Memo

The 64-byte memo data is laid out as follows:

| Byte range | Item |
| ---------- | ---- |
| 0 - 16     | Sender's Address Hash |
| 16 - 48    | Custom Data           |
| 48 - 64    | HMAC                  |

This is memo is mostly the same as RTH's authenticated sender memo. Instead of leaving bytes 16-48 unused, however, this space can be filled with aritrary data by the client.

## 0x0204 Destination With Data Memo

The 64-byte memo data is laid out as follows:

| Byte range | Item |
| ---------- | ---- |
| 0 - 16     | Recipient's Address Hash |
| 16 - 17    | The number of recipients, an unsigned 8-bit number |
| 17 - 24    | Big-endian bytes of fee amount, an unsigned 56-bit number |
| 24 - 32    | Big-endian bytes of the total outlay amount, an unsigned 64-bit number |
| 32 - 64    | Custom Data                   |

This is memo is mostly the same as RTH's destination memo. Instead of leaving bytes 32-64 unused, however, this space can be filled with aritrary data by the client.

## 0x0300 Custom Memo

The 64-byte memo data is laid out as follows:

| Byte range | Item |
| ---------- | ---- |
| 0 - 64     | Custom Data           |

This memo has no defined structure or data boundaries. All bytes are left for the client to fill with arbitrary data. Developers can create their own memo layouts for their applications using this memo type.

# Drawbacks
[drawbacks]: #drawbacks

There are some risks associated with allowing clients full control of the ememo field. Inevitably, applications will introduce their own memo types and/or protocols using custom memo types. We cannot guarantee that end users won't use an account on more than one application (nor that developers will discourage this practice). The dangers of using custom memo types must therefore be documented clearly to try and avoid this as much as possible.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

## Sender and Destination With Data Memos

The current RTH memos are unnecessarily limiting. Nearly half of all sender and destination memo types is unused space. Opening this space up completely to clients will allow them to come up with their own uses for the ememo field. At the same time, the Sender and Destination with Data Memos allow for clients to use RTH.

It may also be possible to modify the existing sender and destination memos to allow for the 32 unused bytes to be written to. Rather than modifying the old memo types, new ones are proposed so that the memo type can signify to applications whether or not the data stored in those 32 bytes is meaningful.

## Custom Memo

For clients that do not need RTH, it wouldn't make sense for them to use the Sender and Destination With Data Memos. Providing this memo type gives clients complete freedom to implement whatever features they want using ememos.

It could also be possible to allow clients to use the memo type bytes as well, but this proposal does not allow for that. The Custom Memo was given a fixed type. Though it generally won't be supported, if a user shares one account across multiple different clients, developers have a standard way of recognizing unsupported data and choosing to ignore (rather than try to process) invalid memos for their application.

There could also be a separate memo type for the payload and change `TxOut`s, but this may be more complicated than necessary. Clients do not need to rely on checking memo types to differentiate change `TxOut`s.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- Would it be beneficial to have two different Custom Memo types: one for the payload `TxOut` and one for the change `TxOut`? Maybe the memo builder can accept a 'data' field and a 'change data' field. If the 'change data' field is not set, then the 'data' field is used for both outputs.
- Do these memo names effectively communicate their function? Some other names that have been considered are "General Purpose" and "Data." Renaming the Custom Memo to Data Memo would be more consistent with the other two memos, but it is anticipated that clients will come up with their own memo types using it.

# Future possibilities
[future-possibilities]: #future-possibilities

The additions outlined in this proposal allow for nearly limitless uses of the encrypted memo field. This will provide developers of MobileCoin applications the freedom to be creative and could aid in the development of countless features that haven't been conceived of yet. After seeing what developers can do with custom memos, it could also lead to the standardization of new memo types in the future.

