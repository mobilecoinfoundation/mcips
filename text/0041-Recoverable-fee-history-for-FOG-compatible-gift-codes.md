- Feature Name: Recoverable Fee History for FOG Compatible Gift Codes
- Start Date: May 18, 2022
- MCIP PR: [mobilecoinfoundation/mcips#0041](https://github.com/mobilecoinfoundation/mcips/pull/0041)
- Tracking Issue: https://github.com/mobilecoinfoundation/mobilecoin/issues/2001

# Summary
[summary]: #summary

[MCIP #32 - FOG Compatible Gift Codes](https://github.com/mobilecoinfoundation/mcips/blob/main/text/0032-fog-compatible-gift-codes.md) 
outlines a proposal for TxOut based gift codes which FOG users can privately create, read, and redeem without having to 
possess a copy of the ledger or rely on an untrusted FOG ledger endpoint.

There are 3 possible actions stated in MCIP 32: gift code funding, cancellation, and sending. 

Each action has a distinct memo recording context about that action.

| Memo type bytes | Name                                              |
| -----------     | -----------                                       |
| 0x0201          | Gift code funding memo                            |
| 0x0202          | Gift code cancellation memo                       |
| 0x0002          | Gift code sender memo                       |

These memos do not currently allocate any bytes for tracking transaction fees. This MCIP proposes to allocate 7 bytes to
each memo representing an unsigned 56 bit number which can be used to track the fee amount paid in each stage of the 
gift code flow.

# Motivation
[motivation]: #motivation

Normal recoverable transaction history isn't possible for gift codes because at the point when a gift code is funded, 
it is not yet known what account will redeem this gift code. When funding or cancelling a gift code, the sender may want
to be able to have a recoverable history for the fees they paid to do so.

Similarly, when a gift code is "redeemed" the amount sent to the change address will differ from the amount of the TxOut
the receiver is able to unblind with the data received from the sender by an amount equal to the fee paid to redeem it. 
This may be confusing to gift code recipients if they are not able to account for the fee that they paid to redeem it.

Thus we propose modifications (explained below) to the gift code memo types in order to create recoverable fee history.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

[MCIP #4 - Recoverable Transaction History](https://github.com/mobilecoinfoundation/mcips/blob/main/text/0004-recoverable-transaction-history.md#0x0200-destination-memo)
provides a design for recoverable transaction history. In the MCIP the "Destination Memo" specifies that 7 bytes be 
allocated for a 56 bit number representing the fee paid.

Staying consistent with the reasoning provided in that MCIP, it's proposed that each gift code memo will also allocate 7
bytes for an unsigned 56-bit number to represent the fee paid for each gift code action. A full 8 bytes could be used, 
but fees are not likely to be within the full range of a 64 bit number and it is desired to save space for descriptive
utf-8 bytes.



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
| Byte range | Item                                                                                             |
| ---------- |--------------------------------------------------------------------------------------------------|
| 0 - 4      | First 4 bytes of blake2b over `tx_out_public_key` of gift code tx out                            |
| 4 - 11      | Big-endian bytes of fee amount, as an unsigned 56-bit number                                     |
| 11 - 64     | "Note": A UTF-8 string with that is either exactly 53 bytes or null terminated if under 53 bytes |

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
| Byte range | Item                                                                                             |
| ---------- |--------------------------------------------------------------------------------------------------|
| 0 - 7      | Big-endian bytes of fee amount, as an unsigned 56-bit number                                     |
| 7 - 64     | "Note": A UTF-8 string with that is either exactly 57 bytes or null terminated if under 57 bytes |

# Drawbacks
[drawbacks]: #drawbacks

By adding recoverable fee history, we take 7 bytes away from the utf-8 fields in the Funding and Sender memos which could
be used to describe useful contextual information within the apps using them.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

The rationale behind this is to create consistent accounting.

However it could potentially be said that this is not strictly necessary and that it should be left up to implementing 
apps to decide what to do with the utf-8 bytes. In a regime limited to 64 bytes or less, 7 bytes could be useful. 
Therefore alternatives could be not to implement fee tracking at all or to use a lesser amount of bytes to track the fee.

# Prior art
[prior-art]: #prior-art

* [MCIP #32 - FOG Compatible Gift Codes](https://github.com/mobilecoinfoundation/mcips/blob/main/text/0032-fog-compatible-gift-codes.md) 
provides the design for FOG compatible gift codes
* [MCIP #4 - Recoverable Transaction History](https://github.com/mobilecoinfoundation/mcips/blob/main/text/0004-recoverable-transaction-history.md#0x0200-destination-memo) 
specifies 7 bytes for fee history and provides reasoning for why this is the correct amount.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

# Future possibilities
[future-possibilities]: #future-possibilities
