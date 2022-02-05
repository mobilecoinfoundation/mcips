- Feature Name: `confidential_token_ids`
- Start Date: 2022-02-05
- MCIP PR: [mobilecoinfoundation/mcips#0025](https://github.com/mobilecoinfoundation/mcips/pull/0025)
- MobileCoin Epic: TBD

# Summary
[summary]: #summary

Enable new token types to be added to the protocol and transacted on chain without revealing or leaving a transparent record of the token type.

# Motivation
[motivation]: #motivation

The MobileCoin ledger protocol currently supports one token type, MOB. We would like to allow the addition of assets to the ledger without revealing which assets were involved in any transaction. The transaction confidentiality and integrity guarantees should be the same for the MobileCoin blockchain no matter how many token types are supported. Namely, the sender, the recipient, and the amount should remain private to all except the participants in the transaction, and no double spending or unauthorized minting should be possible. In addition, we now specify that the asset type should remain private to all except the participants in the transaction.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

A Transaction Output (TXO) has a confidential TokenType enum field (`masked_token_id`). Transactions with inputs and outputs, including a fee output, must be composed of a single TokenType. Some tokens may have additional functionality, such as minting and burning, which require verification of the TokenType as part of the authorization of the action.

Each TokenType has an explicit minimum fee specified by node operators, and included in the attestation handshake, ensuring that all nodes in the network are configured with the same fee minimums and TokenType sets. New TokenTypes can be added only with unanimous agreement from all node operators. 

A transaction specifies the TokenType for the entire transaction, and includes a "Proof of Opening" that all `token_ids` in the inputs and outputs are the same. This prevents spending one TokenType as a member of another TokenType. The TransactionBuilder now takes a `token_id` as an argument in its initializer. The TokenType is set in the clear on the TxPrefix for a transaction, which is only accessed in the enclave, after the payload is delivered via an encrypted, attested channel. This is similar to the TxInputs, and thus has a similar threat model for confidentiality. See [Confidentiality Analysis](#confidentiality-analysis) for an assessment of risk.

Clients must choose how to expose multiple asset types to their users. There may be clients who wish to only expose a subset of asset types, or clients who wish to provide balances and transactionality in all asset types. 

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

### Masked Token ID Field in the TXO

A `TokenType` is added which specifies a `u32 token_id`. The `token_id` for each TXO represents the Token Type of that TXO, with the first `TokenType` being `MOB = token_id(0)`. New TokenTypes increase the `token_id` monotonically. For example, the `TokenType` struct may be something like,

```
struct TokenType {

    \\\ Token Name, such as MOB
    string token_name,
    
    \\\ Token ID to be represented confidentially in each TXO
    u32 token_id,
    
    \\\ Curve base point, unique for each TokenType, from which the TXO's commitment is 
    \\\ generated with the value of the `amount` in the TXO
    RistrettoPoint H_i
}
```

The `masked_token_id` field is added to the [`Amount`](https://github.com/mobilecoinfoundation/mobilecoin/blob/master/transaction/core/src/amount/mod.rs#L28) in each TXO, and functions similar to `masked_value`. The protocol representation of the TXO uses Protobuf, which enables backward compatability in this way: if the field is not populated (Protobuf bytes field of length 0), the default `token_id` is 0. Otherwise, the field is a Protobuf bytes field of length 4. Please see the [Upgrade Plan](#upgrade-plan) for more discussion on rolling out this change with consideration for backward compatibility.

- To compute the 4 byte `masked_token_id` from the `token_id`, we hash the TXO shared secret, with a prefix, XORed with the `token_id`, and take the little endian representation of those bytes.

    ```
    masked_token_id = (token_id ^ Blake2B(token_id_tag | shared_secret)).to_le()
    ```

- To recover the `token_id` from the `masked_token_id`, we reverse the process (if the bytes field is length = 4, otherwise `token_id = 0`).

    ```
    unmasked_token_id = masked_token_id ^ Blake2b(token_id_tag | shared_secret)
    ```

### Proof of Opening: Proof of Knowledge of Uniform Token ID in the Transaction

We would like to be able to prove that all transaction inputs and outputs are using the same `token_ids`. We do this using a non-interactive, zero knowledge Proof of Opening that the sender has the capability to unwrap the sum of output commitments in the transaction, in conjunction with the homomorphic encryption property of [Pedersen Commitments](https://www.cs.cornell.edu/courses/cs754/2001fa/129.PDF), namely that addition on the commitments preserves the additive relationship of the pre-committed values, as long as they are using the same group generator.

- To prove each TXO's commitment to the `amount` value, we use a Pedersen commitment, consisting of a group generator (`H_i`) raised to the value of the `amount`, with a blinding factor to prevent brute-forcing the full range of values (`2^64`). We compute the blinding factor as the blinding group generator raised to a hash of the shared secret for the TXO.

    ```
    blinding_factor = Blake2B(blinding_tag | shared_secret)
    commitment = H_i * Scalar::from(amount) + G * blinding_factor
    ```

    - Note, the group generators for Token Types are computed using hashing to a curve so that they are "orthogonal" for computable adversaries, meaning there is no discernable, nontrivial linear relationship between base points. See our [transaction core group generators reference](https://github.com/mobilecoinfoundation/mobilecoin/blob/master/transaction/core/src/ring_signature/mod.rs#L29). 
    - Note also, the blinding factor base point does not need to be unique per TokenType, so we use the same blinding factor base point for all TokenTypes.
    - Note finally, the Proof of Opening requires all commitments in a transaction were generated using the same group generator, base point `H_i`. If the commitments in input and/or output TXOs in a transaction are constructed with different group generators, and the sender is able to unwrap the sum as specified in the protocol below, it implies that the sender knows a nontrivial linear relationship among the `H_i`, which contradicts the assumption that the `H_i` are orthogonal. 

The non-interactive zero knowledge Proof of Opening protocol is the following: (See [Zero Knowledge Proofs and Commitment Schemes, Page 27](https://www.cs.purdue.edu/homes/ninghui/courses/555_Spring12/handouts/555_Spring12_topic23.pdf))


1. The prover establishes a value, `d`, calculated by hashing their secret values from the commitment (the `token_id` and the `blinding_factor`). Implementation note, we may use [Merlin](https://docs.rs/merlin/latest/merlin/) rather than Blake2B.

    ```
    y = Blake2B(proof_tag_t | token_id)
    s = Blake2B(proof_tag_b | blinding_factor)
    d = G * y + H * s
    ```

2. The non-interactive verifier chooses challenge, `e`, by hashing the commitment and the value `d` from the previous step.

    ```
    e = Blake2B(proof_tag_e | commitment | d)
    ```

3. The prover computes the following, and includes these values in the Proof of Opening:

    ```
    u = y + e * token_id
    v = s + e * blinding_factor
    ```

4. The verifier of the Proof of Opening must verify:

    ```
    G * u + H * v = d + commitment * e
    ```

    - This property follows a similar property of the [Schnorr Proof of Knowledge protocol](https://en.wikipedia.org/wiki/Proof_of_knowledge#Schnorr_protocol)

    
This protocol is non-interactive because the prover anticipates the verifier's `e` value, so requires no interaction from the verifier.

Thus, the Proof of Knowledge could be represented, for example, with:

```
struct ProofOfKnowledge {

    /// Prover value established from hashes of commitment and blinding factor secrets
    CompressedRistretto d,
    
    /// Prover value from the token_id response to the verifier's commitment-derived challenge
    CurveScalar u,
    
    /// Prover value from the blinding_factor response to the verifier's commitment-derived challenge
    CurveScalar v,
}
```

- Implementation note: The proof of knowledge should be constructed at [this line](https://github.com/mobilecoinfoundation/mobilecoin/blob/a9db0bc7bcac9bac1c1232b194003d02bccaf283/transaction/core/src/ring_signature/rct_bulletproofs.rs#L304) in the sign function.

Then, the Transaction Validator need perform the following validation check, after calculating the sum of the outputs (in order to verify the sum of the inputs matches the sum of the outputs, and thus that no value was created or destroyed), at [this line](https://github.com/mobilecoinfoundation/mobilecoin/blob/a9db0bc7bcac9bac1c1232b194003d02bccaf283/transaction/core/src/ring_signature/rct_bulletproofs.rs#L170):

```
verify(G, H, commitment, proof_of_knowledge) {
    e = Blake2B(proof_tag_e | commitment | proof_of_knowledge.d)
    proof_of_knowledge.d + (commitment * e) == G * proof_of_knowledge.u + H * proof_of_knowledge.v
}
```

### Fee Outputs

Currently, the enclave aggregates the fees for multiple transactions in a block, in order to mint a single fee output for the block and conserve space. To support multiple confidential TokenTypes, when minting fee outputs, the enclave must create multiple fee outputs if multiple TokenTypes were used in the block. A consideration here is that we would like for an observer not to discern whether a block contains transactions of multiple asset types by statistically analyzing the number of outputs to derive information about the number of fee outputs. 

A simple proposal is that the enclave discontinue fee aggregation, so that the number of fee outputs scale linearly with the number of transactions in a block (not output TXOs). For example, a block with 3 transactions includes 3 fee outputs. Because each transaction can mint up to 16 output TXOs, there may be a variable number of outputs in the block. 

Note that the untrusted portion of the node software knows how many transactions are in each block, and this is not considered statistically relevant information with respect to confidentiality. In other words, the current fee aggregation behavior is not meant as a confidentiality measure, purely as a space-saving measure.

### Fee Ratio and Transaction Sorting

Currently the [WellFormedTxContext](https://github.com/mobilecoinfoundation/mobilecoin/blob/a092039a4a5cc7f5e3e16eeadd0b2bc3a12667ae/consensus/enclave/api/src/lib.rs#L50) contains the fee, which must be used by untrusted to [sort transactions](https://github.com/mobilecoinfoundation/mobilecoin/blob/a092039a4a5cc7f5e3e16eeadd0b2bc3a12667ae/consensus/enclave/api/src/lib.rs#L131) when [combining transactions](https://github.com/mobilecoinfoundation/mobilecoin/blob/a092039a4a5cc7f5e3e16eeadd0b2bc3a12667ae/consensus/service/src/tx_manager/untrusted_interfaces.rs#L40) for consideration by the enclave, prioritizing higher fees for inclusion in the block during periods of network congestion. In order to keep the `token_id` confidential, we must provide a mechanism for untrusted to accurately sort transactions before asking the enclave to do work on the transactions, without revealing the `token_id`, and at the same time, allowing for different asset types to express differing minimum fee requirements.

To derive a sorting order without revealing the transaction type, the `fee` in the WellFormedTxContext becomes a `fee_ratio`, which is calculated based on the ratio between the minimum fees configured by the node operator on startup. For example, the network operators may have configured the following fees:

```
{
    asset_000_fee = 1,
    asset_111_fee = 2,
    asset_888_fee = 3,
}
```

The ratio between the fees for the `token_ids` is the following, using the least common multiple of the fees:

`token_id` | fee | `fee_ratio` multiplier
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

`transaction_id` | `token_id` | `fee_ratio` multiplier | original fee | `fee_ratio` result
--- | --- | --- | --- | ---
transaction_2 | 000 | 6 | 1.5 | 9
transaction_5 | 888 | 2 | 4 | 8
transaction_1 | 000 | 6 | 1 | 6
transaction_3 | 111 | 3 | 2 | 6
transaction_4 | 888 | 2 | 3 | 6

Note how the two transactions paying more than their minimum fee are sorted to the top, in proportion to how much they are paying above the minimum fee, while all minimum fee transactions have an equal chance of beeing sorted because their `fee_ratio` is equal.

### Performance Impact: Size & Space Analysis

- Each TXO has 6 new bytes allocated in the Amount object for the `masked_token_id` (4 bytes for the value, 2 bytes for Protobuf overhead)
- The RingCT bulletproof signature increases by at least 96 bytes to contain the Proof of Opening (plus Protobuf overhead).
- Each block now contains a number of fee outputs scaling linearly with the number of transactions in a block, as opposed to a single fee output aggregated from multiple transactions in a block.
    - In practice, most clients currently output 2 TXOs, one for the recipient and one for change, unless performing a `split_txo` operation. Output growth on chain is currently dominated by the number of outputs in a transaction, and since blocks typically clear fast enough that there is only one transaction in a block for which to aggregate the fee, this should not have a deleterious effect on the space capacity of the chain, especially given that up to [16 outputs are permitted for every transaction](https://github.com/mobilecoinfoundation/mobilecoin/blob/e74d4c821cb213290975d6d5a9390778661cebfe/transaction/core/src/constants.rs#L17).

### Confidentiality Analysis

The [TxPrefix](https://github.com/mobilecoinfoundation/mobilecoin/blob/a9db0bc7bcac9bac1c1232b194003d02bccaf283/transaction/core/src/tx.rs#L149) contains the `token_id` and the `fee` in the clear. We, and any users of the protocol, need to clearly understand the footprint of this data, and the ways in which an attacker may try to discern the `token_id` of TXOs.

The [TxPrefix](https://github.com/mobilecoinfoundation/mobilecoin/blob/edbc0162cbd647b6605fe817a8753508ca4515a2/transaction/core/src/tx.rs#L149) is included in the [Tx](https://github.com/mobilecoinfoundation/mobilecoin/blob/edbc0162cbd647b6605fe817a8753508ca4515a2/transaction/core/src/tx.rs#L103) that is delivered directly to the enclave through an attested channel at the [client\_tx_propose](https://github.com/mobilecoinfoundation/mobilecoin/blob/2d5190e60f6820cebcd585c19d16cecf9ba4d89b/consensus/enclave/impl/src/lib.rs#L299) endpoint. By attested, this means that the client would not deliver this payload unless the enclave's signed measurement matched the client's expectations.

The [TxManager](https://github.com/mobilecoinfoundation/mobilecoin/blob/2d5190e60f6820cebcd585c19d16cecf9ba4d89b/consensus/service/src/tx_manager/mod.rs) verifies that the Tx is "well-formed" using a 2-step process. First, untrusted performs a [`well_formed_check`](https://github.com/mobilecoinfoundation/mobilecoin/blob/2d5190e60f6820cebcd585c19d16cecf9ba4d89b/consensus/service/src/validators.rs#L52:8) on the Proofs of Memberships, using a [TxContext](https://github.com/mobilecoinfoundation/mobilecoin/blob/2d5190e60f6820cebcd585c19d16cecf9ba4d89b/consensus/enclave/api/src/lib.rs#L183:12) whose TokenID is contained in the [LocallyEncryptedTx](https://github.com/mobilecoinfoundation/mobilecoin/blob/2d5190e60f6820cebcd585c19d16cecf9ba4d89b/consensus/enclave/api/src/lib.rs#L37:12) constructed by the enclave, and is therefore not in the clear to untrusted. Then the enclave [verifies that the Tx is "well-formed"](https://github.com/mobilecoinfoundation/mobilecoin/blob/cfa51d26ae943a9055698bb209c2fe06fa7a7cac/consensus/enclave/impl/src/lib.rs#L338), and outputs the [WellFormedEncryptedTx](https://github.com/mobilecoinfoundation/mobilecoin/blob/a092039a4a5cc7f5e3e16eeadd0b2bc3a12667ae/consensus/enclave/api/src/lib.rs#L46) and [WellFormedTxContext](https://github.com/mobilecoinfoundation/mobilecoin/blob/a092039a4a5cc7f5e3e16eeadd0b2bc3a12667ae/consensus/enclave/api/src/lib.rs#L50) for untrusted to manage as consensus rounds progress. Note that in this proposal, the WellFormedTxContext no longer contains the fee in the clear, but rather the `fee_ratio` as described in [Fee Ratio and Transaction Sorting](#fee-ratio-and-transaction-sorting).

Untrusted proceeds to [sort the WellFormedTxContext](https://github.com/mobilecoinfoundation/mobilecoin/blob/a092039a4a5cc7f5e3e16eeadd0b2bc3a12667ae/consensus/enclave/api/src/lib.rs#L131) by `fee_ratio` as part of the [nominate phase](https://github.com/mobilecoinfoundation/mobilecoin/blob/cfa51d26ae943a9055698bb209c2fe06fa7a7cac/consensus/scp/src/slot.rs#L613).

Now that the Tx is validated as "well-formed," sorted and combined for a ballot with other transactions, and is a candidate for inclusion in the consensus slot, the WellFormedEncryptedTx is [broadcast](https://github.com/mobilecoinfoundation/mobilecoin/blob/2f90154a445c769594dfad881463a2d4a003d7d6/peers/src/traits.rs#L28) to the node's attested peers as required by the consensus protocol, using the [`peer_tx_propose`](https://github.com/mobilecoinfoundation/mobilecoin/blob/2d5190e60f6820cebcd585c19d16cecf9ba4d89b/consensus/enclave/impl/src/lib.rs#L328) attested endpoint.

The TxManager in untrusted for each node maintains a cache of encrypted txs and tx contexts that its enclave has found to be well-formed. Once the consensus protocol is complete, the untrusted TxManager calls [form_block]() which prompts the [enclave to `form_block`](https://github.com/mobilecoinfoundation/mobilecoin/blob/cfa51d26ae943a9055698bb209c2fe06fa7a7cac/consensus/enclave/impl/src/lib.rs#L421) using the full transaction contents.

Because untrusted never sees the `fee` or `token_id` in the clear, the attacker would need to sustain an enclave exploit, such as a side channel attack, that would allow them to access data while the enclave is processing to discern the TokenType of a transaction. With this information, the attacker could, for the duration of the exploit, determine which asset type each transaction was composed of, and could also construct a transaction graph for statistical analysis of the inputs, which are also only visible in the TxPrefix, but not persisted to the chain. Once the integrity of SGX was restored, the attacker would no longer be able to obtain information about the transaction inputs or TokenType, and forward secrecy is preserved. Further, by being thoughtful about the memory access of the `token_id`, and using constant time operations, we can mitigate the data leakage even in the event of a side channel exploit.

### Upgrade Plan

As this is a substantial change to the ledger format and transaction protocol, we need to be thoughtful about what rollout looks like for this change to land with minimal impact on existing clients and maximum backward compatibility.

The rollout proposal is the following:

1. The enclaves are released containing the validation change to support Proof of Opening verification and the `masked_token_id` field in the ledger format for TXOs, starting a 90 day grace period timer.
2. Clients update to:
    - at a minimum:
        - construct transactions with the Proof of Opening and the `masked_token_id` in output TXOs, not allowing any new outputs to use the default empty bytes array for the `masked_token_id` field.
        - default to `token_id = 0` when processing TXOs without a `masked_token_id` field
        - obtain the `minimum_fee` for the default token (MOB)
    - optionally expose multiple asset types by:
        - obtain the `minimum fee` for all assets
        - support constructing transactions with arbitrary `token_id`
        - display balance for some set of `token_ids`
3. The consensus validator nodes are deployed and allow for both new style and old style transactions to be constructed, validated, and externalized to the ledger for some grace period. However, no new outputs can be spent in old-style transactions, and new outputs must have an explicit `token_id` even if it's an old-style transaction
4. After some event, such as a particularly specified minting transaction for a new asset type, the validators no longer allow new outputs to be spent in old-style transactions, and all new transactions must contain the Proof of Opening.
5. From this time forward, new asset types may be minted without an enclave upgrade.

# Drawbacks
[drawbacks]: #drawbacks

This is a substantial change to the protocol, and a breaking change for clients. Other drawbacks include:

- Increasing complexity of TXOs and transaction construction & validation
- Increasing the transaction validation footprint in the enclave to include Proof of Opening
- Increasing the amount of data on the ledger
- `masked_token_id` bytes must be plumbed everywhere, in Fog and clients, but no more serious changes are needed in Fog. Note that if the TokenType was not confidential, Fog Ledger would need a much larger change in order to properly select rings

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- We could use `u16` for the `token_ids` rather than `u32` to save space, but we would like to anticipate the possibility of more than 65,000 asset types. 
- We could use Merlin rather than Blake2B
- We could use non-confidential token types, but this has two downsides: it bifurcates the ledger into two sets which could be statistically analyzed for `token_id`, and it requires much more logic around post processing the ledger for fog. With the proposal here, fog remains oblivious to `token_id` and need not complicate how it selects rings, for example.

# Prior art
[prior-art]: #prior-art

- This proposal contains a non-novel commitment scheme as specified in [Zero Knowledge Proofs and Commitment Schemes, Page 27](https://www.cs.purdue.edu/homes/ninghui/courses/555_Spring12/handouts/555_Spring12_topic23.pdf) and uses properties of [Pedersen Commitments](https://www.cs.cornell.edu/courses/cs754/2001fa/129.PDF) for the Proof of Opening
- [Andrew Poelstra describes Confidential Assets](https://blog.blockstream.com/en-blockstream-releases-elements-confidential-assets/) for Blockstream, along with [this whitepaper](https://blockstream.com/bitcoin17-final41.pdf)

# Unresolved questions
[unresolved-questions]: #unresolved-questions

No unresolved questions at this time.

# Future possibilities
[future-possibilities]: #future-possibilities

No future possibilities at this time.

