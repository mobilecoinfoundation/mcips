- Feature Name: `confidential_token_ids`
- Start Date: 2022-02-05
- MCIP PR: [mobilecoinfoundation/mcips#0025](https://github.com/mobilecoinfoundation/mcips/pull/0025)
- MobileCoin Epic: TBD

# Summary
[summary]: #summary

Enable new token types to be added to the protocol and transacted on chain without revealing or leaving a transparent record of the token type.

# Motivation
[motivation]: #motivation

The MobileCoin ledger protocol currently supports one token type, MOB. We would like to allow the addition of assets to the ledger without revealing which assets were involved in any transaction. The transaction confidentiality and integrity guarantees should be the same for the MobileCoin blockchain no matter how many token types are supported. Namely, the sender, the recipient, and the amount should remain private to all except the participants in the transaction, and no double spending or unauthorized minting should be possible. In addition, we now specify that the token type should remain private to all except the participants in the transaction.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

A Transaction Output (TxOut) has a confidential `token_id` bytes field (`masked_token_id`). Transactions with inputs and outputs, including a fee output, must be composed of a single `token_id`. Some tokens may have additional functionality, such as minting and burning, which require verification of the `token_id` as part of the authorization of the action.

Each `token_id` has an explicit minimum fee specified by node operators, and included in the attestation handshake, ensuring that all nodes in the network are configured with the same fee minimums and `token_id` sets. New `token_ids` can be added only with unanimous agreement from all node operators.

A transaction specifies the `token_id` for the entire transaction, with a cryptographic guarantee that all inputs and outputs are of the same `token_id`. This prevents spending one `token_id` as a member of another `token_id`. The TransactionBuilder now takes a `token_id` as an argument in its initializer. The `token_id` is set in the clear on the `TxPrefix` for a transaction, which is only accessed in the enclave, after the payload is delivered via an encrypted, attested channel. This is similar to the `TxInputs`, and thus has a similar threat model for confidentiality. See [Confidentiality Analysis](#confidentiality-analysis) for an assessment of risk.

Clients must choose how to expose multiple asset types to their users. There may be clients who wish to only expose a subset of asset types, or clients who wish to provide balances and transactionality in all asset types. 

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

### Masked Token ID Field in the TxOut

For each token type that is added, we specify a `u32 token_id`. We maintain a list of `KnownTokenIds` in both the external proto definition, as well as in the transaction core crate. The first token type is `MOB = token_id(0)`, with the assumption that an unspecified `token_id` defaults to 0, for backward cmopatibility. New `token_ids` increase monotonically. For example, the `KnownTokenId` enum in the external proto may be something like,

```
enum KnownTokenId {

    \\\ Token Name = token_id
    MOB = 0;

}
```

### Amount

Previously, an `amount` of picomob in a TxOut is represented by a Pedersen commitment to that number, as well as a `masked_value`, which is created by XORing the number of picomob with a pseudorandom mask (derived by hashing a shared secret). The purpose of the `masked_value` is that when the recipient receives a TxOut, they can unmask this value, then reconstruct the Pedersen commitment from the number, to confirm successful recovery of the value.

After this change:

A `masked_token_id` is added, which, similarly to `masked_value`, provides a way for the recipient to learn the `token_id` (and then subsequently confirm it). The Pedersen commitment to the value now is made using a basepoint `H_i` which depends on the `token_id`. (The basepoint for the blinding factor is the same as before regardless of the `token_id`.)

Thus, the recipient can attempt to scan a TxOut by first constructing the shared secret, then unmasking the `masked_amount` and `masked_token_id`, and then reconstructing the Pedersen commitment. If this reconstruction is successful, then the recipient knows the `amount` and the `token_id`, similarly as before the support for a confidential `token_id`.

The `masked_token_id` field is added to the [`Amount`](https://github.com/mobilecoinfoundation/mobilecoin/blob/cb10d41d404c552cab19de0163992923a878ed5e/transaction/core/src/amount/mod.rs#L46) in each TxOut. The protocol representation of the TxOut uses Protobuf, which enables backward compatability in this way: if the field is not populated (Protobuf bytes field of length 0), the default `token_id` is 0. Otherwise, the field is a Protobuf bytes field of length 4. Please see the [Upgrade Plan](#upgrade-plan) for more discussion on rolling out this change with consideration for backward compatibility.

* To compute the 4 byte `masked_token_id` from the `token_id`, we hash the TxOut shared secret, with a prefix, XORed with the `token_id`, and take the little endian representation of those bytes.

    ```
    masked_token_id = (token_id ^ Blake2B(AMOUNT_TOKEN_ID_DOMAIN_TAG | shared_secret)).to_le()
    ```

* To recover the `token_id` from the `masked_token_id`, we reverse the process (if the bytes field is length = 4, otherwise `token_id = 0`).

    ```
    unmasked_token_id = masked_token_id ^ Blake2b(AMOUNT_TOKEN_ID_DOMAIN_TAG | shared_secret)
    ```

### Pedersen Generators

Previously, we used the same curve base point for all Pedersen generators to Amounts. After this change, we will need a curve base point, unique for each `token_id`, from which the TxOut's commitment is generated with the value of the `token_id` in the TxOut.

The requirements for the Pedersen generators are:

* The generator used for `MOB = token_id(0)` is the same as before, so `H_0` is a known value.

* The set of all generators used for `token_ids` should be cryptographically "orthogonal", meaning that, it should be intractable for anyone to find a linear relationship between them.

* The function mapping a `token_id`, `i`, to the generator `H_i` must be implemented in constant-time.

The established value for `H_0` in mobilecoin is to take the `RISTRETTO_BASEPOINT` bytes, hash them using Blake2b, and then hash this to the ristretto curve using the elligator map exposed by curve25519-dalek.

We propose to compute `H_i` by converting `i` to little endian bytes, and XOR'ing these bytes over the first four bytes of the `RISTRETTO_BASEPOINT` bytes before hashing them with Blake2b, because this is a simple way to meet these requirements.

For example, an implementation to obtain a generator for `token_id = i` could look like:

```
B = RistrettoPoint::from_hash(Blake2b(HASH_TO_POINT_DOMAIN_TAG | RISTRETTO_BASEPOINT ^ token_id))
B_blinding = RISTRETTO_BASEPOINT
```

After this change, `range_proofs`, `rct_bulletproofs`, and `ring_signature/mlsags` will be relative to a Pedersen generator, and it is not possible to construct a range proof relative to another generator if those generators are orthogonal. Thus, it is guaranteed that all transaction inputs and outputs are using the same `token_ids`. This is implied by the homomorphic encryption property of [Pedersen Commitments](https://www.cs.cornell.edu/courses/cs754/2001fa/129.PDF), namely that addition on the commitments preserves the additive relationship of the pre-committed values, as long as they are using the same group generator. `H_i` is revealed to the verifier, who must show that the sum of inputs does not overflow, or `sum(outputs) = a * G + b * H_i` for some TxOut-specific values of `a` and `b`. 

### Commitments

To prove each TxOut's commitment to the `amount` value, we use a Pedersen commitment, consisting of a group generator (`H_i`) raised to the value of the `amount`, with a blinding factor to prevent brute-forcing the full range of values (`2^64`). We compute the blinding factor as the blinding group generator raised to a hash of the shared secret for the TxOut.

```
blinding_factor = Blake2B(blinding_tag | shared_secret)
commitment = H_i * Scalar::from(amount) + G * blinding_factor
```

- Note, the group generators for `token_ids` are computed using hashing to a curve so that they are "orthogonal" for computable adversaries, meaning there is no discernable, nontrivial linear relationship between base points. See our [transaction core group generators reference](https://github.com/mobilecoinfoundation/mobilecoin/blob/master/transaction/core/src/ring_signature/mod.rs#L29). 
    - Note also, the blinding factor base point does not need to be unique per `token_id`, so we use the same blinding factor base point for all `token_ids`.

### Fee Outputs

Currently, the enclave aggregates the fees for multiple transactions in a block, in order to mint a single fee output for the block and conserve space. To support multiple confidential `token_ids`, when minting fee outputs, the enclave must create multiple fee outputs if multiple `token_ids` were used in the block. A consideration here is that we would like for an observer not to discern whether a block contains transactions of multiple asset types by statistically analyzing the number of outputs to derive information about the number of fee outputs. 

A simple proposal is that each block now contains a number of fee outputs scaling linearly with the total number of supported `token_ids`. This way we can continue to aggregate fees, but not reveal how many token types were included in the block. For blocks with fewer transactions than `token_ids`, the number of fee outputs is `min(num_token_ids, num_transactions_in_block)`

Note that the untrusted portion of the node software knows how many transactions are in each block, and this is not considered statistically relevant information with respect to confidentiality. In other words, the current fee aggregation behavior is not meant as a confidentiality measure, purely as a space-saving measure.

### Fee Priority and Transaction Sorting

Currently the [WellFormedTxContext](https://github.com/mobilecoinfoundation/mobilecoin/blob/a092039a4a5cc7f5e3e16eeadd0b2bc3a12667ae/consensus/enclave/api/src/lib.rs#L50) contains the fee, which must be used by untrusted to [sort transactions](https://github.com/mobilecoinfoundation/mobilecoin/blob/a092039a4a5cc7f5e3e16eeadd0b2bc3a12667ae/consensus/enclave/api/src/lib.rs#L131) when [combining transactions](https://github.com/mobilecoinfoundation/mobilecoin/blob/a092039a4a5cc7f5e3e16eeadd0b2bc3a12667ae/consensus/service/src/tx_manager/untrusted_interfaces.rs#L40) for consideration by the enclave, prioritizing higher fees for inclusion in the block during periods of network congestion. In order to keep the `token_id` confidential, we must provide a mechanism for untrusted to accurately sort transactions before asking the enclave to do work on the transactions, without revealing the `token_id`, and at the same time, allowing for different asset types to express differing minimum fee requirements.

To derive a sorting order without revealing the transaction type, the `fee` in the `WellFormedTxContext` derives a `priority`, which is calculated based on some ratio between the minimum fees configured by the node operator on startup. For example, the network operators may have configured the following minimum fees:

```
{
    asset_000_fee = 1,
    asset_111_fee = 2,
    asset_888_fee = 3,
}
```

The ratio between the fees for the `token_ids` is the following, using the least common multiple of the fees:

`token_id` | `fee` | `priority` multiplier
--- | --- | ---
000 | 1 | 6
111 | 2 | 3
888 | 3 | 2


Then, imagine that a block contains the following set of transactions with the given fees:

```
[
    transaction_1 {token_id: 000, fee: 1},
    transaction_2 {token_id: 000, fee: 1.5},
    transaction_3 {token_id: 111, fee: 2},
    transaction_4 {token_id: 888, fee: 3},
    transaction_5 {token_id: 888, fee: 4}
]
```

The sort order would be:

`transaction_id` | `token_id` | `priority` multiplier | original `fee` | `priority` result
--- | --- | --- | --- | ---
transaction_2 | 000 | 6 | 1.5 | 9
transaction_5 | 888 | 2 | 4 | 8
transaction_1 | 000 | 6 | 1 | 6
transaction_3 | 111 | 3 | 2 | 6
transaction_4 | 888 | 2 | 3 | 6

Note how the two transactions paying more than their minimum fee are sorted to the top, in proportion to how much they are paying above the minimum fee, while all minimum fee transactions have an equal chance of beeing sorted because their `priority` is equal.

For users who are adjusting the fees of their submitted transactions to increase the likelihood of their transaction being accepted during times of congestion, they can observe the `priority` multiplier for their `token_id`, and the recently accepted `priority` values, and determine what fee value is appropriate to increase the chances of acceptance.

### Performance Impact: Size & Space Analysis

- Each TxOut has 6 new bytes allocated in the Amount object for the `masked_token_id` (4 bytes for the value, 2 bytes for Protobuf overhead)
- Each block now contains a number of fee outputs capped by `min(num_token_ids, num_transactions_in_block)`

### Confidentiality Analysis

The [TxPrefix](https://github.com/mobilecoinfoundation/mobilecoin/blob/a9db0bc7bcac9bac1c1232b194003d02bccaf283/transaction/core/src/tx.rs#L149) contains the `token_id` and the `fee` in the clear. We, and any users of the protocol, need to clearly understand the footprint of this data, and the ways in which an attacker may try to discern the `token_id` of TxOuts.

The [TxPrefix](https://github.com/mobilecoinfoundation/mobilecoin/blob/edbc0162cbd647b6605fe817a8753508ca4515a2/transaction/core/src/tx.rs#L149) is included in the [Tx](https://github.com/mobilecoinfoundation/mobilecoin/blob/edbc0162cbd647b6605fe817a8753508ca4515a2/transaction/core/src/tx.rs#L103) that is delivered directly to the enclave through an attested channel at the [client\_tx_propose](https://github.com/mobilecoinfoundation/mobilecoin/blob/2d5190e60f6820cebcd585c19d16cecf9ba4d89b/consensus/enclave/impl/src/lib.rs#L299) endpoint. By attested, this means that the client would not deliver this payload unless the enclave's signed measurement matched the client's expectations.

The [TxManager](https://github.com/mobilecoinfoundation/mobilecoin/blob/2d5190e60f6820cebcd585c19d16cecf9ba4d89b/consensus/service/src/tx_manager/mod.rs) verifies that the Tx is "well-formed" using a 2-step process. First, untrusted performs a [`well_formed_check`](https://github.com/mobilecoinfoundation/mobilecoin/blob/2d5190e60f6820cebcd585c19d16cecf9ba4d89b/consensus/service/src/validators.rs#L52:8) on the Proofs of Memberships, using a [TxContext](https://github.com/mobilecoinfoundation/mobilecoin/blob/2d5190e60f6820cebcd585c19d16cecf9ba4d89b/consensus/enclave/api/src/lib.rs#L183:12) whose TokenID is contained in the [LocallyEncryptedTx](https://github.com/mobilecoinfoundation/mobilecoin/blob/2d5190e60f6820cebcd585c19d16cecf9ba4d89b/consensus/enclave/api/src/lib.rs#L37:12) constructed by the enclave, and is therefore not in the clear to untrusted. Then the enclave [verifies that the Tx is "well-formed"](https://github.com/mobilecoinfoundation/mobilecoin/blob/cfa51d26ae943a9055698bb209c2fe06fa7a7cac/consensus/enclave/impl/src/lib.rs#L338), and outputs the [WellFormedEncryptedTx](https://github.com/mobilecoinfoundation/mobilecoin/blob/a092039a4a5cc7f5e3e16eeadd0b2bc3a12667ae/consensus/enclave/api/src/lib.rs#L46) and [WellFormedTxContext](https://github.com/mobilecoinfoundation/mobilecoin/blob/a092039a4a5cc7f5e3e16eeadd0b2bc3a12667ae/consensus/enclave/api/src/lib.rs#L50) for untrusted to manage as consensus rounds progress. Note that in this proposal, the WellFormedTxContext no longer contains the fee in the clear, but rather the `fee_ratio` as described in [Fee Ratio and Transaction Sorting](#fee-ratio-and-transaction-sorting).

Untrusted proceeds to [sort the WellFormedTxContext](https://github.com/mobilecoinfoundation/mobilecoin/blob/a092039a4a5cc7f5e3e16eeadd0b2bc3a12667ae/consensus/enclave/api/src/lib.rs#L131) by `fee_ratio` as part of the [nominate phase](https://github.com/mobilecoinfoundation/mobilecoin/blob/cfa51d26ae943a9055698bb209c2fe06fa7a7cac/consensus/scp/src/slot.rs#L613).

Now that the Tx is validated as "well-formed," sorted and combined for a ballot with other transactions, and is a candidate for inclusion in the consensus slot, the WellFormedEncryptedTx is [broadcast](https://github.com/mobilecoinfoundation/mobilecoin/blob/2f90154a445c769594dfad881463a2d4a003d7d6/peers/src/traits.rs#L28) to the node's attested peers as required by the consensus protocol, using the [`peer_tx_propose`](https://github.com/mobilecoinfoundation/mobilecoin/blob/2d5190e60f6820cebcd585c19d16cecf9ba4d89b/consensus/enclave/impl/src/lib.rs#L328) attested endpoint.

The TxManager in untrusted for each node maintains a cache of encrypted txs and tx contexts that its enclave has found to be well-formed. Once the consensus protocol is complete, the untrusted TxManager calls [form_block]() which prompts the [enclave to `form_block`](https://github.com/mobilecoinfoundation/mobilecoin/blob/cfa51d26ae943a9055698bb209c2fe06fa7a7cac/consensus/enclave/impl/src/lib.rs#L421) using the full transaction contents.

Because untrusted never sees the `fee` or `token_id` in the clear, the attacker would need to sustain an enclave exploit, such as a side channel attack, that would allow them to access data while the enclave is processing to discern the `token_id` of a transaction. With this information, the attacker could, for the duration of the exploit, determine which asset type each transaction was composed of, and could also construct a transaction graph for statistical analysis of the inputs, which are also only visible in the TxPrefix, but not persisted to the chain. Once the integrity of SGX was restored, the attacker would no longer be able to obtain information about the transaction inputs or `token_id`, and forward secrecy is preserved. Further, by being thoughtful about the memory access of the `token_id`, and using constant time operations, we can mitigate the data leakage even in the event of a side channel exploit.

### Upgrade Plan

As this is a substantial change to the ledger format and transaction protocol, we need to be thoughtful about what rollout looks like for this change to land with minimal impact on existing clients and maximum backward compatibility.

The upgrade plan is described in full in [MCIP #26](https://github.com/mobilecoinfoundation/mcips/blob/main/text/0026-block-version-based-protocol-evolution.md).

# Drawbacks
[drawbacks]: #drawbacks

This is a substantial change to the protocol, and a breaking change for clients. Other drawbacks include:

- Increasing complexity of TxOuts and transaction construction & validation
- Increasing the amount of data on the ledger
- `masked_token_id` bytes must be plumbed everywhere, in Fog and clients, but no more serious changes are needed in Fog. Note that if the `token_id` was not confidential, Fog Ledger would need a much larger change in order to properly select rings

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- We could use `u16` for the `token_ids` rather than `u32` to save space, but we would like to anticipate the possibility of more than 65,000 asset types. 
- We could use Merlin rather than Blake2B
- We could only accept fees in one asset type, to prevent complication of client logic and enclave management of multiple fee aggregations
- We could use non-confidential token types, but this has two downsides: it breaks the ledger into multiple sets which could be statistically analyzed for `token_id`, and it requires much more logic around post processing the ledger for fog. With the proposal here, fog remains oblivious to `token_id` and need not complicate how it selects rings, for example.

# Prior art
[prior-art]: #prior-art

- [Spats](https://github.com/AaronFeickert/spats) is an extension to the [Spark](https://eprint.iacr.org/2021/1173) transaction protocol to support confidential assets
- [Andrew Poelstra describes Confidential Assets](https://blog.blockstream.com/en-blockstream-releases-elements-confidential-assets/) for Blockstream, along with [this whitepaper](https://blockstream.com/bitcoin17-final41.pdf)

# Unresolved questions
[unresolved-questions]: #unresolved-questions

No unresolved questions at this time.

# Future possibilities
[future-possibilities]: #future-possibilities

No future possibilities at this time.

