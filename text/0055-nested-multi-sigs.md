- Feature Name: nested_multi_sig
- Start Date: 2022-11-02
- MCIP PR: [mobilecoinfoundation/mcips#0053](https://github.com/mobilecoinfoundation/mcips/pull/0053)
- Tracking Issue: https://github.com/mobilecoinfoundation/mobilecoin/issues/2693

# Summary
[summary]: #summary

We currently require multi-signatures for authorizing `MintConfigTx` and `MintTx` transactions. At the moment, each multi-signature is defined as requiring M-of-N signatures. The set of authorized signers (`SignerSet`) specifies the potential signers coupled with the threshold (the `M` in `M-of-N`). Each signer is identified by their Ed25519 pulic key.
We would like to evolve this such that each member of a signer set can either be an individual Ed25519 public key, or another signer set.

# Motivation
[motivation]: #motivation

Allowing each signer in a multi-sig `SignerSet` to either be an individual key or it's own M-of-N set allows us to better represent the real-world signature hierarchy that we would like to use for minting and minting configurations.

For example, when specifying the required signers for some arbitrary token, we might say that we want two entities to authorize a minting transaction:
1. The MobileCoin foundation
2. A liquidity provider that holds the backing asset for the token being issued

Right now, there is no good way to specify this, unless each organization simply has a single key. That is suboptimal, since in reality the organization might prefer to have a few keys and require it's own M-of-N such that there is no single person in the organization that can authorize a mint. That is a more secure scheme.

Following the change in this PR, it will become possible to specify such signer sets, where each signer can be either an individual key, or another signer set.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Multi-signatures, and signer set definitions are provided by the `mc-crypto-multisig` crate. This crate exposes an object called `SignerSet`, which is an array of public keys and a threshold that specifies the minimum number of signatures that is required for a multi-signature to be considered valid.

We propose to add a third field to the struct, `multi_signers`, that allows including other `SignerSet`s. Signature validation code will be changed to look for matches over both `individual_signers` and `multi_signers`, where a match over a multi-signer is considered as a single signature when counting against the threshold.

We will make minor changes to the `tokens.json` file format and the `mc-consensus-mint-client` tool to allow specifying nested signer sets.

Since we are only adding a field, this change is backwards compatible when it comes to the protobuf representation of `SignerSet`s. . However, `SignerSet`s make their way into blocks (they are included in `MintConfig`s and `ValidatedMintConfigTx`s and those go into `BlockContents`. This means we will need to gate this feature on a block version change, so that clients have time to upgrade before it gets enabled and used. This is similar to what we have done in previous changes that affect data going into the ledger.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

We propose to add a third field to the struct, `multi_signers`, that allows including other `SignerSet`s. Following the change, the struct will look like this:
```
pub struct SignerSet<P: Default + PublicKey + Message> {
    /// List of potential individual signers.
    #[prost(message, repeated, tag = "1")]
    #[digestible(name = "signers")]
    individual_signers: Vec<P>,

    /// List of potential sets of signers. This allows us to declare a nested
    /// multi-signature scheme. Ideally the two arrays in this struct would be
    /// combined into a single one that holds an enum of (IndividualSigner,
    /// MultiSignerSet), but since this feature was added after we went live
    /// and instances of SignerSet made their way into the ledger, it is not
    /// possible to change the array type without breaking backwards
    /// compatibility. This is also the reason the tag numbers in the struct
    /// are not sequential.
    #[prost(message, repeated, tag = "3")]
    multi_signers: Vec<SignerSet<P>>,

    /// Minimum number of signers required. The potential signers are the union
    /// of `individual_signers` and `multi_signers`.
    /// This implies that the upper limit (total number of possible signers) is
    /// `individual_signers.len() + multi_signers.len()`.
    #[prost(uint32, tag = "2")]
    threshold: u32,
}
```

The matching proto definitions inside `external.proto` will also need to be changed to include this new field.

Since we are gating the support for the nested signer sets on a block version bump, there are places where we will need to ensure a `SignerSet` does not have any `multi_signers`, unless we are running a block version that supports it:
1. The consensus enclave should ensure that no governors utilize the new `multi_signers` field, unless the block version supports it
1. The `MintConfigTx` validation code should ensure that no `MintConfig` contains a `SignerSet` with any `multi_signers`, unless the current block version we are validating for supports it.


# Drawbacks
[drawbacks]: #drawbacks

This has the standard drawbacks that come from a ledger change - it requires all clients to update their code to support it, once the new block version goes live. It touches security-critical code, and introducing a bug there could lead to unauthorized minting.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

If we were writing the `mc-crypto-multisig` crate from scratch, and adequately planning for the nesting requirement, we would've likely created an enum to represent the two possible signer types (individual public key, another signer set) and had a single `signers: Vec<SignerEnum>` inside `SignerSet`. That is a slightly cleaner design, and adding the `multi_signers` field to `SignerSet`s is potentially slightly more confusing.

The reason we chose to do that is because it is a much simpler change. If we were to switch the signers definition into a enum, this new `SignerSet` will no longer be protobuf-backwards-compatible. Since we already have `SignerSet`s in the ledger on both testnet and mainnet we will have to continue supporting the existing implementation. That essentially means we will need to have a `SignerSetV1` and a `SignerSetV2`, together with a couple of `oneof` fields. `oneof`s require supporting the case of no known variant provided, which means more error cases along the path of anything that needs to store a `SignerSet`.

We could skip doing this entirely, but that means we are putting off an improvement that allows more secure and redundant key management. It is our belief that the benefit of allowing signing entities to be split into their own M-of-N configurations outweights the cost of making this change.

# Prior art
[prior-art]: #prior-art

[0037-minting.md] is the MCIP that introduced the need for `mc-crypto-multisig`.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

None at the moment.

# Future possibilities
[future-possibilities]: #future-possibilities

None at the moment
