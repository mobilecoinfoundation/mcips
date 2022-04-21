- Feature Name: `minting`
- Start Date: 2022-04-21
- MCIP PR: [mobilecoinfoundation/mcips#0037](https://github.com/mobilecoinfoundation/mcips/pull/37)
- Tracking Issue: N/A

# Summary
[summary]: #summary

A normal MobileCoin transfer transaction contains a set of inputs, which are marked spent, to create (mint) a new set of outputs. The network validators confirm that the sum of the value of the inputs is equal to the sum of the value of the outputs, using Ring Confidential Transactions. 

To support multiple assets, some of which represent backed assets, we will use an auditable process to introduce new token types to the MobileCoin blockchain, following a minting and burning procedure that corresponds with the status of the backed assets. 

The birds-eye view is that after a set of _Governors_ has been agreed upon by the MobileCoin consensus validator node operators, a new _Minting Configuration_  is submitted to specify the signers who are allowed to mint the new asset type on the MobileCoin blockchain. An amount of the backing asset will be verifiably locked, for example into a smart contract on another chain, and then a _Minting Transaction_ for the locked amount establishes a new set of transaction outputs for which there were no inputs marked spent on the MobileCoin blockchain, using the _confidential token ID_ (see [MCIP #25](https://github.com/mobilecoinfoundation/mcips/pull/25)) for the corresponding backing asset. When the backing funds are desired to be released from the lock, the amount to be released will be _verifiably burned_ (see [MCIP #35](https://github.com/mobilecoinfoundation/mcips/pull/35)) on the MobileCoin blockchain.

To achieve the set of functionality required to support backed assets on the MobileCoin blockchain, we will add 2 new transaction types:

* *Multiparty Minting Transaction*, `MintTx`, producing a TxOut which is indistinguishable from other TxOuts on the chain, along with a new type of block contents that include information specific to the token mint that occurred
* *Authorized Signer Declaration*, `MintConfigTx`, (also known as ``Set Minting Configuration Transaction''), producing a new ledger entry: *Minting Configuration*

# Motivation
[motivation]: #motivation

We would like to support multiple asset types on our chain, all of which need to undergo some minting process to be initialized on the chain. This process outlines a way to enable minting in an auditable way, to support multiple asset types, including stablecoins, which is a primary focus of this design.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## Multiparty Minting Transaction (`MintTx`)

For the Stablecoin minting procedures, each minting operation must coincide with an amount of the backing asset being ``locked", for example, into a multiparty signature (multisig) contract on another blockchain. 

*Definition: Multisig Contract.* A smart contract requiring a minimum number of parties to approve a transaction before committing it. The threshold is often referred to as _M-of-N_.

*Definition: Bridge.* A manual or automated inter-blockchain communication pathway that registers actions on either blockchain and commits actions in response to the corresponding blockchain. For example, an amount of asset is locked in a smart contract on one chain, triggering a mint on the other chain.

Once the transaction has been approved for minting by the Bridge, the Bridge Operator will construct a minting transaction for the MobileCoin blockchain, consisting of one or more transaction outputs to the destination address, along with a set of signatures from the approved signers list.

Each of _N_ signers maintains an Ed25519 keypair:

`(A_i, a_i)`

and provides their public key to be published alongside the contract on the backing asset locking chain, for example in a [Gnosis Safe](https://gnosis-safe.io/) contract. For space consideration savings, due to high gas fees for permanent storage, a hash of all public keys 

`H(A_i || A_{i+1} || ... || A_N)`

may be published on the contract, and the full set of signer public keys and the hash algorithm can be published to the MobileCoin blockchain (see [Signers](#signers)), or to another decentralized record service, such as [IPFS](https://ipfs.io/).

The contents of the `MintTx` include the full set of signatures of each of _M_ signers over the transaction's `TxPrefix`. 

Each Consensus Validator node is configured with the approved signers list, and this list is also previously published to the chain via the `MintConfigTx`. When a Consensus Validator node sees a `MintTx` with an _M-of-N_ set of signatures, it considers the transaction valid, and consensus proceeds as normal. When a quorum is reached, and the nodes externalize the transaction, the minted TxOuts are written, along with a new block contents section, `mint_txs`, which contains:


* The Token ID (only some Token Types allow minting, which is verified by the enclave)
* The amount being minted
* The destination's public subaddress view key
* The destination's public subaddress spend key
* the Ed25519 multisig signature set, including the total _M_ signatures required for the _M-of-N_ threshold, specified by the `MintConfigTx` previously accepted and written to the blockchain
* Nonce: random value to protect against replay attacks
* Tombstone block: block index after which this transaction will be discarded by consensus, to prevent the transaction from being nominated indefinitely


When submitting a `MintTx`, we include a random nonce to protect against replay attacks, and a tombstone block to prevent the transaction from being nominated indefinitely, and these are committed to the chain.

The block, and the latest `MintConfigTx`, ensure that the blockchain is therefore a self-contained document that can be audited to confirm that the minting was sound. In addition, minting is immune to replay attacks, meaning the same transaction cannot be submitted multiple times.

### Privacy Properties of Minted Transaction Outputs

Once the TxOuts are minted, they are exactly the same as any other TxOuts on chain, with the exception that the recipient view and spend public keys were published in the block's `MintTx`. The privacy guarantees for these TxOuts are that even with the view and spend published, one cannot verify with the TxOut alone that it was indeed sent to the specified public address. Rather, the public address is used by the enclaves to mint the amount requested to that address, and its publication in the block indicates that some number of the TxOuts in the block was sent to that address. The enclave does guarantee that at least one TxOut was sent to the specified public address. There is nothing stopping the block from containing other transactions for multiple token IDs, though in practice, it should be assumed that the TxOuts in a block with a `MintTx` should be considered "known" to have gone to that address, and all subsequent transactions from those TxOuts have full MobileCoin privacy guarantees.

## Declaring Authorized Signers (`MintConfigTx`)
[signers]: #signers

To prepare the consensus validator nodes to validate minting transactions, they must be aware of the set of _N_ authorized signers, and this set needs to be published to the blockchain for security, auditability, and recoverability.

The _Minting Configuration_, `MintConfigTx`, specifies the required information to declare the set of authorized signers:

* Token ID: the token type to which this configuration applies
* Signer set: the set of _N_ signers, `{A_i, A_{i+1}, ..., A_N}`
* Threshold: the number, _M_, of signers required per `MintTx`
* Mint limit: the total amount that can be minted in this configuration
* Nonce: random value to protect against replay attacks
* Tombstone block: block index after which this transaction will be discarded by consensus, to prevent the transaction from being nominated indefinitely

All of the values above are committed to the blockchain and will be used when verifying `MintTxs`.

Consensus validator nodes perform consensus when a `MintConfigTx` is proposed via an admin endpoint on the nodes. This provides flexibility to the node operators and keeps the network resilient so that a quorum is reached to allow a new minting configuration, while ensuring liveness of the network, so that the network need not restart in order to change the minting configuration. 

Consensus validator nodes use the latest minting configuration to validate `MintTxs`, and a node in catchup would use the latest `MintTx` it has seen. 

* Note, however, that nodes in catchup mode do not participate in consensus. The network is already resilient to nodes in catchup, and the minting procedures are similarly resilient.

## Governors: The Signer Set for a Consensus Validator
[governors]: #governors

The consensus validator nodes must maintain a set of "master minters," also known as _governors_, who can sign the `MintConfigTx` transactions. The governor set is maintained in the node's startup configuration file, and it is the responsibility of the node operator to perform due diligence on any governor they are adding to the set.

The governor set is a set of Ed25519 public keys, and a `MintConfigTx`'s `TxPrefix` must be signed by the threshold number of governors required by each node in a quorum of nodes to establish a new Minting Configuration for the network.

### Multiple Asset Fees

The startup-config for the consensus validator node contains the governors, as well as a `TokensConfig`, which specifies the following per token:

* Token ID
* Minimum fee
* Whether any fee is allowed 
* The list of governors who can change the Minting Config for this token

Every transaction's fee is paid in the same token type as the inputs and outputs of the transaction.

Prioritization on the consensus validator nodes is determined from the fee of the transaction, multiplied by a priority number derived from the minimum fee for the token asset (see the [Tx fee priority commit](https://github.com/mobilecoinfoundation/mobilecoin/commit/9450b1966cef23cedd53cac5f169110019fe2f2d). Therefore, prioritization can occur without revealing the token types involved in the transactions in a block.

A client can adjust its fee during times of congestion by knowing both the minimum fee for the token type they are attempting to transact, along with the rejected fee value they submitted.

Currently, the enclaves aggregate the fee for multiple transactions in the same block so that there is only one fee output in order to save space in the ledger. With multiple fees, the aggregation will produce a number of fee outputs capped by the minimum of either the number of token IDs in the block or the number of transactions in a block.


# Drawbacks
[drawbacks]: #drawbacks

Adding multiple confidential asset types with minting, burning, and bridge operation increases the complexity of the system.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

## Set of Signers vs Threshold Signatures

An alternate proposal would be to use a Threshold Signature, such as [Flexible Round-Optimized Schnorr Threshold Signatures (FROST)](https://datatracker.ietf.org/doc/pdf/draft-irtf-cfrg-frost-01.pdf). The reason we do not intend to use Threshold Signatures at this time is that we consider the explicit list of signers to be a desirable feature for auditability. In a threshold signature, the signer public key is a composite of all the participants, and it is not known exactly which _M_ of the _N_ participants participated to create the signature.

## Smart Contracts for Multiple Asset Support

Many other blockchain ecosystems support smart contracts to enable the minting of new assets, the most well-known probably being the [ERC-20](https://ethereum.org/en/developers/docs/standards/tokens/erc-20/) token standard on Ethereum. A fully-featured smart contracting system is desirable in our ecosystem and on our roadmap, but we can achieve the functionality of supporting multiple asset types without the overhead of full smart contract support.

# Prior art
[prior-art]: #prior-art

> Discuss prior art, both the good and the bad, in relation to this proposal.
> A few examples of what this can include are:
>
> - For consensus and fog proposals: Does this feature exist in other systems, and what experience have their community had?
> - For community proposals: Is this done by some other community and what were their experiences with it?
> - For other teams: What lessons can we learn from what other communities have done here?
> - Papers: Are there any published papers or great posts that discuss this? If you have some relevant papers to refer to, this can serve as a more detailed theoretical background.
>
> This section is intended to encourage you as an author to think about the lessons from other systems, provide readers of your MCIP with a fuller picture.
> If there is no prior art, that is fine - your ideas are interesting to us whether they are brand new or if it is an adaptation from other systems.
>
> Note that while precedent set by other systems is some motivation, it does not on its own motivate an MCIP.
> Please also take into consideration that MobileCoin sometimes intentionally diverges from common cryptocurrency features.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

> - What parts of the design do you expect to resolve through the MCIP process before this gets merged?
> - What parts of the design do you expect to resolve through the implementation of this feature before stabilization?
> - What related issues do you consider out of scope for this MCIP that could be addressed in the future independently of the solution that comes out of this MCIP?

# Future possibilities
[future-possibilities]: #future-possibilities

> Think about what the natural extension and evolution of your proposal would
> be and how it would affect the project as a whole in a holistic way. Try to
> use this section as a tool to more fully consider all possible interactions
> with aspects of the project in your proposal. Also consider how this all
> fits into the roadmap for the project and of the relevant team.
>
> This is also a good place to "dump ideas", if they are out of scope for the
> MCIP you are writing but otherwise related.
>
> If you have tried and cannot think of any future possibilities,
> you may simply state that you cannot think of anything.
>
> Note that having something written down in the future-possibilities section
> is not a reason to accept the current or a future MCIP; such notes should be
> in the section on motivation or rationale in this or subsequent MCIPs.
> The section merely provides additional information.

