* Feature Name: `block_metadata`
* Start Date: 2022-05-24
* MCIP PR: [mobilecoinfoundation/mcips#43](https://github.com/mobilecoinfoundation/mcips/pull/43)
* Tracking Issue: https://github.com/mobilecoinfoundation/mobilecoin/issues/2104

Contents:
- [Summary](#summary)
- [Motivation](#motivation)
- [Guide-level explanation](#guide-level-explanation)
- [Reference-level explanation](#reference-level-explanation)
  - [Protobuf schema](#protobuf-schema)
  - [Signing metadata](#signing-metadata)
  - [Signing key history](#signing-key-history)
    - [Key History TOML format](#key-history-toml-format)
    - [Key History JSON format](#key-history-json-format)
  - [Historical AVRs](#historical-avrs)
    - [AVR History TOML format](#avr-history-toml-format)
    - [AVR History JSON format](#avr-history-json-format)
  - [Block Version bump](#block-version-bump)
- [Drawbacks](#drawbacks)
- [Rationale and alternatives](#rationale-and-alternatives)
  - [Alternatives considered](#alternatives-considered)
- [Prior art](#prior-art)
- [Unresolved questions](#unresolved-questions)
- [Future possibilities](#future-possibilities)

# Summary
[summary]: #summary

Add metadata to the blockchain, with consensus nodes reporting the Attested
Verification Report and quorum set at the time a block was externalized.

# Motivation
[motivation]: #motivation

This is a long-awaited feature for the MobileCoin blockchain.

The AVR proves that the block was signed by an enclave, as the pubkey used to
sign the block is in the AVR. Signing the metadata with the SCP message signing
key proves that the block and metadata were generated by the holder of that
private key.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

When consensus nodes externalize a block, they will include `BlockMetadata` with
the AVR and quorum set for that block.

LedgerDB and WatcherDB also track and persist the block metadata.

Note that while the `BlockMetadata` can be used to verify chain integrity, it
couldn't be used to falsify any new transactions or blocks.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

Add a `message BlockMetadata` to `blockchain.ArchiveBlock`, and an analogous
`struct BlockMetadata` to `mc_blockchain_types::BlockData`.

## Protobuf schema
```protobuf
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

    /// Message signing key.
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

    /// Block signature.
    BlockSignature signature = 3;

    /// Additional signed metadata about this block.
    BlockMetadata metadata = 4;
}
```

Since this is a protobuf-additive change, we don't need a new `ArchiveBlock`
version/type.

## Signing metadata
The metadata signature will always differ from the block signature. The block
signature is generated by the enclave using a private key that never leaves the
enclave, while the metadata is signed by the "untrusted" code outside the
enclave using the SCP message signing key, since the enclave does not have this
metadata.

The metadata signature is generated from the `Digestible` encoding of
`BlockMetadataContents`, since the protobuf wire format is not a canonical
encoding.

## Signing key history
MobileCoin Foundation will publish a mapping/list of node_id to a range of block
indexes, which node operators and consumers can use to verify the metadata
signatures.

This mapping will contain multiple node_id:block_range entries

### Key History TOML format
```toml
[[node]]
# Pub key, as hex-encoded or PEM-encoded data or a PEM file path.
message_signing_pub_key = "..."
# Index of the first block signed with this key.
first_block_index = X
# Optional index of the last block signed with this key.
last_block_index = Y
```

### Key History JSON format
```json
{
  "node": [
    {
      "message_signing_pub_key": "...",
      "first_block_index": 0,
      "last_block_index": 42
    }
  ]
}
```

## Historical AVRs
With `BlockMetadata`, consumers of the blockchain may not need a Watcher DB.
Ledger DB can default to reading AVRs from `BlockMetadata` (using the
aforementioned signing key history to authenticate the metadata signature),
using the AVR to verify the enclave key used for the block signature, and fall back
to a lookup table containing AVRs for older blocks.

MobileCoin Foundation will publish a bootstrap file defining this lookup table
with historical consensus enclave AVRs, up to when consensus nodes publish
`BlockMetadata` with their AVRs.

### AVR History TOML format
```toml
[[node]]
# Responder ID for the consensus node.
responder_id = ""
# Index of the first block signed with this enclave.
first_block_index = X
# Optional index of the last block signed with this enclave.
last_block_index = Y
[node.avr]
# Hex-encoded signature bytes
sig = "..."
# An array of hex-encoded pub key bytes
chain = ["x", "y"]
# IAS HTTP response body
http_body = '{...}'
```

### AVR History JSON format
```json
{
  "node": [
    {
      "responder_id": "",
      "first_block_index": 0,
      "last_block_index": 42,
      "avr": {
        "sig": "...",
        "chain": ["x", "y"],
        "http_body": "{...}"
      }
    }
  ]
}
```

## Block Version bump
Block version 3 (release v1.3) will make the `block_metadata` required, at which
point consensus nodes and clients should reject blocks without metadata.

# Drawbacks
[drawbacks]: #drawbacks

* This makes each block bigger (more bytes), but we gain an important ability to
  backfill a ledger from `ArchiveBlocks`.
* A introduces complexity around backfilling a ledger, namely looking up
  historical AVRs.
* Also entails additional configuration, with the associated operational cost.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

This addition of AVRs allows consumers to verify the blockchain with fewer data
sources, including post-hoc validation that the blocks were externalized by
legitimate consensus enclaves. This enables a single source of truth in the
steady state (rather than requiring a Watcher DB in addition to the actual block
data).

While this capability is necessary for block streaming, it is useful to decouple
the step of adding AVRs and quorum sets to the blockchain.

## Alternatives considered
#### Write metadata for older blocks
It would be nice if we can generate metadata for older blocks, as that would
keep the lookup logic simple, but we cannot generate signatures since MCF does
not have all the private signing keys, nor should it.

# Prior art
[prior-art]: #prior-art

The author has not found any direct analogs for including network metadata in a
blockchain.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

None as of this writing.

# Future possibilities
[future-possibilities]: #future-possibilities

1. Enables the streaming and archive APIs defined in
   [MCIP #29](0029-block-streaming.md) to use equivalent representations across
   the streaming and archive block APIs.
2. Enables block streaming and other consumers to include and validate the AVRs
   for persisted blocks, which opens a path to replacing mobilecoind and watcher
   in Fog.