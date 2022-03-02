* Feature Name: `block_streaming`
* Start Date: 2022-03-01
* MCIP PR: [mobilecoinfoundation/mcips #29](https://github.com/mobilecoinfoundation/mcips/pull/29)
* Tracking Issue: [mobilecoin #1433](https://github.com/mobilecoinfoundation/mobilecoin/issues/1433)

# Summary
[summary]: #summary

This MCIP formalizes the API for getting blocks from consensus nodes.

This includes defining an API for streaming blocks using a dedicated fanout
mechanism, rather than repeatedly polling S3 for new blocks.

# Motivation
[motivation]: #motivation

About 40% of the end-to-end transaction time is spent writing to and reading
from S3. This can be optimized using a dedicated publish/subscribe mechanism to
distribute updates across nodes.

<details>
<summary>Details on the 40% figure</summary>

As shown in the screenshot below (dated 2021-11-17), of the 5 seconds to
externalize a transaction, we spent almost one second uploading to S3, and
another second downloading from S3.

![trace screenshot](../images/29-trace.png)
</details>

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

This is a backend implementation change that should be transparent to everyone else.

When a node starts up, it will subscribe to updates from its peer consensus nodes, and request (or be configured with) the base URL for fetching archived blocks, and fetch those in parallel.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

There are two mechanisms for getting blocks from consensus nodes:
1. A streaming API for getting the latest blocks, and
2. A backfill/discovery API specifying where to find archived blocks.

Each consensus node's ledger is distributed via a local "side-car" process,
`ledger-distribution`, to an associated directory in the `mobilecoin.chain` S3
bucket.

The other nodes then download these blocks from S3, usually via `mobilecoind` or
`fog`, using the `LedgerSyncService` or directly invoking a
`TransactionsFetcher`.

This MCIP expands the distribution to also publish to a message bus, and expands
the `TransactionsFetcher` idiom to support receiving updates from this message
bus.

When a node needs to get blocks and has not received them via the message bus,
it will fall back to fetching the node from S3, matching existing behaviour.

## Public Entrypoint API

This is the entrypoint for subscribing to block updates, leveraging
[server streaming gRPC](https://grpc.io/docs/what-is-grpc/core-concepts/#server-streaming-rpc).

### Streaming API
The Streaming API will look like the following:
```proto3
message SubscribeRequest {
    /// Block index to start from.
    uint64 starting_height = 1;
}

message BlockWithQuorumSet {
    /// The block data.
    blockchain.ArchiveBlock block = 1;
    /// QuorumSet when the block was externalized.
    QuorumSet quorum_set = 2;
    /// Attestation Verification report.
    external.VerificationReport report = 3;
}

message SubscribeResponse {
    BlockWithQuorumSet result = 1;
    external.Ed25519Signature result_signature = 2;
}

service ConsensusUpdates {
    rpc Subscribe(SubscribeRequest) returns (stream SubscribeResponse);
}
```

### Backfill API
The Backfill API will look like the following:
```proto3
message ArchiveBlocksUrlRequest {
    // May include hints for geographic proximity and/or caching.
}

message ArchiveBlocksUrlResponse {
    string base_url = 1;
}

service ArchiveBlocks {
    rpc GetBaseUrl(ArchiveBlocksUrlRequest) returns (ArchiveBlocksUrlResponse);
}
```

### Paths for archived blocks

### Single Blocks
The block-specific path is a function of the block index. The file name is the
hexadecimal representation of the index, ~~padded~~ to 16 hexadecimal characters.
The index-as-file-name is repeated in the directory path.

For example:
* block with index `1` will be at
  `$BASE/00/00/00/00/00/00/00/0000000000000001.pb`,
* block with index `0x1a2b_3c4e_5a6b_7c8d` will be at
  `$BASE/1a/2b/3c/4e/5a/6b/7c/1a2b3c4e5a6b7c8d.pb`

See
[`block_num_to_s3block_path`](https://github.com/mobilecoinfoundation/mobilecoin/search?q=fn.block_num_to_s3block_path)
for an implementation of this logic.

### Merged Blocks
We also support merged blocks to reduce the number of file operations. The
merged block path is similar, with a prefix directory `merged-$N` where `N > 1`
is the bucket size, and the index is that of the starting block (also a multiple
of `N`).

`ledger-distribution` will default to merging into buckets of 100, 1000 and 10000 blocks.

For example:
* merged blocks with `N=10` and starting index `0` will be at
  `$BASE/merged-10/00/00/00/00/00/00/00/0000000000000000.pb`,
* merged blocks with `N=1000` and starting index `1_000_000_000_000_000` will be
  at `$BASE/merged-1000/00/03/8d/7e/a4/c6/80/00038d7ea4c68000.pb`

See
[`merged_block_num_to_s3block_path`](https://github.com/mobilecoinfoundation/mobilecoin/search?q=fn.merged_block_num_to_s3block_path)
for an implementation of this logic.

# Drawbacks
[drawbacks]: #drawbacks

The primary drawback is introducing a dependency on another subsystem, namely
the message bus, which may affect our uptime or/and throughput.

With that said, this is an additive change, since we will fall back to S3 if no
messages arrive, so the behaviour and performance in the worst-case scenario
matches the existing system.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

The main constraints for the fanout mechanism are performance, maintaining
decentralization (i.e. not introducing a centralized service), and interfacing
across nodes run by separate operators. Given that, we have opted for a custom API.

We shall start with one publisher per consensus node, for maximum flexibility.
These nodes can be merged/shared per operator cluster, without loss of
generality.

One alternative is to have the MobileCoin Foundation run a pub/sub service, but
that introduces a centralized service that can become a single point of failure.

The authors have found no other alternative that satisfies our constraints.

The impact of not doing this is what we currently have, which wastes precious
time doing S3 I/O, effectively using it as a message bus, for which it was
simply not designed

# Prior art
[prior-art]: #prior-art

None found.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

We will need to experiment with different pub/sub implementations, and may
switch implementations as needed. See [publish-subscribe-mechanisms] below.

# Future possibilities
[future-possibilities]: #future-possibilities

While this MCIP focuses on distributing blocks from consensus nodes, the same
concepts can be useful for Fog and other subsystems.

## Publish/Subscribe mechanisms
[publish-subscribe-mechanisms]: #publish-subscribe-mechanisms

This section outlines some possible implementations backing the public API.

### Kafka

[Apache Kafka](https://kafka.apache.org/) is a mature offering.

Quoting [Confluent](https://www.confluent.io/what-is-apache-kafka):
> Apache Kafka is a community distributed event streaming platform capable of
> handling trillions of events a day. Initially conceived as a messaging queue,
> Kafka is based on an abstraction of a distributed commit log. Since being
> created and open sourced by LinkedIn in 2011, Kafka has quickly evolved from
> messaging queue to a full-fledged event streaming platform.

Pros:
+ Kafka is an established message bus, with a fair bit of established experience hosting and operating instances, including many cloud-hosted options.
  * Cloud providers include
    [Confluent Cloud](https://www.confluent.io/confluent-cloud/),
    [Amazon Managed Streaming](https://aws.amazon.com/msk/),
    [Cloud Kafka](https://www.cloudkarafka.com/)
  * Self-hosting is also a fairly popular option.
  * [Managed Apache Kafka Vs. DIY: What’s The Difference And How To Choose?](https://keen.io/blog/managed-apache-kafka-vs-diy/)

Cons:
- Unclear how to safely expose Kafka interface across operators.
- Having one Kafka broker per consensus node could negate the benefits of using
  Kafka

### Google Cloud Pub/Sub

[Google Cloud also offers a Pub/Sub solution](https://cloud.google.com/pubsub/)

Pros:
* Horizontally scalable managed messaging offering.

Cons:
* Risks vendor lock-in.
