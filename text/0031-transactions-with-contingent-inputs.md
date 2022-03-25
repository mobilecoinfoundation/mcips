- Feature Name: Transactions with Contingent Inputs
- Start Date: Mar 21, 2022
- MCIP PR: [mobilecoinfoundation/mcips#0031](https://github.com/mobilecoinfoundation/mcips/pull/0031)
- Tracking Issue: None

# Summary
[summary]: #summary

After MCIP [#25](https://github.com/mobilecoinfoundation/mcips/pull/0025),
evolve the transaction format with two new features:

* Multi-asset transfers: A single Tx can now contain inputs and outputs of multiple different types. Each input and output must be annotated with its type,
to enable range proof validation. As long as the Tx is balanced across all types, it is valid. The transaction fee can be paid with any of the token types that are used.
* Contingent inputs: A "contingent input" is a transaction input which has been signed by the owner, for which there are additional rules which must be satisfied
  by a transaction which uses it, for that transaction to be valid. In particular, a contingent input can specify that a particular output must also appear in the
  transaction, in order for the transaction to be valid.

# Motivation
[motivation]: #motivation

Contingent inputs are interesting because they allow users to construct "transaction fragments" which can then be sent to other users safely.
For example, if I want to buy a token for 5 MOB, I could create an output which gives me that token, and sign a contingent input which spends 5 of my MOB
in exchange for someone producing that output. I can then bundle this up and send it to a counter party -- if they agree to trade at my price, they can build
a transaction using my transaction fragment and submit it.

There is no way that they can spend my 5 MOB without producing the desired output for me, so my funds
are safe, and moreover, never leave my custody during this process until the moment the trade is final.
It costs me nothing to produce a transaction fragment and does not require an on-chain event -- there is only one actual on-chain transaction
to settle the swap, which happens with the speed and privacy of MobileCoin.

The broader significance is the following:
* If MCIP #25 happens, then MobileCoin will become a multi-asset ecosystem. However, there will not be simple ways for the multiple tokens
  to interact on chain at that point.
* In cryptocurrency ecosystems, it is important to provide users with convenient ways to swap tokens of various types.

In the Ethereum ecosystem, the dominant way of doing this is using tools like Automated Market Makers (AMM's). But, this requires smart contract
functionality, which is very complex. It is also quite difficult to create AMM's with significant privacy properties.

Many research papers have been written on cross-chain atomic swaps, particularly, describing protocols using Bitcoin's scripting language
to permit swapping bitcoin in a "trustless" way for other assets, using hash time locked contracts. These protocols are often quite complex,
requiring each party to "commit" their funds first, then trigger the swap, and have contingencies for each party to recover their committed funds
if the counterparty renegs. These protocols may require several rounds of interaction, and require several on-chain transactions, incurring fees and
increasing the total delay for the swap to be finished. (Because of this complexity, alternatives have been sought such as "wrapped Bitcoin" which is an
ERC20 representation of locked bitcoin which can be used more easily on the Ethereum network.) Another comment here is that in these multistep protocols,
if the counterparty stops participating, you can still recover your funds, but you still pay a transaction fees to recover them from the escrow, and you
didn't manage to complete your swap. You can try again with a different counterparty, but if this happens repeatedly you will lose money to fees.
If there are a lot of bad actors in the network, then this could happen often, and it may be very difficult for the network to prevent this from happening
when the underlying mechanism is a multistep protocol.

This proposal is considerably simpler -- the rounds of interaction are a minimum, requiring only one side to send a transaction fragment to another.
No one is required to "escrow" their funds into a smart contract or temporary address. A party who desires to swap can broadcast their transaction fragment
to many potential counterparties. Because the Ring MLSAG contains the key image of the spent input, everyone is able to check if the transaction fragment is
still valid, and the initiating party can cancel the swap proposal by simply spending the input to themselves in a separate transaction, burning its key image
and preventing any of the broadcasted transaction fragments from being used in a valid transaction.

A feature like this could potentially be used to build a fast, clean asset-swap experience in a mobile app, without relying on smart contracting functionality.
Note that, much like with current app experiences, there is no "pending" state for a swap, it either happens atomically or it doesn't before the tombstone block,
and so the app becomes responsible for tracking pending swaps just as with pending transfers.

More generally, it may be interesting to explore ways that users can collaboratively sign transactions securely. This is one way that we can experiment with this,
and could potentially be extended in the future.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

In a MobileCoin transaction, an input consists of a collection of `TxOut`'s in the ledger, one of which is truly being spent and others of which are just being mixed in. This ring,
and the proofs of membership, appear in the `TxIn`, in the `TxPrefix`. A transaction produces several new `TxOut`'s, and these outputs appear in the `TxPrefix`.

The signature part of a transaction contains several things:
* A `RingMLSAG` for each `TxIn`
* A `pseudo_output_commitment` for each `TxIn`
* A range proof for each `pseudo_output_commitment`, relative to the generator for this token id.

Transaction validation ensures that the sum of outputs minus the sum of pseudo-outputs is the zero point on the elliptic curve, (including the implicit fee output)
which ensures that the transaction is valid.

This feature contains two discrete steps:
* Allowing mixed transactions. This requires that individual outputs and pseudo-outputs in a transaction can be labeled with token ids.
  (The MCIP #25 token id, added to the tx prefix, is now interpreted as the fee token id.)
  When validating range proofs, each individual output and pseudo-output has its range proof checked relative to its specified token id.
  No other changes are required to support mixed transactions.
* Allowing contingent inputs in a transaction. A contingent input is a `TxIn` together with an additional `ContingentInputRules` object.
  The `RingMLSAG` for a contingent input signs over (a hash of) the `ContingentInputRules`. When validating a Tx with contingent input rules,
  those rules must be satisfied by the transaction as a whole for the transaction to be valid.
  Examples of rules include: "a particular output must appear in the `TxPrefix`", "the tombstone block may not exceed" a certain value.
  Because the `RingMLSAG` signs over the `ContingentInputRules`, these rules cannot be tampered with once the `RingMLSAG` is signed.

A "signed contingent input" is a payload consisting of:
* A `TxIn`, with `ContingentInputRules`.
* A `pseudo_output_commitment` for this ring.
* A `RingMLSAG` signing this.
* A token id, value, and blinding factor for the `pseudo_output_commitment`.
* A token id, value, and blinding factor for any output in the `ContingentInputRules`.

Anyone who owns a `TxOut` can use it to create a `SignedContingentInput`. The signed contingent input can be thought of as an offer
to trade "this for that" -- you are willing to give away your `TxOut` if someone produces for you the `TxOut` that you specify.
Once you sign the contingent input, you can give it away, and anyone who has it can fulfill your request to trade.

Giving away the signed contingent input does not reveal your public address, nor which `TxOut` in the ledger you actually own.
It does reveal the token id and value of your pseudo-output, but this is unavoidable in any such scheme.

Anyone who receives a `SignedContingentInput` can simply add it to the transaction builder and incorporate both the input and the required
outputs into a transaction. As long as they produce a balanced transaction at the end, their transaction will be valid and they will be able
to perform the swap with their counterparty atomically in a normal transaction.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

To support mixed transactions, the `Tx` format is extended as follows.

The `SignatureRctBulletProofs` object has two new fields, indicating token ids of outputs and pseudo output commitments.

```
message SignatureRctBulletproofs {
    repeated RingMLSAG ring_signatures = 1;
    repeated CompressedRistretto pseudo_output_commitments = 2;
    bytes range_proofs = 3;
    repeated fixed32 pseudo_output_commitment_token_ids = 4;
    repeated fixed32 output_token_ids = 5;
}
```

The `pseudo_output_commitment_token_ids` list must be the same length as `pseudo_output_commitments`,
and `output_token_ids` must be the same length as `outputs` in the `TxPrefix`, or the transaction is invalid.
(Alternatively, for backwards compatibility we could allow that if these lists are empty then they are filled with zeros.)

In the `TxPrefix`, the `token_id` field is renamed to `fee_token_id`, and it only represents the token id of
the fee output now.

```
// A transaction that a client submits to consensus
message TxPrefix {
    // Transaction inputs.
    repeated TxIn inputs = 1;

    // Transaction outputs.
    repeated TxOut outputs = 2;

    // Fee paid to the foundation for this transaction
    uint64 fee = 3;

    // The block index at which this transaction is no longer valid.
    uint64 tombstone_block = 4;

    // Token id for this transaction
    fixed32 fee_token_id = 5;
}
```

When checking range-proofs of outputs and pseudo-outputs, we must use the generator indicated by `pseudo_output_commitment_token_ids`
and `output_token_ids`, rather than the `token_id` in the `TxPrefix`.

To support contingent inputs, we introduce the following objects:

A `ContingentInputRules` object is a protobuf with the following schema:

```
message ContingentInputRules {
    // Outputs that must appear in the transaction for a transaction using this input to be valid
    repeated external.TxOut required_outputs = 1;
    // A maximum tombstone block value for the transaction. If zero, it means no maximum is enforced.
    fixed64 max_tombstone_block = 2;
}
```

It may be hashed in a standard way using the `mc-crypto-digestible` hashing scheme (like everything else).

This `required_outputs` are there to allow the main use-case of agreeing to a swap.
The `max_tombstone_block` is there both, to allow that some of the required outputs can be fog outputs (since fog imposes tombstone block limits),
and to allow that the party who builds and signs a `SignedContingentInput` can establish a "time to live" on the swap request.

The `TxIn` object is modified, adding an Optional `ContingentInputRules` member.

```
message TxIn {
    // "Ring" of inputs, one of which is actually being spent.
    repeated TxOut ring = 1;

    // Proof that each TxOut in `ring` is in the ledger.
    repeated TxOutMembershipProof proofs = 2;
    
    // Any rules which must be satisfied by the Tx that contains this TxIn
    ContingentInputRules contingent_input_rules = 3;
}
```

Since the `TxIn` objects are under the `TxPrefix`, which gets hashed to form the `message` for the overall ringct
signature, at this point the `ContingentInputRules` are "under the signature" for every MLSAG.

https://github.com/mobilecoinfoundation/mobilecoin/blob/694fd2de00443338e6dfc63359c6178840cf7e52/transaction/core/src/validation/validate.rs#L310

https://github.com/mobilecoinfoundation/mobilecoin/blob/694fd2de00443338e6dfc63359c6178840cf7e52/transaction/core/src/ring_signature/rct_bulletproofs.rs#L370

However, this is not compatible with our desired use-case, where one user, who doesn't know the `TxPrefix` signs one of the MLSAGs,
and another user fills in the rest of the transaction and signs the other MLSAGs.

Therefore we propose that when signing `RingMLSAG`'s, there are two different cases, one for the situation where there are `ContingentInputRules`,
and the other for the case where there are not `ContingentInputRules`.
* When there are no `ContingentInputRules` on the input, the message to the MLSAG is the `extended_message_digest` the same as before: https://github.com/mobilecoinfoundation/mobilecoin/blob/694fd2de00443338e6dfc63359c6178840cf7e52/transaction/core/src/ring_signature/rct_bulletproofs.rs#L209
* When there are contingent input rules present on the input, the message signed by the the MLSAG just consists of the digest of the `TxIn` but always omitting the `TxOutMembershipProof`, and including any `ContingentInputRules`.

(The `TxOutMembershipProof` should not be signed because consensus may enforce that membership proofs have similar root membership elements. There is no problem to allow the final submitter of the
transaction to update the root membership elements of these proofs.)

This creates the possibility that the extended message digest is never signed -- if every `TxIn` has contingent input rules, then none of them will actually sign over the `extended_message_digest`,
and if there are additional outputs in the `Tx` besides those appearing in the rules, no one will have signed over them, and no one will have signed over the fee. This would cause these parts of the
transaction to become malleable -- if the process of proposing the Tx to consensus, a man in the middle were able to intercept it, they could tamper with these parts of the transaction, in a way that
isn't possible for transactions currently.

Since transactions are submitted over attested channels to the consensus enclave, there is already quite a lot of protection against tampering with in-flight transactions.
However, another simple way to mitigate this is that a client can ensure that at least one input to the `Tx` does not have contingent input rules, and so signs the whole `TxPrefix`.

Another possibility we considered is that inputs and outputs in a `TxPrefix` can be ordered together, and the MLSAG for an input always signs over everything that appears before it, but not over what appears later.
However, this has a drawback, which is that if two people separately generate `SignedContingentInput`'s, then both inputs cannot be used in the same transaction.
This is also a breaking change, while the actual proposal above allows for backwards compatibility.

Finally, we describe the `SignedContingentInput` object.

The idea here is that:
* A signed contingent input contains enough data that `TransactionBuilder::add_signed_contingent_input(...)` can be implemented and building the transaction can succeed.
* No information besides that is transmitted. In particular, the one-time private key for the signed input is not needed, the true ring input not need be known, and the public address
  to which the output is destined need not be known either.

However, in order for the recipient to prove that the transaction is balanced, they must be able to know the amounts and blinding factors for the outputs and pseudo outputs.

An unmasked amount represents all of the unmasked data underlying a commitment to an amount, and allows to reconstruct an amount in a `TxOut` faithfully and spend it in a transaction.
It is also enough to build a range proof.

```
message UnmaskedAmount {
    // The value of this amount, in the smallest representable unit
    fixed64 value = 1;
    // The token id of this amount
    fixed32 token_id = 2;
    // The blinding factor used in the Pedersen commitment for this amount
    CompressedRistretto blinding_factor = 3;
}
```

The signed contingent input represents essentially a request to trade this for that, with sufficient information to execute
the trade, but no unnecessary additional information.

```
message SignedContingentInput {
    // The input which is signed. Note that the membership proofs may be omitted, and built by the final Tx submitter instead.
    TxIn input = 1;
    // A signature over the ring inputs. Note that this includes the `KeyImage` of the true output,
    // and can be used to determine if this SignedContingentInput is still spendable.
    RingMLSAG ring_signature = 2;
    // The unmasked information underlying the pseudo output from this ring
    UmaskedAmount unmasked_pseudo_output = 3;
    // The unmasked information underlying any required output from the ContingentInputRules.
    repeated UnmaskedAmount unmasked_required_outputs = 4;
}
```

A `SignedContingentInput` can be validated by:
* Checking that each `unmasked_required_output` corresponds to the commitment in the `required_outputs` field of `ContingentInputRules` in the `TxIn`. (and that there are the same number of each)
* Building the `pseudo_output` from the `unmasked_pseudo_output`, and confirming that the `ring_signature` actually signs it, when the message is the hash of `input`.

A `SignedContingentInput` is still spendable if the key image appearing in `ring_signature` has not appeared in the ledger yet.

The `TransactionBuilder` can now accept `SignedContingentInput`'s, and stores them in a list. It can check them for validity when they are provided.
The user of the `TransactionBuilder` may be expected to provide up-to-date / consistent merkle proofs for the inputs.

In `mc-transaction-core`, the sign function for `SignatureRctBulletproofs` can be modified so that it takes a list of `SignedContingentInput`'s.
These MLSAGs, inputs, pseudo-outputs and `required_output`'s are incorporated directly into the transaction, and the `unmasked_amount` are used to build range proofs, and then are discarded.

# Drawbacks
[drawbacks]: #drawbacks

The main drawback is that it increases the complexity of transaction validation.
If a simpler or better way of doing swaps is discovered and this is used only rarely, we will still have to carry the maintanence burden forwards.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

## Why is this design the best in the space of possible designs?

We can measure the complexity of a swap protocol by considering the number of rounds of interactions
between the participants and the number of on-chain events. In this proposal, these are both 1,
which is optimal.

Because there are fewer independent steps, there are fewer failure modes for the swap -- places where
one party can conduct part of the protocol and then change their mind, which then require failsafe mechanisms
to help the other party recover their escrowed funds. For comparison see "Bitcoin Monero Cross-chain Atomic Swaps"
and "Lightning Network Overview".

Privacy is also a key consideration. In this proposal, the first party sends only a `SignedContingentInput`,
so the second party does not learn their public address, only what they propose to trade and for how much.
The first party does not learn anything about the second participant except that they executed the trade.
The second party is unable to send subsequent transactions to the first party, since all they got was an output
created by the first party, and the TxOut `public_key` is enforced to be unique across the ledger.

In a realistic situation, it seems likely that the first and second parties know who each other are, since
they must initiate the interaction somehow. However, that is not required by the protocol proposed here.

Additionally, because the first party does not send a message to the blockchain to initiate the trade,
and only sends a `SignedContingentInput` instead, there is no permanent record that the trade was proposed,
nor is there a fee for proposing the trade. Additionally there is no possibility of "Miner Extractable Value"
(wherein the consensus nodes re-order incoming transactions for their own profit) because the consensus nodes
never see the trade proposal. The second party sees the token types and values being traded, but when they
add the `SignedContingentInput` to the transaction builder, those details do not appear in the final transaction
and are deleted.

Finally, this design is good because compared to smart contracts, it is a relatively small, incremental change to
implement given where we are today, and would meet the goal of enabling secure atomic swaps on-chain.

## What other designs have been considered and what is the rationale for not choosing them?

The main alternative for on-chain swaps is to use a smart-contract based system.
Indeed, for most forms of decentralized finance, the first step is escrowing your funds into a smart contract that you trust.

This has some immediate consequences:
* The user is exposed to smart contract risk, the risk that there is a bug in the code that will allow an attacker to seize the escrowed funds irreversibly.
* The user has to pay a fee and wait for a transaction to clear, just to support the escrow operation.
* There is a permanent record of the escrow operation.
* The smart contract has to know enough about you to send the funds back to you.

It also requires waiting for the design and implementation of a smart contracting system which has adequate privacy safeguards.

One criticism of the AMM / liquidity pool approach is that, smart-contract based AMMs generally impose fees which are comparable
to those imposed by centralized exchanges, which are paid to the liquidity providers. The contracts are thus in some sense operated
by and for the liquidity providers. Because anyone can create a smart contract and a liquidity pool, there is competition that drives
fees down, but pools with less liquidity are less efficient and result in higher price impact for large trades. This means that the
biggest pools tend to win out, even if they have larger fees. Additionally, when capital gets fragmented across many liquidity pools,
it cannot be used as efficiently. Many protocols have worked on either finding ways to fuse their pools and make more efficient use of
capital (see Aave v3 for instance). Additionally, there has been a rise of "Defi Aggregators" like 1inch that search across numerous
Defi systems for the best possible trade. A marketplace of limit orders (represented by `SignedContingentInput`'s), where many
liquidity providers can participate, much like AirSwap, could perhaps be simpler and more efficient.
(However, this is only one possible architecture for a swapping service based on `SignedContingentInput`'s.)

In the Ethereum blockchain, it is possible to submit a bundle of multiple transactions that are guaranteed to all execute atomically.
This mechanism could conceiveably be used to execute a swap, although to our knowledge it is never done this way because there is not
a trustless mechanism to build such transactions. (However, see also AirSwap).

MobileCoin could conceivably implement a linked transaction concept like this, but it would be complicated, and the main use for it is actually smart contracts,
which we don't have right now. For peer-to-peer swaps, we think that such a design would lead to significantly more complexity than `SignedContingentInputs`.

## What is the impact of not doing this?

The impact of not doing this will likely be that, if MobileCoin creates additional tokens per MCIP #25,
users will only be able to trade them on centralized exchanges, since there is no smart contract
mechanism in MobileCoin yet and such a mechanism cannot be expected to happen for quite a while, due to
the size and complexity of the design and implementation task. To our knowledge there are not any other
existing proposals for how users can perform on-chain swaps securely.

Another benefit of doing this is that, it provides another answer to the concerns that motivate allowing
users to pay transaction fees with tokens other than MOB. In blockchains such as Ethereum, Ether is the only
token that can be used to pay gas fees, so even if you have alot of other valuable tokens, if you don't have
any Ether you are stuck. You may be able to buy some from centralized exchange using fiat currency to get unstuck,
or borrow from a friend. In layer two networks, they may not be directly listed on any exchange, and the simplest
thing to do may be to get the gas token from a faucet somewhere. This is a negative user experience, and one way
to resolve it is to allow paying fees in other tokens (as in MCIP #25).

This mechanism provides another way for the user to get out of this state, since creating a `SignedContingentInput`
does not require an on-chain event and so does not require gas. They could e.g. create a `SignedContingentInput` that
spends one of their `TxOut`'s, pays them back some MOB to pay for gas fees, and also change from their original input.
Then they could send this to a counterparty, who would pay the gas fees when they fulfill the order. In effect they can
swap even if they don't have any gas balance, unlike in Ethereum network.
This could justify simplifying MCIP #25 by only allowing MOB as gas.

# Prior art
[prior-art]: #prior-art

- [Uniswap](https://docs.uniswap.org/protocol/introduction): An introduction to the Uniswap protocol
- [What is an automated marker maker?](https://www.coindesk.com/learn/2021/08/20/what-is-an-automated-market-maker/): Coindesk article describing AMMs
- [Bitcoin Monero Cross-chain Atomic Swaps](https://eprint.iacr.org/2020/1126): A proposal using hashed time-locked contracts and bitcoin scripting language
- [AirSwap](https://about.airswap.io/): A decentralized peer-to-peer swap network
- [MEV](https://ethereum.org/en/developers/docs/mev/): A definition and discussion of MEV, particularly in connection to Ethereum
- [Aave](https://docs.aave.com/portal/): A decentralized lending protocol
- [Aave v3 Whitepaper](https://github.com/aave/aave-v3-core/blob/master/techpaper/Aave_V3_Technical_Paper.pdf): A description of the changes they made in Aave V3, including to try to improve capital efficiency
- [1inch](https://1inch.io/): A defi aggregator which searches hundreds of DEX's
- [dxdy](https://dydx.exchange/): A decentralized exchange using STARKware technology to perform roll-ups onto the Ethereum main chain
- [Aleo records](https://developer.aleo.org/aleo/concepts/records/): Mentioned for reference to the "birth program" and "death program" concepts.
- ["How the Lightning Network Actually Works"](https://www.youtube.com/watch?v=yKdK-7AtAMQ): A nice youtube video describing all the steps of creating a lightning network channel and settling payments.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

For Automated Market Makers like Uniswap, the end user experience is going to a website, and (typically) using a browser wallet to build and submit a transaction to the smart contract.

For swaps based on contingent inputs, one possible experience is that the service generates a signed contingent input for you, and you build a transaction with it and submit it to the network.
However, it could also be that the user builds a signed contingent input and submits it to the service (like AirSwap), where it enters an order book and is filled by someone else.
Potentially, SGX could be leveraged somehow, so that the service can prove to the user that they won't be front-run when they submit a trade order.

We think there are numerous ways that this primitive could be used to build a swapping service and a myriad of tradeoffs, and we prefer to scope this MCIP just to specifying the signed contingent input primitive.

# Future possibilities
[future-possibilities]: #future-possibilities

The `SignedContingentInput` mechanism could be extended in the future to support other as yet unforseen
use-cases. For example, conceivably a `SignedContingentInput` could have a rule that only allows it to be spent
if a script or a smart contract agrees. This is inspired in part by Aleo's "death program" concept.
