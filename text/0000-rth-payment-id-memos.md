- Feature Name: (`Recoverable Transaction History Paymend ID Memos`)
- Start Date: (2022-10-18)
- MCIP PR: [mobilecoinfoundation/mcips#0000](https://github.com/mobilecoinfoundation/mcips/pull/0000)
- Tracking Issue: (link to tracking issue)

# Summary
[summary]: #summary

[#4 Recoverable Transaction History (RTH)](https://github.com/mobilecoinfoundation/mcips/pull/4) introduced three different memo types for [encrypted memos](https://github.com/mobilecoinfoundation/mcips/pull/3). This proposal specifies three more memo types:

- Destination With Payment Request Id Memo
- Authenticated Sender With Payment Id Memo
- Destination With Payment Id Memo

This feature will accomplish two main things:

1. Allow clients utilizing payment requests to recover complete transaction history for both users
2. Implement recoverable transaction history for payment intent flows

# Motivation
[motivation]: #motivation

The Authenticated Sender With Payment Request Id Memo was introduced to support clients implementing a payment request flow. A user can request a payment with a specific ID. Upon receiving a payment and sender memo with the same payment request ID, it can be reasonably assumed that the newly received payment is associated with the original request.

The user fulfilling a payment request this way, however, has no way of identifying any outgoing payment as belonging to a specific request ID after sending it. They only attach a regular destination memo to the change TxOut. Only the user which made the payment request is capable of reconstructing this aspect of the transaction history. This should be resolved so that recoverable transaction history between two parties is the same for both. This can be accomplished by creating a new destination memo type which contains the payment request ID.

Additionally, some clients may wish to implement the opposite flow. A sending user could use an ID to enable the recipient to associate that payment with some previously shared and/or agreed upon payment intent. It is proposed that this be implemented with two new memo types: the Authanticated Sender With Payment Id Memo and the Destination with Payment Id Memo.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

This proposal is an addition to [#4 Recoverable Transaction History](https://github.com/mobilecoinfoundation/mcips/pull/4). As such, the information included on new memo types here builds on the background recoverable transaction history information contained in that proposal.

## Destination With Payment Request Id Memo

This memo will be paired with recoverable transaction history's Authenticated Sender With Paymend Request Id Memo. This memo will have type bytes `0x0203`. The Authenticated Sender With Paymend Request Id Memo that RTH defines is sent to a user to let them know that the incoming payment is related to a specific payment request. However, attached to the change TxOut of that transaction is just a normal Destination Memo. This prevents the user sending the transaction from having a record of fulfilling a payment request. The party receiving the transaction therefore has more information about the transaction history than the sender.

This proposed Destination With Payment Request Id Memo fixes this by holding an 8-byte payment request ID number. If Alice wants to fulfill a payment request from Bob, Alice can attach a sender with payment request ID memo to Alice's main TxOut being sent to Bob. Alice can then attach a destination with payment request ID memo (containing the same request ID) to the change TxOut. This allows Alice to have her own record of fulfilling the payment request.

## Authenticated Sender and Destination With Payment Id Memos

For clients wishing to implement the opposite flow, the sender and destination with payment ID memos are proposed (type bytes `0x0102` and `0x0204`, respectively). Suppose Alice wishes to send a payment to Bob, but Bob either does not have a public address or has not shared it with her. Alice can, using some shared service or communication channel, notify Bob that a payment with a specific ID is waiting for him. Bob may then wish to create and/or share his public address with Alice to receive the payment.

With all of this established, Alice is able to then create and send the payment to Bob. Alice should build a sender with payment ID memo and attach it to the TxOut for Bob. Paired with that, for her own record, she should attach a destination with payment ID memo to her change TxOut so she also has record of this payment intent being fulfilled. Once Bob receives the payment, he can see that it contains a sender memo with the same payment ID as the previously shared payment intent. This informs Bob that the pending payment to him is complete. Had Bob received a different payment (from Alice or another party) in between the time the intent was shared and completed, he would know that this is not related to the same intent.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## 0x0203 Destination With Payment Request Id Memo

The 64-byte memo data is laid out as follows:

| Byte range | Item |
| ---------- | ---- |
| 0 - 16     | Recipient's Address Hash |
| 16 - 17    | The number of recipients, as an unsigned 8-bit number |
| 17 - 24    | Big-endian bytes of fee amount, as an unsigned 56-bit number |
| 24 - 32    | Big-endian bytes of the total outlay amount, as an unsigned 64-bit number |
| 32 - 40    | Big-endian bytes of 8-byte payment request id number |
| 40 - 64    | Unused bytes                  |

This is exactly identical to RTH's destination memo except it contains an 8-byte payment request ID number. A user fulfilling a payment request will use this memo and populate that field with the same ID they put into an authenticated sender with payment request ID memo. This will allow the fulfilling user to keep a record that the payment request was fulfilled.

## 0x0102 Authenticated Sender With Payment Id Memo

The 64-byte memo data is laid out as follows:

| Byte range | Item |
| ---------- | ---- |
| 0 - 16     | Sender's Address Hash |
| 16 - 24    | Big-endian bytes of 8-byte payment intent id number |
| 24 - 48    | Unused bytes          |
| 48 - 64    | HMAC                  |

This is exactly identical to RTH's authenticated sender with payment request ID memo. The only difference is the memo type. Utilizing the 2-byte memo type, client applications can differentiate payment requests from payment intents.

## 0x0204 Destination With Payment Id Memo

The 64-byte memo data is laid out as follows:

| Byte range | Item |
| ---------- | ---- |
| 0 - 16     | Recipient's Address Hash |
| 16 - 17    | The number of recipients, as an unsigned 8-bit number |
| 17 - 24    | Big-endian bytes of fee amount, as an unsigned 56-bit number |
| 24 - 32    | Big-endian bytes of the total outlay amount, as an unsigned 64-bit number |
| 32 - 40    | Big-endian bytes of 8-byte payment intent id number |
| 40 - 64    | Unused bytes                  |

This is exactly identical to the herein proposed destination with payment request ID memo. The only difference is the memo type. Utilizing the 2-byte memo type, client applications can differentiate payment requests from payment intents.

# Drawbacks
[drawbacks]: #drawbacks

There is a limited number of memo types (65536). It likely won't be feasible to add more in the future. This proposal will utilize three more memo types that we won't be able to use for something else in the future.

Additionally, adding three new memo types adds complexity to recoverable transaction history. Many client applications may not decide to utilize payment requests or intents but may wish to implement basic recoverable transaction history. Fortunately, this proposal cements the idea that the sender and destination memos are extended by the payment request ID and payment intent ID variations. It will take additional dev work, but MobileCoin clients and SDKs will be able to use this hierarchy to allow handling of the new memo types the same as the old ones. End-user facing applications may strictly write code to handle the basic sender and destination memo types. This code will always work regardless of any new features that they do not want being added in the future.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

## Rationale

The original intent of recoverable transaction history was that any two parties could recover the history of all transactions between each other. This has not been accomplished with the current implementation of payment request IDs. Since there is the ability for the recipient of a transaction to determine which payment request it relates to (if applicable), the sender of that transaction should also be able to do so. As of right now there is no way for the sender to know whether or not it relates to a payment request at all.

Since recoverable transaction history supports payments requests, it follows that it should also support payment intents. This has many use-cases such as being able to send to a user that has not created or shared their public address with the prospective sender. This allows applications more options for things such as inviting new users or inintiating a new transaction history between (theretofore unintroduced) existing users.

## Alternative Implementation

Rather than use the memo type bytes to differentiate between payment requests and intents, some of the yet allocated data in existing memo types could be utilized.. A single additional byte could be designated for this. Only one new destination memo type would be needed to implement this. This implementation begs other considerations be made, however. If this is a binary value determining strictly either request or intent, should it occupy 1 bit rather than an entire byte? This could allow for a larger payment ID by packing an additional 7 bits into the extra byte. But that would not allow for other payment ID memo types in the future. So how many bits are needed to make this reasonably future-proof?

# Prior art
[prior-art]: #prior-art

There are numerous examples of payment platforms that allow for actions such as payment requests and payment intents.

- [How to invite Venmo users by sending them a payment](https://help.venmo.com/hc/en-us/articles/217042708-Inviting-Friends)
- [Sending payment requests through Venmo](https://help.venmo.com/hc/en-us/articles/210413477-Sending-Requesting-Money)
- [Sending payment intent through Zelle to an unenrolled user](https://www.zellepay.com/faq/what-if-person-i-am-sending-money-hasnt-enrolled-zelle)
- [Receiving payment intent as a new Zelle user](https://www.zellepay.com/faq/someone-sent-me-money-zelle-how-do-i-receive-it)
- [Zelle offers the ability to request payments from users](https://www.zellepay.com/faq/how-can-i-use-zelle)

This proposal supports applications in implementing standard payment platform features such as these in a privacy-protecting way—the complete history of which can reliably be recovered strictly from data on the blockchain.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

This feature does not have any questions that need be resolved in order for a design to be completed.
