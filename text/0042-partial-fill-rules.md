- Feature Name: Partial Fill Rules
- Start Date: 2022-06-02
- MCIP PR: [mobilecoinfoundation/mcips#0042](https://github.com/mobilecoinfoundation/mcips/pull/0042)
- Tracking Issue: [mobilecoinfoundation/mobilecoin#2081](https://github.com/mobilecoinfoundation/mobilecoin/issues/2081)

# Summary
[summary]: #summary

Extend the set of supported "input rules", to allow that someone can produce a signed quote indicating willingness to trade "this for that", but moreover, "this for that, at this exchange rate, up to this volume".

# Motivation
[motivation]: #motivation

In [MCIP #31](https://github.com/mobilecoinfoundation/mcips/blob/main/text/0031-transactions-with-contingent-inputs.md) we proposed a way that two parties can swap assets -- the first one can make a signed offer to trade this for that using a "Signed Contingent Input", then anyone who agrees can add this to their transaction.

The way it was proposed, the first party has to state the total amount of the first asset and
the total amount of the second asset. But if the second party doesn't agree exactly, then they
may not proceed, so there is a need to negotiate exactly the price and the volume.

One way to make this more practical is that, the first party could instead sign that they are willing to trade at a particular price, up to a particular maximum volume. We call this a "partial fill", in analogy with partial fill limit orders in traditional exchanges.

**Note** however, that this is not conceptually an implementation of an "exchange" which accepts and then determines settlement of orders. It's more like a fancy quoting service, wherein one party can offer a signed quote which another party may act on or ignore.

We want to extend the [MCIP #31](https://github.com/mobilecoinfoundation/mcips/blob/main/text/0031-transactions-with-contingent-inputs.md) functionality to support this.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

An input rules object may now optionally contain a "partial fill rules" specification.

A partial fill specification includes:

* A list of TxOut's that may be partially filled. Each tx out here is called a "partial fill output".
* A partial fill change output. This sends any left-over value from the signed input to the originator when the quote is partially filled.
* A minimum fill value. This is configurable by the originator and allows them to prevent filling the quote for dust amounts, which can be a form of griefing.

These partial fill rules are specified (and signed) by the **originator** of an SCI, who is conceptually offering a price quote.

When a Signed Contingent Input (SCI) containing partial fill rules is incorporated into a transaction by the **counterparty** to a swap, there is an implicit
"partial fill fraction", which is the fraction of the maximum allowed volume that is actually transacted when the transation settles.

When a transaction contains partial fill rules, they must be satisfied, or the transaction is invalid, and this is enforced by the consensus network.

To satisfy partial fill rules, all partial fill outputs and the partial fill change output must appear in the TxPrefix, similarly as `required_outputs` in [MCIP #31](https://github.com/mobilecoinfoundation/mcips/blob/main/text/0031-transactions-with-contingent-inputs.md). However, the amounts of these corresponding TxOut's are allowed to be different -- every other field must be the same. These modified TxOut's which are actually the ones which will enter the blockchain ledger, are called the "fractional outputs" and "fractional change output". These TxOut's are created by the **counterparty** by modifying the TxOut's which the originator signed. Essentially, the amounts are all modified according to the **partial fill fraction** which the counterparty desires. Transaction validation must confirm that the amounts are different in an acceptable way, making sure that each fractional output is large enough.

An example partial fill flow goes like this:
* Alice (the originator) has an output for 1500 MOB. Alice would like to trade up to 1000 MOB for MEOWB at a price of 10 MEOWB/MOB.
  Alice doesn't want the trade to happen unless the counterparty takes at least 0.1 MOB.
* Alice creates an SCI using her 1500 MOB input.
  * She creates a required output to herself for 500 MOB
  * She creates a fractional output to herself for 10,000 MEOWB
  * She creates a fractional change output to herself for 1000 MOB
  * She sets the `min_fill_value` to 0.1 MOB
* Bob (the counterparty) sees this quote, and wants to trade 200 MEOWB for 20 MOB (+ transaction fee).
  * Bob adds this quote to the transaction builder, and indicates that he wants to fill 2% of the quote.
  * Bob adds an input worth at least 200 MEOWB + transaction fee.
  * Bob adds a change output with any MEOWB change to himself.
  * Bob assigns a 20 MOB output to himself.

When the transaction settles, Bob has paid net 200 MEOWB + transaction fee, and received net of 20 MOB.
Alice has paid a 1500 MOB output, and received the 500 MOB required output, and a 980 MOB fractional output,
as well as 200 MEOWB from Bob.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The reference level explanation splits into two parts:

1. The [Amount shared secret](#Amount-shared-secret)
2. The [Partial fill rules schema and semantics](#Partial-fill-rules-schema-and-semantics) of the new SCI fields

Neither of these are legal before block version 3. The new SCI fields are legal in block version 3, and the alternate derivation for
`TxOut` `MaskedAmount` must be used after block version 3.

## Amount shared secret

In order to verify that partial fills were performed correctly, we must reveal the amounts of all fractional outputs (which correspond to partial fill outputs) in the `TxPrefix` to the enclave. To do this, we reveal the secret which underlies both the amount commitment and the masked value, so that the enclave can essentially do view-key matching on these TxOut's. A new mechanism is specified to do precisely this, called Amount shared secret.

In order to avoid revealing unnecessary information, we modify the derivation of the value mask and the amount blinding factor of the Pedersen commitment from the TxOut shared secret. We introduce a second step, with a second key called the Amount shared secret.

Before:
```mermaid
graph TB
    A[TxOut SharedSecret] -->|hkdf sha512| B[MemoKey]
    A[TxOut SharedSecret] -->|blake2b| C[Confirmation Number]
    A[TxOut SharedSecret] -->|blake2b| D[Amount Blinding Factor]
    A[TxOut SharedSecret] -->|blake2b| E[Value mask]
```

After:
```mermaid
graph TB
    A[TxOut SharedSecret] -->|hkdf sha512| B[MemoKey]
    A -->|blake2b| C[Confirmation Number]
    A -->|sha512| D[Amount Shared Secret *new*]
    D -->|hkdf sha512| E[Amount Blinding Factor v2]
    D -->|hkdf sha512| F[Value mask v2]
```

The purpose is that instead of revealing the TxOut shared secret to the enclave, we can reveal the amount shared secret,
which permits unmasking the masked value and confirming against the Pedersen commitment, without revealing anything that
can impact the memos or the confirmation numbers.

**Note** that, this deviates from the model for "normal" transactions. For normal transactions, amounts are not revealed
even to the enclave, because it is unnecessary, and we can use RingCT to validate transactions without revealing the amounts. However, for partial fill swap transactions, the use case is that SCI's are broadcast
to an exchange network -- the goal there is that both the originator and the counterparty are anonymous, but not that the amounts being offered to transact are secret. So when validating these swaps, keeping the amounts secret from the consensus enclave does not impact the threat model.

For compatibility, we propose that the `TxOut` structure is evolved per [MCIP #26](https://github.com/mobilecoinfoundation/mcips/blob/main/text/0026-block-version-based-protocol-evolution.md), so that
client software can support both the old and new derivation schemes. The old scheme must always be used
prior to block version 3, and the new scheme must always be used after block version 3.

Example rust code for computing the new derivations is as follows:

```rust
/// Computes the amount shared secret from the tx_out shared secret
fn get_amount_shared_secret(tx_out_shared_secret: &RistrettoPublic) -> [u8; 32] {
    let mut hasher = Sha512::new();
    hasher.update(&AMOUNT_SHARED_SECRET_DOMAIN_TAG);
    hasher.update(&tx_out_shared_secret.to_bytes());
    // Safety: Sha512 is a 512-bit (64-byte) hash.
    hasher.finalize()[0..32].try_into().unwrap()
}
```

```rust
/// Computes the value mask, token id mask, and blinding factor for the
/// commitment, in a masked amount.
///
/// # Arguments
/// * `amount_shared_secret` - The amount shared secret, derived as a hash of
///   `rB`.
fn get_blinding_factors(amount_shared_secret: &[u8; 32]) -> (u64, u64, Scalar) {
    // Use HKDF-SHA512 to produce blinding factors for value, token id, and
    // commitment
    let kdf = Hkdf::<Sha512>::new(
        Some(AMOUNT_BLINDING_FACTORS_DOMAIN_TAG),
        amount_shared_secret,
    );

    let mut value_mask = [0u8; 8];
    kdf.expand(AMOUNT_VALUE_DOMAIN_TAG.as_bytes(), &mut value_mask)
        .expect("Digest output size is insufficient");

    let mut token_id_mask = [0u8; 8];
    kdf.expand(AMOUNT_TOKEN_ID_DOMAIN_TAG.as_bytes(), &mut token_id_mask)
        .expect("Digest output size is insufficient");

    let mut scalar_blinding_bytes = [0u8; 64];
    kdf.expand(
        AMOUNT_BLINDING_DOMAIN_TAG.as_bytes(),
        &mut scalar_blinding_bytes,
    )
    .expect("Digest output size is insufficient");

    (
        u64::from_le_bytes(value_mask),
        u64::from_le_bytes(token_id_mask),
        Scalar::from_bytes_mod_order_wide(&scalar_blinding_bytes),
    )
}
```

At the level of protobuf, the `TxOut.masked_amount` field is replaced with a `oneof`, selecting between V1 and V2.
`MaskedAmountV1` uses the historical derivations and must be supported by all clients indefinitely, as outputs from block versions <=2 always use this derivation.
`MaskedAmountV2` uses the new derivation and is mandatory for all `TxOut`'s from block version 3 and on.

## Partial fill rules schema and semantics


The proposed changes to the `TxIn` structure and the input rules are captured in this diagram:

```mermaid
classDiagram
direction BT
InputRules --> TxIn
RevealedTxOut --> InputRules

TxIn: Vec~TxOut~ ring
TxIn: Vec~...~ membership_proofs
TxIn: Option~InputRules~ input_rules

InputRules: Vec~TxOut~ required_outputs
InputRules: uint64 max_tombstone_block
InputRules: Vec~RevealedTxOut~ partial_fill_outputs *new*
InputRules: Option~RevealedTxOut~ partial_fill_change *new*
InputRules: uint64 min_partial_fill_value *new*

RevealedTxOut: TxOut tx_out
RevealedTxOut: bytes amount_shared_secret
```

Recall that the `InputRules` are created (and signed) by the **originator**, and this TxIn is added to a transaction
by the **counterparty**, who is also obligated to add TxOut's to the TxPrefix which match to the partial fill change
and partial fill outputs appropriately. (This is actually done under the hood by the transaction builder.)

The consensus enclave must enforce any input rules, including these new partial fill rules. This is done in the following way.

1. If `partial_fill_change` is not present, then no other partial fill rules may be used, and any other partial fill fields must be defaulted.
1. If `partial_fill_change` is present, then decrypting the amount and token id using `amount_shared_secret` must succeed. The partial fill change
   may not have a 0 value (to help clients avoid division by zero).
1. If `partial_fill_change` is present, then a corresponding TxOut called the `fractional_change_output` must exist in the `TxPrefix.outputs`.
   This must match the `partial_fill_change` in every field except possibly the `masked_amount`.
1. The `fractional_change_output` must decrypt successfully using the `amount_shared_secret` of the `partial_fill_change`, and
   it's amount must have the same token id as the `partial_fill_change`, and its value must be less than or equal to the `partial_fill_change`
1. The fill fraction is inferred at this point.
   * The numerator is `partial_fill_change.value - fractional_change_output.value`.
   * The denominator is `partial_fill_change.value`.
1. The minimum fill rule is enforced: `partial_fill_change.value - fractional_change_output.value >= min_partial_fill_value`.
   If this inequality does not hold, then the transaction is rejected for reasons of not meeting the minimum partial fill prescribed
   by the originator.
1. For each `partial_fill_output`:
   * It must decrypt succesfully using the recorded `amount_shared_secret`.
   * There must be a corresponding `fractional_output` among the `TxPrefix.outputs` which matches it in every field except possibly
     the `masked_amount`.
   * The fractional output must decrypt successfully using the `partial_fill_output`'s `amount_shared_secret`, and must match the
     token id of the `partial_fill_output`.
   * It must be the case that `fractional_output.value >= n/d * partial_fill_output.value` where `n` and `d` are the fill fraction
     numerator and denominator. This equation must be validated by clearing denominators and checking an inequality of `u128`'s, to
     avoid numerical issues.
     
     ```
     fractional_output.value * fill_fraction_denominator >= fill_fraction_numerator * partial_fill_output.value
     ```
     
     If this inequality does not hold, then the transaction must be rejected for reasons of not respecting the fill fraction.

# Drawbacks
[drawbacks]: #drawbacks

This proposal has a few drawbacks.

* The amounts of partial fill transactions are revealed to the enclave. Currently, MobileCoin transactions are validated using RingCT and do not reveal amounts to the enclave. It would be better if there were a way to validate partial fills without revealing this to the enclave, but for the use cases envisioned, this data already has to be revealed anyways, so the use cases aren't negatively impacted. We could imagine that an exchange service is based on SGX, and so the amounts of these quotes are only revealed inside of SGX, and are matched inside of SGX. Even in this case, it is still the case that if you penetrate SGX you will see the amounts of the SCI quotes.
* Partial-fill SCI quotes cannot be re-used by a second counterparty later if the first counter-party only matched some of the volume.
In a traditional exchange, limit orders that are partially-filled remain on the books until being totally filled, so that potentially dozens of different parties can each match a piece of the liquidity provided by the originator. In this proposal, that would not work, because as soon as the first counterparty matches the order, the key image underlying the SCI is burned. One could imagine that there might be a way that the fractional change output which goes back to the originator could be automatically signed as an SCI somehow, so that matching can continue without the originator being in the loop. This proposal instead envisions that if the originator wants to continue matching they simply sign a new SCI themselves and re-broadcast.
* The same input can be used to "back" multiple SCI quotes at once, in which case if any one of them is matched, the others instantly become impossible to match. This is somewhat different from how limit orders work in an exchange -- each limit order must be backed by distinct liquidity. This reflects the idea that SCI's are more in analogy with a signed quote than with a limit order as such. This is potentially confusing to e.g. a trader who looks at a list of available SCI quotes and infers that there is a certain amount of liquidity depth, reasoning as they would in a traditional exchange. But if the same inputs are used to sign multiple quotes e.g. for different currencies, it may be impossible that all of these quotes could actually match, and the "real" liquidity may be considerably less. This can be mitigated by making sure that the software they are using communicates this idea clearly.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

An alternative design which we explored first attempted to use homomorphic properties of Pedersen commitments to verify correct partial fills without revealing them:

* Amount shared secret is not needed.
* An "Input rule verification data" field is added to TxIn, and contains the fill fraction, expressed as a u64 `numerator` and a u64 `denominator`.
* To verify equations like `real_output >= num / denom * fractional_output`, we clear denominators and convert to a form `X >= 0`.
  `denominator * real_output - num * fractional_output >= 0`. Then we consider the LHS as a "partial fill verification commitment"
  and fold this into the range proofs that already exist in the transaction. It is not too hard to see that the counter-party can always
  ensure that this number is in the range `[0, u64::MAX)` for a valid transaction.
* To verify the fractional change output, we need to verify an equation `(fraction_change - real_change) >= num / denom * real_change`.
  This could similarly be verified without revealing commitments by clearing denominators and converting to the form `X >= 0`, then using
  range proofs to verify the equation.

The problem with this design is that, in MobileCoin, as with all RingCT-based cryptocurrencies, the amount of a TxOut actually appears in two places -- a Pedersen commitment, and a masked_value. The masked_value is a more conventional, symmetric-key encryption of the TxOut value, while the Pedersen commitment is an elliptic curve point which commits to the value, but cannot itself be directly "decrypted" efficiently by someone who doesn't already know the value.

All recipients determine the value of their TxOut's by the following process:
* Derive the TxOut shared secret
* Use the TxOut shared secret to derive the value mask, token id mask, and the amount commitment blinding factor. This computation never fails with an error, but produces "conjectures" which are expected to be correct if the Recipient owns this TxOut.
* Unmask the masked value and token id, obtaining a u64 value and a token id. These may or may not be the correct value of the Pedersen commitment.
* Reconstruct the Pedersen commitment using these conjectured values and the amount commitment blinding factor. If this matches the commitment, then value recovery was successful.

Thus value recovery relies on the masked_value and the amount commitment actually encoding the same numbers. If this is not the case, then value recovery fails, even if the TxOut is truly owned by the recipient.

However, consensus does not validate in any way that the masked_value actually matches the amount commitment. It only validates the correctness of the commitments.

Historically, the rationale for this is as follows:
* In a simple transaction, if the sender intentionally sends an output to a recipient that has a bad masked_value, this is equivalent to sending money to into a hole. The recipient will not be able to recover the value, so they will not render services to the sender.
* In [MCIP #31](https://github.com/mobilecoinfoundation/mcips/blob/main/text/0031-transactions-with-contingent-inputs.md) atomic swaps without partial fills, the originator is setting the masked value for the required outputs which are going to them. If the counterparty changes the masked value at all, the transaction is invalid. So the originator can only poison their own masked value, which only throws their own money into a hole.

(This is also the reasoning for all other RingCT-based cryptocurrencies.)

However, once we start doing partial fills, now we have a situation where the counterparty has to create the masked value for the originator. If consensus does not in some way validate the masked value, then a griefing attack becomes possible:

* An originator makes a signed partial fill offer.
* A counterparty fills the offer, but poisons the masked values going to the originator.
* The counterparty walks away with their side of the exchange, and the originator technically received their fractional outputs, but view key matching fails to produce the correct value for these outputs, and so they cannot actually spend these outputs unless they brute-force the discrete logarithm over a u64 search space, which would be very expensive. (If you use baby-step giant-step algorithm here, then perhaps this can be done with ~4 billion elliptic curve operations. So, perhaps not impossible, but very inconvenient.)

It is in principle possible that we could validate the masked value using a zero-knowledge proving framework. For example, in principle it is possible that a ZK-SNARK can prove that a particular Pedersen commitment and a particular masked value both encrypt the same value, and this proof need not reveal what that value is.

However, this is an extremely complicated solution to this problem. Embedding both Ristretto scalar multiplication and blake2b hashing into a ZK-SNARK would mean that we are building zero-knowledge proofs that are similar in complexity to Zcash. Making this mobile-friendly becomes a challenging research problem. So this is for now a dead-end and something we could only hope to do later as part of an upgrade to all of the transaction math.

Another approach might involve migrating the masked value from a simple symmetric-key encryption to something with homomorphic properties, so that we can verify it similarly to how we verify the partial fill equations for the Pedersen commitments. However, this ultimately doesn't seem very promising, because even if we could homomorphically verify inequalities over the masked values, that isn't enough, because the masked value for each output has to actually match exactly to the corresponding Pedersen commitment, and two inequalities being satisfied doesn't imply this.

Since revealing the amount secret in the enclave is a much simpler way to resolve this, which doesn't negatively impact the threat model for users of a decentralized exchange based on this feature, this seems like the best way to move forward. We can revisit zero-knowledge proofs for partial fills at a later date when more infrastructure for this exists in our ecosystem.

# Prior art
[prior-art]: #prior-art

Much of the prior art from MCIP #31 also informs this MCIP.
Section 6.2 of the [Zexe paper](https://eprint.iacr.org/2018/962.pdf) is also worth reading.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

None at this time.

# Future possibilities
[future-possibilities]: #future-possibilities

In the future, if/when we replace SGX transaction validation entirely with zero-knowledge proofs, we should use that proving framework to also
support zero-knowledge partial fill validation.
