- Feature Name: Recoverable-fee-history-For-FOG-compatible-gift-codes
- Start Date: May 18
- MCIP PR: [mobilecoinfoundation/mcips#0041](https://github.com/mobilecoinfoundation/mcips/pull/0041)
- Tracking Issue: https://github.com/mobilecoinfoundation/mobilecoin/issues/2001

# Summary
[summary]: #summary

[MCIP #32 - FOG-compatible-gift-codes](https://github.com/mobilecoinfoundation/mcips/blob/main/text/0032-fog-compatible-gift-codes.md) 
outlines a proposal for TxOut based gift codes which FOG users can privately create, read, and redeem without having to 
possess a copy of the ledger or rely on an untrusted FOG ledger endpoint. This enables gift codes to be used by mobile 
wallet users and potentially be sent to individuals who do not yet have a MobileCoin account.

This is achieved in 2 stages:
1. Sending a TxOut to a special subaddress reserved for gift codes at u64::MAX - 2 along with a *funding memo* written 
to the change output of the transaction that records the first 4 bytes of a Blake2b512 hash of the gift code TxOut and a
60 byte utf-8 note that can be used to describe information about the gift code. If the sending party decides to cancel 
this gift code TxOut (say, if it is never redeemed by a receiver) it is sent back to the sending party's change address with 
a *cancellation memo* that includes the global index of the gift code TxOut originally sent to the reserved gift code 
subaddress.


2. Sending a protobuf message via grpc to another party giving the information required to locate, unmask, and spend the 
TxOut. The receiving party "redeems" the gift code by sending it to their change subaddress along with a *sender memo* 
that allows the receiver to record up to 64 utf-8 bytes to document any desired information about the gift code.

The memos which record this information are shown in the table below:

| Memo type bytes | Name                                              |
| -----------     | -----------                                       |
| 0x0201          | Gift code funding memo                            |
| 0x0202          | Gift code cancellation memo                       |
| 0x0002          | Gift code sender memo                       |

However, as designed, the gift code memos do not allow any room for tracking information about the fees paid to create, 
redeem, or cancel any of the memos.

This MCIP proposes to allocate 7 bytes to each memo representing an unsigned 56 bit number which can be used to track 
the fee amount paid in each stage of the gift code flow.

# Motivation
[motivation]: #motivation

Normal recoverable transaction history isn't possible for gift flow because at the point when a gift code is funded and 
sent to the gift code subaddress, it is not yet known what account will redeem this gift code. When funding or 
cancelling a gift code, the sender may want to be able to have a recoverable history for the fees they paid to do so.

Similarly, when a gift code is "redeemed" the amount sent to the change address will differ from the amount received in
the protobuf message by an amount equal to the fee paid to redeem it. This may be confusing to gift code recipients if 
they are not able to account for the fee that they paid to redeem it.

Thus we propose modifications to the gift code memo types in order to create recoverable fee history.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Each memo type will have allocate 7 bytes for an unsigned 56-bit number which represents the fee amount paid in each 
stage of the gift code flow. A full 8 bytes could be used, but fees are not likely to be within the range of a 64 bit 
number and it is desired to save space for descriptive utf-8 bytes.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The following proposed changes to each memo type are described below.

## 0x0201 Gift code funding memo

### Current description
| Byte range | Item |
| ---------- | ---- |
| 0 - 4      |  First 4 bytes of blake2b over `tx_out_public_key` of gift code tx out |
| 4 - 64     | "Note": A UTF-8 string with that is either exactly 60 bytes or null terminated if under 60 bytes |

### Proposed change
| Byte range | Item |
| ---------- | ---- |
| 0 - 4      |  First 4 bytes of blake2b over `tx_out_public_key` of gift code tx out |
| 4 - 11      | Big-endian bytes of fee amount, as an unsigned 56-bit number |
| 11 - 64     | "Note": A UTF-8 string with that is either exactly 60 bytes or null terminated if under 60 bytes |

## 0x0202 Gift code cancellation memo

### Current description
| Byte range | Item |
| ---------- | ---- |
| 0 - 8      | `global_index` of gift code tx out that was cancelled |
| 8 - 64     | Unused bytes |

### Proposed change
| Byte range | Item |
| ---------- | ---- |
| 0 - 8      | `global_index` of gift code tx out that was cancelled |
| 8 - 15      | Big-endian bytes of fee amount, as an unsigned 56-bit number |
| 15 - 64     | Unused bytes |

## 0x0002 Gift code sender memo

### Current description
| Byte range | Item |
| ---------- | ---- |
| 0 - 64     | "Note": A UTF-8 string with that is either exactly 64 bytes or null terminated if under 64 bytes |

### Proposed change
| Byte range | Item |
| ---------- | ---- |
| 0 - 7      | Big-endian bytes of fee amount, as an unsigned 56-bit number |
| 7 - 64     | "Note": A UTF-8 string with that is either exactly 64 bytes or null terminated if under 64 bytes |

# Drawbacks
[drawbacks]: #drawbacks

By adding recoverable fee history, we take 7 bytes away from the utf-8 fields in the Funding and Sender memos which could
be used to describe useful contextual information within the apps using them.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

The rationale behind this is to create consistent accounting.

However it could potentially be said that this is not strictly necessary and that it should be left up to implementing 
apps to decide what to do with the utf-8 bytes. In a regime limited to 64 bytes or less, 7 bytes could be useful. 
Therefore alternatives could be not to implement fee tracking at all OR to only use 6 bytes to track the fee.

# Prior art
[prior-art]: #prior-art

[MCIP #32 - FOG-compatible-gift-codes](https://github.com/mobilecoinfoundation/mcips/blob/main/text/0032-fog-compatible-gift-codes.md)

# Unresolved questions
[unresolved-questions]: #unresolved-questions

The merits of recoverable fee history over the utf-8 bytes they replace

# Future possibilities
[future-possibilities]: #future-possibilities

Future extensions of the gift code memos or creation of new gift code memo types
