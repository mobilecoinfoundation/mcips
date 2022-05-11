- Feature Name: `confidential_token_ids`
- Start Date: 2022-02-05
- MCIP PR: [mobilecoinfoundation/mcips#0025](https://github.com/mobilecoinfoundation/mcips/pull/0025)
- Implementation PR: [mobilecoinfoundation/mobilecoin#1438](https://github.com/mobilecoinfoundation/mobilecoin/pull/1438)

# Summary
[summary]: #summary

Enable new token types to be added to the protocol and transacted on chain without revealing the token type.

# Motivation
[motivation]: #motivation

The MobileCoin ledger protocol currently supports one token type, MOB. We would like to allow the addition of assets to the ledger without revealing which assets were involved in any transaction. The transaction confidentiality and integrity guarantees should be the same for the MobileCoin blockchain no matter how many token types are supported.

Specifically, for both MOB and synthetic assets,
* the sender
* the recipient
* the amount

of the transaction should remain private to the sender and recipient of the transaction.

In addition we would also like that the token id should remain private.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

A `token_id` is a 64-bit number. Every `TxOut` will now have a corresponding `token_id`, and this will be committed to the chain via a new `masked_token_id` field.
The `token_id` of a `TxOut` is recovered by a `TxOut` owner in a similar way to the `value`, during view key matching.

The transaction object may now spend inputs and outputs of any token id, but in a single transaction all inputs and outputs must have the same `token_id`.

The fee for such a transaction is also paid in that `token_id`. This is now specified in the `fee_token_id` field in the `TxPrefix`, and implies the token id for all inputs and outputs.

The minimum fee varies from token id to token id. This is configured by network operators at startup, and hashed into the responder id during node-to-node attestation to ensure that the network
has one uniform value for minimum fee.

Previously, the fee of a `Tx` is exposed directly to the nodes (outside of the enclave) so that they can sort transactions by fee. After this change, the fee value may reveal information about the
token id, because different tokens have different minimum fees. To mitigate this, the enclave divides the fee by the minimum fee, deriving a new value called `priority` which is revealed to the node
to sort the transactions.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

### Recap

Recall that in current revisions of MobileCoin, the value of a `TxOut` impacts the fields of the `TxOut` in two ways:

* It appears in the masked value, where it bytes have been XOR'ed with the result of hashing the `TxOut` shared-secret
* It appears in the Pedersen commitment (`Amount::Commitment`), which has the form `v * H + v_blinding * G`

Here `H` is a curve point derived by hashing to curve, and `G` is the ristretto basepoint. `v_blinding`, the blinding factor,
is also derived by a hash of the `TxOut` shared secret.

During view key matching, the recipient of a `TxOut` attempts to compute the shared secret, by key exchange with their view private key
against the `TxOut` public key. They cannot confirm that this shared secret computation was successful -- however, if it is successful,
then they can infer the value mask and the `v_blinding` value. They can use the value mask and the masked value to compute what `v` must be,
and they can then infer what the entire commitment should be. They can then check if this matches the commitment appearing in the `TxOut`. If
it does, then view key matching was successful.

### Changes to TxOut and amount commitments

We propose to similarly encode the token id in two places:

* The `masked_token_id` is a new bytes field which contains the little endian bytes of the token id, XOR'ed with a value derived by Blake2b hashing of the shared secret.

    ```
    masked_token_id = (token_id.to_le_bytes() ^ Blake2B(AMOUNT_TOKEN_ID_DOMAIN_TAG || shared_secret))
    ```

  For backwards compatibility, clients interpret `TxOut`'s with an empty `masked_token_id` as having token id zero. So, zero is the token id of MOB.

* The base point `H` in the Pedersen commitment now depends on the token id.

  For backwards compatibility, we must ensure that `H_0` is the same as the generator currently used for MOB.
  
  ```
  H = RistrettoPoint::from_hash(Blake2b(HASH_TO_POINT_DOMAIN_TAG || RISTRETTO_BASEPOINT_BYTES ^ token_id))
  ```

  So we now have an amount commitment of the form

  `v * H_i + v_blinding * G`

  derived using the value, token id, and blinding factor.
  
  It is important for us that computing `H_i` from the token id can be done in constant-time.

View key matching is similarly extended:
* At the same time that we unmasked the masked value, we also attempt to unmask the token id.
* We confirm successful unmasking as before, using all of the value, token id, and blinding to derive the commitment, and reject if it is not a match.

In this way, every `TxOut` that a client recovers now has both a value and a token id.

The `masked_token_id` will be forbidden before block version, and required after block verison 2.

### Token id configuration

In order for the enclave to handle a token id, it must know the minimum fee value for that token id.
This is configured using a `tokens.toml` file which is provided at startup to the node.

The data in the `tokens.toml` file which goes to the enclave is hashed into the responder id. This ensures that
nodes that do not have the same configuration will fail to attest to eachother, and so the working consensus network
agrees about what tokens have been configured and what the minimum fees are.

Attempts to send transactions using tokens that haven't yet been configured are rejected, at the time that this fee lookup fails.

All operations involving looking up minimum fees must be constant-time in the enclave to avoid revealing token ids over side-channels.

### Fee Outputs

Currently, the enclave aggregates the fees for multiple transactions in a block, in order to mint a single fee output for the block and conserve space. To support multiple confidential `token_ids`, when minting fee outputs, the enclave must create multiple fee outputs if multiple `token_ids` were used in the block.

Naively, we could simply aggregate the fees as before, creating only exactly as many new fee outputs as are needed. However, this means that in a block with two transactions, the number of fee outputs revealed if these two users had the same token id or a different token id. If one of the users is the attacker, then this is a source of information leakage about the other user's token id.

Another way to aggregate fees would be to always mint the maximum possible number of fee outputs, one for each configured token. However, this could massively increase the ratio of fee outputs and massively increase the size of the blockchain for no good reason.

We propose a more subtle strategy, which is that, the number of fee outputs is always `min(num_token_ids, num_transactions_in_block)`. This means that in the regime of blocks with only one transaction,
there is still only one fee output, and in a block with two transactions, there are always two fee outputs, one of which will be zero if the two transactions used the same fee token id. (This is there to
prevent the information leakage mentioned earlier.) In the regime of big blocks, the `min` will always select `num_token_ids` and will always be creating a fee output for every configured token id.
This is the worst-case anyways since in a big block it is possible that every configured token id is used.

### Fee Priority and Transaction Sorting

Currently, the enclave returns a [WellFormedTxContext](https://github.com/mobilecoinfoundation/mobilecoin/blob/a092039a4a5cc7f5e3e16eeadd0b2bc3a12667ae/consensus/enclave/api/src/lib.rs#L50) object to the untrusted side, which contains the minimal data such as the TxHash and the fee which untrusted needs to handle it. The fee value is used both in the priority queue that holds submitted transactions
before they are proposed to the network, and in the combine step, always prioritizing higher fees.

However, since different tokens may have different scaling factors and different minimum fees, the fee may reveal quite a bit about the token id being used.
To try to remove this information, while still permitting effective sorting, we make the following assumptions:

* Minimum fees will, over time, be calibrated across tokens so that the value of a minimum fee in one token is similar to the value of a minimum fee in another token.
  This is necessary because the smallest minimum fee is the most likely DOS vector, so we don't want any of them to be much smaller than the others.
* Revealing the ratio of the Tx fee to the minimum fee still allows effective sorting.

If we term this ratio as the `priority` of a Tx, then `token_id` and the `priority` together determine the `fee`. In this sense we have factored the `fee` into two separate pieces of information. We keep the `token_id` as secret to the enclave, but we reveal the `priority` to allow for effective transaction prioritization. These `priority` numbers may ultimately be exposed to users as well so that they can predict what fee is needed to clear the network.

This means for instance that, any two `Tx` submitted at the same priority level are indistinguishable to untrusted even if they use different token ids.

To actually compute the priority, we used the following approach:
* We implemented a constant-time u64 division algorithm for the enclave, so that fee can be divided by minimum fee in constant time.
* Before dividing fee by minimum fee, we divide minimum fee by 128.

The reason to divide minimum fee by 128 is that, currently the network is not heavily loaded and all transactions pay the minimum fee.
If the network were to become loaded, then the smallest fee that would have a higher priority than minimum fee is `minimum_fee * 2`, since
we round down after dividing.

Dividing by 128 first means that all priority values are at least 128, and increments of 1% of the `minimum_fee` lead to a difference in the computed priority value.
This is thought to increase flexibility in the fee market and help avoid large increases in the fee if the network is under load.

This also imposes a rule that the minimum fee for any token cannot be less than 128. However, this creates no significant problems.

#### Examples

For example, the network operators may have configured the following minimum fees:

```
[[tokens]]
token_id = 0
minimum_fee = 400000000

[[tokens]]
token_id = 1
minimum_fee = 1024
[tokens.governors]
signers = """
$GOVERNOR_PUBKEY_1
"""
threshold = 1

[[tokens]]
token_id = 2
minimum_fee = 2048
[tokens.governors]
signers = """
$GOVERNOR_PUBKEY_2
```

These minimum fees, after being divided by 128, have these values:

`token_id` | `minimum_fee / 128`
--- | ---
0 | 31250000
1 | 8
2 | 16

Then, imagine that a block contains the following set of transactions with the given fees:

```
[
    transaction_1 {fee_token_id: 0, fee: 4000000000},
    transaction_2 {fee_token_id: 0, fee: 6000000000},
    transaction_3 {fee_token_id: 1, fee: 1024},
    transaction_4 {fee_token_id: 2, fee: 2048},
    transaction_5 {fee_token_id: 2, fee: 4096}
]
```

The priority computation sort order would be:

`transaction_id` | `token_id` | original `fee` | `minimum_fee / 128` | `priority` result
--- | --- | --- | --- | ---
transaction_5 | 2 | 4096 | 16 | 256
transaction_2 | 0 | 6000000000 | 192
transaction_3 | 1 | 1024 | 8 | 128
transaction_4 | 2 | 2048 | 16 | 128
transaction_1 | 0 | 4000000000 | 128

Note how the two transactions paying more than their minimum fee are sorted to the top, in proportion to how much they are paying above the minimum fee, while all minimum fee transactions have an equal chance of beeing sorted because their `priority` is equal.

For users who are adjusting the fees of their submitted transactions to increase the likelihood of their transaction being accepted during times of congestion, they can observe the `priority` multiplier for their `token_id`, and the recently accepted `priority` values, and determine what fee value is appropriate to increase the chances of acceptance.

### Extended message digest

Finally, we would like to make a cryptography improvement around the extended message which is signed by Ring MLSAGs.
Instead of signing the concatenation of the `TxPrefix` digest with numerous amount commitment and range proof bytes,
we would prefer to hash this all in a structured way using Merlin. This is being called the `extended_message_digest`
and starting in block version 2 this will always be used for signing when creating and validating Ring MLSAGs, replacing
the old `extended_message`.

# Drawbacks
[drawbacks]: #drawbacks

This is a substantial change to the protocol, and a breaking change for clients. Other drawbacks include:

* Increasing complexity of `TxOut` and transaction construction & validation
* Increasing the amount of data on the ledger
* `masked_token_id` bytes must be plumbed everywhere, in Fog and clients, but no more serious changes are needed in Fog. Note that if the `token_id` was not confidential, Fog Ledger would need a much larger change in order to properly select rings

Compared to a design like Spats, our choice has some drawbacks:

* Our set of generators is not fixed at build time, but instead there is a limitless pool of Pedersen generators that could be used for tokens.
  The enclave must now compute these at runtime, which may cause significant overhead. If we attempt to cache them, the caching strategy must be oblivious,
  or somehow arranged so that the cache operation does not reveal which token id was used.
* The enclave necessarily knows the token id of every input and output. In an SGX compromise scenario this data is revealed.
* To support mixed transactions in the future, we would need to have a bulletproof per token id used, for present revisions of bulletproofs.
  This is because our current bulletproofs implementation only supports one generator at a time. In the future this could perhaps be improved.

Choosing to allow fees in tokens other than MOB creates some privacy problems, because this design means that the fee token id is necessarily the same as the
transaction token id, and the fee token id may be easier to infer.

* The fee token id is known to the MobileCoin foundation.
* The priority number which is revealed seems effective for purposes of preventing a passive adversary from inferring the token id from the priority, but
  not necessarily an active one. An adversary which is capable of changing minimum fees on nodes *and* causing user transactions to fail and then be resubmitted
  may be able to infer the token id by inferring if priority numbers change when they do this.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

## Confidentiality Analysis

* This design reveals no information about the sender, recipient, amount, or token id to a passive adversary who cannot penetrate SGX.
* It is impossible for an adversary who actively submits transactions at the same time as another user, and has visibility into the blockchain
  and into enclave memory access patterns, to infer the token id of the other user's transaction.
* There is a token id oracle for an adversary who has root access on a node to which users are submitting transactions. (However in our opinion this
  attack is likely not very practical.)
* The token id is revealed to the MobileCoin foundation which receives the fees, so in blocks with a single transaction, this data is not confidential.

## Performance Impact: Size & Space Analysis

- Each TxOut has 10 new bytes allocated in the Amount object for the `masked_token_id` (8 bytes for the value, 2 bytes for Protobuf overhead)
- Each block now contains a number of fee outputs capped by `min(num_token_ids, num_transactions_in_block)`

The size overhead has the most significant impact on Fog, which has to store `TxOut`'s in oblvious RAM.
The number of fee outputs is still one for any block with a single transaction. For blocks with a large number of transactions, fee aggregation
still ensures that there are at most `num_token_id`'s in the block, which is the minimum which could be theoretically required. In between, there
may be some blocks with more fee output's than woudl be required.

## Changes contemplated

* We could use a smaller integer for token types, to save space, but we would like to anticipate the possibility of more than 4 billion asset types
* We could use Merlin rather than Blake2B at various places. We mostly tried to follow existing patterns in the design.
* We could use non-confidential token types to avoid implementation complexity, but this actually adds implementation complexity: it requires much more logic around post-processing the ledger for fog. With the proposal here, fog remains oblivious to `token_id` and need not complicate how it selects rings, for example.

## Why not do amount commitments the Spats way

The spats way of doing amount commitments differs from ours -- instead of doing "vector Pedersen commitments" with a "basis vector" for each token id (and the blinding factor),
Spats prefers to introduce a single basis vector for token ids, and then essentially a second blinding factor over these. Since it is no longer sound to simply add the amount
commitments, every commitment has to have its token id component subtracted before balance checking can be performed.

There are advantages and disadvantages here.

Advantages of this method:

* Token Id values would be confidential even from the enclave
* Fewer elliptic curve operations to create generators (since caching them may cause cache timing attacks).
* Generalizes to mixed transactions easily without requiring multiple bulletproofs.

Drawbacks of this method:

* Additional elliptic curve operations on client and server side to deal with the extra token id blinding factors and zeroing out of token id components.
* More data sent per Tx
* May prevent us from using cryptographic schemes which would need vector Pedersen commitments, or would be simpler if we had vector Pedersen commitments.

For example, if I want to prove to you that the sum of one set of outputs is equal to some fraction of the value of some other set of outputs, in a vector Pedersen
commitment world, I can multiply both sides and add, and then reveal a commitment to zero. When the token id is instead arranged as another value, these token id
components have to be zeroed before we can multiply, and they have to be grouped by token id before we can add. This may require revealing more information.
There may be other situations where using vector Pedersen commitments will be helpful. (On the other hand, there is a risk that this will be less useful than we think.)

## Why not allow fees only in MOB

Advantages of allowing fees only in MOB:

* Mitigates privacy concerns around the token id oracle attack
* Simplifies the task of ensuring that fees are effectively rate limiting the network
* Simplifies the enclave code, since less constant-time handling is required around fees and fee token ids
* Ensures that MOB has a central role as the gas currency of the network

Disadvantages of allowing fees only in MOB:

* Requires users to obtain MOB in order to use another currency sent to them by their friend. This adds complexity to the user experience. This is our overriding concern.
* Requires us to implement mixed transactions support in order to ship synthetic assets at all. This increases the scope of the implementation task.

Additionally, if we are able to support mixed transactions later, then revealing the fee token id is less interesting, since the user need not be paying the fee
in the token they are actually sending. This makes the fee token id oracle less significant in the longer term.

# Prior art
[prior-art]: #prior-art

There are many blockchains that support synthetic assets, and it is worth referring to the Ethereum approach, as well as others
- [Ethereum ERC-20 token standard](https://ethereum.org/en/developers/docs/standards/tokens/erc-20/)
- [Algorand standard assets](https://developer.algorand.org/docs/get-details/asa/)
- [Cardano Native tokens ](https://docs.cardano.org/native-tokens/learn)

Our approach resembles Algorand and Cardano much more than Ethereum. It is interesting that even though they have smart contracts, they thought
that a native asset approach was superior to the ERC-20 approach.

It is perhaps more interesting to compare with other confidential asset type proposals:
- [Spats](https://github.com/AaronFeickert/spats) is an extension to the [Spark](https://eprint.iacr.org/2021/1173) transaction protocol to support confidential assets
- [Andrew Poelstra describes Confidential Assets](https://blog.blockstream.com/en-blockstream-releases-elements-confidential-assets/) for Blockstream, along with [this whitepaper](https://blockstream.com/bitcoin17-final41.pdf)

# Unresolved questions
[unresolved-questions]: #unresolved-questions

No unresolved questions at this time.

# Future possibilities
[future-possibilities]: #future-possibilities

In the future, we would like to use MCIPs to introduce proposals that would actually assigning token ids to newly proposed synthetic assets.
