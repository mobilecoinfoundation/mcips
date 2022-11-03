- Feature Name: tx-fee-map-digest
- Start Date: 2022-11-03
- MCIP PR: [mobilecoinfoundation/mcips#0056](https://github.com/mobilecoinfoundation/mcips/pull/56)
- Tracking Issue: https://github.com/mobilecoinfoundation/mobilecoin/issues/2611

# Summary
[summary]: #summary

Modify the `Tx` struct to include a digest of the minimum fee map, which provides a mitigation against a possible information leak attack that can be executed by a malicious node operator.

A security audit performed by Trail of Bits has identified a potential information leak attack: The consensus enclave fee map can be changed by the untrusted side, by restarting the node. If the attacker controls a node and is able to get the user to submit their Tx once with one fee map, and once with another, then the priority value leaks information about what the token id of the Tx is.

In principle they could do this without the user knowing -- they would restart the node and it would not be able to attest to the peers after changing the fee map, but it could still accept Tx proposals and see the WellFormedTxContext which conatins the priority number. Then they turn the node back to normal. The user's app retries when their transaction doesn't land, and the same Tx comes back to their node, now with the regular fee map.

This requires having root on a consensus node, but the threat model is that attackers with root on the consensus node should not be able to see the user's token id.

We view this as a time-of-check-vs-time-of-use ([TOCTOU](https://en.wikipedia.org/wiki/Time-of-check_to_time-of-use)) bug, and we try to fix it a way that those bugs are sometimes fixed. The user, who checked the fee map, also makes an assertion about what the fee-map is when they try to use the system (by submitting a Tx). Before accepting the transaction, the enclave checks if the user's assertion is correct, and rejects the Tx without creating a WellFormedTxContext if the assertion is not correct. (This is sort of like dealing with races when manipulating atomic variables by using compare-and-swap.)

# Motivation
[motivation]: #motivation

We aim to keep to a minimum the amount of information about a transaction that a node operator has access to. The attack described below could enable leaking token id information, and the solution proposed in this MCIP addresses this issue.


# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

As mentioned above, the attack requires the malicious node operator to alter the minimum fee map. By manipulating the fee map for a given token the node operator can potentially observe changes to the priority value returned by the well-formedness check when the client re-submits a transaction. The attacker will need to capture the priority value once before the fee map change and once afterwards, and if they had managed to get it to change by altering the minimum fee, they could deduce which token id is transacted.

To protect from this, we need to ensure that the fee map the client is using to come up with the fee they are paying is the same one that is configured on the node. We propose adding a new field to the `Tx` structure that contains the digest of the fee map. This allows the enclave to verify that the fee map the client is using matches its own, and if not to reject the transaction without revealing any information about it.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

We will modify the `Tx` structure to look like this:
```
pub struct Tx {
    /// The transaction contents.
    #[prost(message, required, tag = "1")]
    pub prefix: TxPrefix,

    /// The transaction signature.
    #[prost(message, required, tag = "2")]
    pub signature: SignatureRctBulletproofs,

    /// Client's belief about the minimum fee map, expressed as a merlin digest.
    ///
    /// The enclave must reject the proposal if this doesn't match the enclave's
    /// belief, to protect the client from information disclosure attacks.
    /// (This is TOB-MCCT-5)
    #[prost(bytes, tag = "3")]
    pub fee_map_digest: Vec<u8>,
}
```

Clients wanting to protect themselves from this attack will begin including the minimum fee map digest inside the `Tx`. We will introduce a new method to `TransactionBuilder`, `set_fee_map` that takes a `FeeMap` and uses it to populate `fee_map_digest` when the transaction is constructured. `FeeMap` will need to be modified to include a method to calculate a digest, so that all users of `FeeMap` calculate the digest in the same way.
We will also need to move `FeeMap` into a different crate, since right now it is inside `mc-consensus-enclave-config`. However, following the changes proposed here, `TransactionBuilder` will also need to import the `FeeMap` object so that it can calculate its digest when constructing a transaction (this might also be useful for having sane default minimum fees). It doesn't make sense that `TransactionBuilder` will have a depdency on any consensus enclave crates.

Since this is a new field, and it can be empty, it is entirely backwards compatible. Old clients will not include it, and in that case the enclave will not attempt to compare it to its own `FeeMap` digest. It will only get included by clients that wish to protect themselves from this attack.

Clients can learn about the FeeMap a node is configured with by using the `BlockchainAPI.GetLastBlockInfo` GRPC API. While there is no guarantee the node isn't lying about the fee map it hands the user, the fact that the enclave will later compare the `fee_map_digest` the client computes from the fee map handed by the node to the one it has been initialized with provide the protection from the attack described above.

It is worth noting that the entire `Tx` is sent over an encrypted attested connection, and as such an attacker is not able to manipulate its contents even if the new `fee_map_digest` is not included as part of the transaction's signature. In fact, not including it serves a useful purpose: it allows clients to easily change it as needed if the fee map has changed on consensus without requiring to re-sign the transaction.

# Drawbacks
[drawbacks]: #drawbacks

This makes the `Tx` slightly bigger. `Tx`s are relayed to all consensus nodes as part of the consensus process, and they are stored on all nodes in their pending queue until they are processed (or expire). We do not believe this is a meaningful amount of data given the current size of `Tx`s (multiple kilobytes).

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

Some might find it confusing that the `Tx` structure now contains the transaction data (inside the `TxPrefix`), a signature over that data (`SignatureRctBulletproofs`) and now some data that is not covered by a signature (even though there is no need for it to be). If `client_propose_tx` accepted something like a `TxRequest` instea of just a `Tx`, it would've been possible to add the `fee_map_digest` to this `TxRequest`, leaving the `Tx` intact.
Unfortunately, `client_propose_tx` does not do that and instead accepts a `Tx` - so we are left with two options:
1. We modify the `Tx` like we are proposing in this MCIP
2. We add a new API endpoint `client_tx_propose_v2` that accepts this new `TxRequest`

While creating a new API for proposing a `Tx` will have the advantage of making it easier to add fields that are not directly tied to the transaction and its signature, it will complicate client migration. If we were to add a new endpoint, all clients will need to support both using the current `client_propose_tx` API, and the new one (`client_propose_tx_v2`), and make a choice on which one to use based on current block version or enclave version. Additionally, this means we will need to keep the old API endpoint around for long enough until we have ensured no clients are using it. This is more work and more code than simply adding the field to `Tx`, which is why we think that approach is superior.

# Prior art
[prior-art]: #prior-art

[MCIP 25](text/0025-confidential-token-ids.md), which introduced confidential token ids.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

None.

# Future possibilities
[future-possibilities]: #future-possibilities

The naming of `Tx` and `TxPrefix` are confusing. It might be worth renaming `TxPrefix` into `Tx`, since that is actually the contents of the transaction. Following that, it would make sense to rename `Tx` into `TxProposal` or something like that, since it contains the proposed `Tx`, a signature to validate it and any other proposal-related data such as the `fee_map_digest`.

Now that `TransactionBuilder` encourages users to provide it with the minimum fee map, it can be modified to use this for figuring out the fee when it is not explicitly provided by `set_fee`.
