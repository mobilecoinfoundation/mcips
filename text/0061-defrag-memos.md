- Feature Name: RTH Defragmentation Memos
- Start Date: 2023-02-03
- MCIP PR: [mobilecoinfoundation/mcips#0061](https://github.com/mobilecoinfoundation/mcips/pull/0061)

# Summary
[summary]: #summary

This proposal introduces a new memo type to support [Recoverable Transaction History (RTH)](https://github.com/mobilecoinfoundation/mcips/pull/4). The new memo type will be used by clients to mark `TxOut`s created as part of account defragmentation. This can be used by client applications as a reliable way to exclude defragmentation transactions from appearing as regular transactions.

# Motivation
[motivation]: #motivation

[RTH](https://github.com/mobilecoinfoundation/mcips/pull/4) introduced a standard method of utilizing [encrypted memos](https://github.com/mobilecoinfoundation/mcips/pull/3) for recording transaction history between two users. Not all transactions on the MobileCoin blockchain are between two different users or accounts, however. Defragmenation transactions are transactions that a user sends to themselves in order to consolidate small `TxOut`s into larger ones. This is necessary if a user needs to send a large transaction but cannot satisfy the amount using 16 `TxOut`s or less.

From the client's perspective, these larger `TxOut`s created by defragmentation appear as `TxOut`s received through normal transactions. This would be confusing to users if displayed this way in an application. Currently, the only way to differentiate defragmentation transactions from regular ones is to use [RTH](https://github.com/mobilecoinfoundation/mcips/pull/4) and check if the sender and recipient are both the user's account. This isn't immediately clear though, especially to new developers who have not been introduced to concepts like fragmentation or [RTH](https://github.com/mobilecoinfoundation/mcips/pull/4). Thus, a new memo type is proposed to mark defragmentatino transactions.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

This proposal adds one new encrypted memo type, the Defragmentation Memo. Defragmentation transactions will write this memo to the payload `TxOut`. Defragmentation transactions will also not produce a change `TxOut`, as there should never be change and it is not needed to construct RTH. This memo has type bytes `0x0002`.

The Defragmentation Memo contains three pieces of data. Like RTH's Destination Memo, the fee and total outlay of the transaction are written to the memo. In addition to this, there is also a defragmentation ID number. If a highly fragmented account needs to send a large amount, it may require more than one transaction to sufficiently defragment. The defragmentation ID number can be used by clients to group defragmentation transactions from one session together. This might be desirable in order to show how much was paid in fees for defragmentation. If used, the defragmentation ID number should be selected randomly.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## 0x0002 Defragmentation Memo

The 64-byte memo data is laid out as follows:

| Byte range | Item |
| ---------- | ---- |
| 0 - 7      | Big-endian bytes of fee amount, an unsigned 56-bit number |
| 7 - 15     | Big-endian bytes of the total outlay amount, an unsigned 64-bit integer |
| 15 - 23    | Big-endian bytes of the defragmentation transaction ID, an unsigned 64-bit integer |
| 23 - 64    | Unused bytes |

The fee amount is stored as a u56 in the first 7 bytes. This is done for consistency with RTH Destination memos. The next 8 bytes contain the total outlay of the defragmentation transaction. Lastly, the defragmentation ID is stored in 8 bytes as a u64.

This memo will be written to the payload `TxOut` of a defragmentation transaction. Defragmentation transactions should not produce a change `TxOut`.

# Drawbacks
[drawbacks]: #drawbacks

Accounts which have already performed defragmentation will either have no memos or sender/destination memos for their old defragmentation transactions. Though we encourage not using an account with more than one application, not all developers will implement protections against it nor will all users heed this advice. If new applications do not handle legacy defragmentation transactions, they may inadvertently display them as normal transactions.

This proposal will also require minor modifications to SDK clients. Since the addition of RTH, all transactions will generate a change `TxOut` regardless of whether or not there is change or if RTH memos are even being written at all. It is fairly trivial to make the required changes, however.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

One alternative to defragmentatino memos would be to continue using RTH for identifying defragmentation transactions. But this approach has a higher barrier of entry for client developers in terms of MobileCoin knowledge. It is much simpler to just check the memo type; and when it comes to sharing example code, defragmentation memos are much easier to demonstrate.

Including a defragmentation ID may also not be necessary. Conversely, it may be better to use a larger value. A single u64 was chosen for simplicity and should be enough for one account. If clients do not need the ID field, they can also choose to ignore it.

It may also be viable to implement this without removing change outputs from defragmentation transactions. The defragmentation memo could just be written to both outputs. Clients could differentiate the `TxOuts` still as the change amount would be 0. Change outputs are not necessary though for defragmentation, so it makes more sense to remove them. They will always have a value of 0 for defragmentation transactions and are not necessary for RTH purposes.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- Is a defragmentation ID necessary or valuable? Should it be larger than 64 bits?
- Does this memo need any sort of validation? If client applications are hiding defragmentation transactions, users with custom clients could create 'invisible' transactions. This doesn't seem like a major issue, however.

# Future possibilities
[future-possibilities]: #future-possibilities

This proposal should make it much easier for developers to create apps that behave more like regular payment apps. It will also reduce the amount of code needed to identify defragmentation transactions.

