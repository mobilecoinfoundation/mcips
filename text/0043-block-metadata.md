* Feature Name: block_metadata
* Start Date: 2022-05-24
* MCIP PR: [mobilecoinfoundation/mcips#43](https://github.com/mobilecoinfoundation/mcips/pull/43)
* Tracking Issue: https://github.com/mobilecoinfoundation/mobilecoin/issues/2104

- [Summary](#summary)
- [Motivation](#motivation)
- [Guide-level explanation](#guide-level-explanation)
- [Reference-level explanation](#reference-level-explanation)
  - [Protobuf schema](#protobuf-schema)
  - [Tracking history of metadata signing keys](#tracking-history-of-metadata-signing-keys)
  - [Backfilling AVRs from Watcher](#backfilling-avrs-from-watcher)
- [Drawbacks](#drawbacks)
- [Rationale and alternatives](#rationale-and-alternatives)
- [Prior art](#prior-art)
- [Unresolved questions](#unresolved-questions)
- [Future possibilities](#future-possibilities)




# Summary
[summary]: #summary

Add metadata to the blockchain, with consensus nodes reporting the Attested Verification Report and quorum set at the time a block was externalized.

# Motivation
[motivation]: #motivation

This is a long-awaited feature for the MobileCoin blockchain.

This also lets the streaming and archive APIs defined in [MCIP #29](0029-block-streaming.md) use equivalent representations.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

When consensus nodes externalize a block, they will include the AVR and quorum set for that block.

LedgerDB and WatcherDB also track and persist the block metadata.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

Add a `message BlockMetadata` to `blockchain.ArchiveBlock`, and an analogous `struct BlockMetadata` to `mc_blockchain_types::BlockData`.

## Protobuf schema

```proto3
message BlockMetadataContents {
    /// The Block ID.
    BlockID id = 1;

    /// Quorum set configuration at the time of externalization.
    QuorumSet quorum_set = 2;

    /// IAS report for the enclave which generated the signature.
    VerificationReport verification_report = 3;
}

message BlockMetadata {
    /// Metadata signed by the consensus node.
    BlockMetadataContents contents = 1;

    /// Message signing key (signer).
    Ed25519Public node_key = 2;

    /// Signature using `node_key` over the Digestible encoding of `contents`.
    Ed25519Signature signature = 3;
}

// NB: Only addition is field #4.
message ArchiveBlockV1 {
    /// Block.
    Block block = 1;

    /// Contents of the block.
    BlockContents block_contents = 2;

    /// Block signature, when available.
    BlockSignature signature = 3;

    /// Additional signed metadata about this block.
    BlockMetadata metadata = 4;
}
```

Since this is a protobuf-additive change, we don't need a new `ArchiveBlock` type. A future block version change will make the `quorum_set` and `verification_report` required, rejecting blocks without those fields.

The metadata signature will always differ from the block signature. The block is signed by the enclave using a private key that never leaves the enclave, while the metadata is signed by the "untrusted" code outside the enclave, in part because the enclave cannot generate this metadata.

Note that while the `BlockMetadata` can be used to verify chain integrity, it couldn't be used to falsify any new transactions or blocks.

## Tracking history of metadata signing keys
TODO

## Backfilling AVRs from Watcher
TODO

# Drawbacks
[drawbacks]: #drawbacks

This makes each block bigger (more bytes), but we gain an important ability to backfill a ledger from ArchiveBlocks.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

While this is necessary for block streaming, it is useful to decouple the step of adding AVRs and quorum sets to the blockchain.

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

None at the time of writing.

# Future possibilities
[future-possibilities]: #future-possibilities

1. Opens a path to removing WatcherDB from Fog servers.
1. Enables block streaming and other consumers to fetch and validate the AVRs for persisted blocks.
1. Replace mobilecoind in Fog servers with block streaming.
