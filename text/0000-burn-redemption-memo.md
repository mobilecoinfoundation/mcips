- Feature Name: Burn-Redemption-Memo
- Start Date: 2022-05-02
- MCIP PR: [mobilecoinfoundation/mcips#0000](https://github.com/mobilecoinfoundation/mcips/pull/0000)
- Tracking Issue: N/A

# Summary
[summary]: #summary

We propose adding a new memo type to Recoverable Transaction History, extending the scheme from [MCIP #4](0004-recoverable-transaction-history.md) to handle burn transactions that correspond with the redemption of tokens on an external (non-MobileCoin) blockchain.


# Motivation
[motivation]: #motivation

With the upcoming addition of minting [MCIP #0037](https://github.com/mobilecoinfoundation/mcips/pull/37/) and the already-existing [MCIP #0035](0035-verifiable-burning.md) the path for adding and removing tokens from circulation on the MobileCoin blockchain has been established.
It is expected that burning (the action of permanently removing tokens from circulation on the MobileCoin blockchain) would often be correlated with an event happening outside of the MobileCoin blockchain. For example, it is possible for a bridge between the MobileCoin blockchain and some other external blockchain to exist, that either manually or automatically:
* Mints new tokens on the MobileCoin blockchain when handed tokens from the external blockchain.
* Burns (see [MCIP #0035](0035-verifiable-burning.md)) tokens on the MobileCoin blockchain and hands out tokens on the external blockchain.

When burn operations of this nature take place, it is desirable to have the option of including data in the MobileCoin transaction that somehow ties it to the external event that triggerred the burn. This will allow such bridge operators and outside auditors scanning the blockchain for burn transactions to tie the burn to the operation that happens outside of the MobileCoin blockchain.

Recoverable Transaction History ([MCIP #4](0004-recoverable-transaction-history.md)) provides a way to attach up to 64 bytes of memo to a `TxOut`. Additionally, it contains two bytes for specifying the memo type, so that clients know how to parse the memo contents. We propose adding a new memo type that is used for indicating burn `TxOut`s that contain 64-bytes of unstructured payload. The reason the payload is unstructured is that every token on the MobileCoin network might (and will likely) have different burn semantics. Additionally, this data is used by the entity performing the burn to do their own bookkeeping, and it is unnecessary for us to put constraints on what they can store. This is acceptable since this memo is not meant to be read by regular MobileCoin wallet implementations (since most wallets will not be view-key-scanning the burn address at all).


# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation


This is a simple proposal. We propose reserving the memo type 0x0001 for the burn redemption memo:

| Memo type bytes | Name                                              |
| -----------     | -----------                                       |
| 0x0001          | Burn redemption memo                              |
|

## 0x0001 Burn redemption memo

| Byte range | Item |
| ---------- | ---- |
| 0 - 64     | Unstructured payload  |

The payload of the burn redemption memo is left unstructured to give burners/bridge operators/etc the flexibility of storing whatever information they desire in order to associate the burn with activity outside of the MobileCoin blockchain. For example, this could contain an ERC20 address of the receipient of the tokens been handed out as a result of the burn, a transaction hash linking the burn to the transaction on the external blockchain that corresponds to the burn on the MobileCoin blockchain, etc.

# Drawbacks
[drawbacks]: #drawbacks

In our opinion, there is no reason why not to reserve a memo type for this purpose. We are very far from running out of memo types, and having this memo would enable the use-cases of linking burn transactions to outside activity as mentioned above.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

Given that we want a way to include information such as a correlated ERC20 address or transaction id on an external blockchain with a MobileCoin burn transaction, using RTH is the natural way to do so without needing a ledger format change.

While it is possible to add more information to the ledger to store this kind of data, it would make no sense as the majority of `TxOut`s are not burn `TxOut`s. Additionally, the memo field was created exactly for this kind of auxiliary data.

We could enforce a schema over the payload, e.g. the first byte could specify what kind of address or id is stored in the remaining 63 bytes. However, this seems suboptimal since it is impossible for us to predict how this field would get used and we do not want to enforce arbitrarily limits on entities choosing to use this mechanism to track burns. Additionally, there is no way for Consensus to actually verify the contents of the memo - so such schema would be a recommendation at most, but anyone using this memo could still store whatever they desire in the payload and ignore the schema.

We could avoid standardizing this memo field altogether, but as mentioned above, this is not enforceable in the transaction validation code. As such, we think it is better to explicitly reserve a memo type for this, so that it does not get re-purposed by future MCIPs or properly-behaved clients.


# Prior art
[prior-art]: #prior-art

None that we are aware of.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

# Future possibilities

None that we are aware of.

[future-possibilities]: #future-possibilities

In the future we might want to encourage a specific schema for the payload, or have a new memo type that contains a specific schema. That would only be possible after we get an insight into how this field is used.
