- Feature Name: populating-block-timestamps
- Start Date: 2023-11-30
- MCIP PR: [mobilecoinfoundation/mcips#0067](https://github.com/mobilecoinfoundation/mcips/pull/67)
- Tracking Issue: [mobilecoinfoundation/mobilecoin#1617](https://github.com/mobilecoinfoundation/mobilecoin/issues/1617)

# Summary
[summary]: #summary

A way for consensus nodes to agree on the value that goes into the block
timestamp field.

# Motivation
[motivation]: #motivation

MobileCoin nodes use
[SCP](https://www.stellar.org/papers/stellar-consensus-protocol) to consense on
block contents. Each node independently creates a block out of a set of
statements it obtains from executing SCP in conjunction with other nodes. If
nodes don't create exactly the same block for each block index, then they will
diverge and become unable to interoperate.

This means nodes must all add the same timestamp to a given block.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

When a node nominates a value for other nodes to consense on it will provide a
timestamp with that value. This value and timestamp will be refered to as a
_value timestamp pair_. The timestamp will be the Unix time since epoch
(UTC), as determined by the node's system clock. The node will ensure that this
timestamp is greater than the timestamp of the node's last known block.

Nodes will validate the _value timestamp pair_ from other nodes. A node will reject the _value
timestamp pair_ if the timestamp is not newer than the node's last known block
or if the timestamp is too far into the future.

Due to how _value timestamp pairs_ progress in SCP, a timestamp is not checked
for being too far in the past. Having a check for a timestamp being too old can
cause issues when taking into account network latency and how long it takes the
nodes to reach consensus. This means if a network is idle for a long period of
time, it is possible for a node to provide an older time than the current time,
and have other nodes consense on this time.

## Creating a block

When creating a block the nodes will use the largest timestamp from all of the
_value timestamp pairs_. Using this largest timestamp ensures that the latest, or
most recent, timestamp is used in the block. This timestamp should be the
closest to the current time that the block is being created.

Like other blockchain timestamps, the block timestamp is an approximation of
when the block landed on the chain and is not guaranteed to be accurate.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

When values are passed from the byzantine ledger worker,
[propose\_pending\_values()](https://github.com/mobilecoinfoundation/mobilecoin/blob/f5907cf3374b1e7b57a173af72950f247b789864/consensus/service/src/byzantine_ledger/worker.rs#L433), 
to the SCP implementation they will be populated with the
[duration](https://doc.rust-lang.org/std/time/struct.Duration.html) since Unix
epoch in milliseconds.

In order to account for network time skew, at the time of this writing, we allow
timestamps to be up to 3 seconds in the future. This 3 second value is based on
the
node signed at time data from the first 2 million blocks of the blockchain.

## Local Timestamp Validation Failures

The SCP implementation will validate the _value timestamp pair_. If, due to
time skew, the timestamp is not valid, then the SCP implementation will reject
the _value timestamp pair_. Once the SCP implementation finishes processing its
current block, the byzantine ledger worker will see that the previously
submitted _value timestamp pair_ were not consumed and thus resubmit them for
the next iteration.

In general the only time that a timestamp should fail to validate locally is if
the last known block has a newer timestamp than the node's system timer. This
can happen due to network and system time skew. Given this failure condition the
node's system time should eventually catch up to the last known block's time and
thus the byzantine ledger worker resubmitting to SCP should eventually succeed.

## Peer Timestamp Validation Failures

When a node receives a _value timestamp pair_ from a peer. It will validate the
timestamp with its own system time and its last known block's timestamp. 

If the timestamp is not greater than the last known block's timestamp then one
of two things occured:

1. The peer that sent the _value timestamp pair_ is in error. This means all
   other nodes in a well behaved quorum should reject this 
   _value timestamp pair_ as well.
2. This node is in error. This means all other nodes in a well behaved quorum
   should accept the _value timestamp pair_ and this node will not participate
   in consensing on the block.

If the timestamp is too far in the future compared to this node's system time
then this node will reject the _value timestamp pair_. The other node will
continue to transmit the _value timestamp pair_ in subsequent messages. If the
timestamp is in the future due to network skew, then it's likely that one of
these subsequent messages will be received once this node's system time catches
up.

If the timestamp is so far in the future that this node is never able to
validate it before consensus happens then one of two things will happen:

1. the other nodes in a well behaved quorum also reject the timestamp, omitting
   the _value timestamp pair_ from the next block.
2. the other nodes in a well behaved quorum validate the timestamp consensing on
   a block including that _value timestmap pair_. This node will not participate in consensing on the block.

# Drawbacks
[drawbacks]: #drawbacks

The timestamps are not accurate in representing any specific event. They are an
approximation of when a block was generated and are subject to the inaccuracies
of the node system timers.

Not having a check for old timestamps being used on idle networks has the
potential for a node to mislead. The current MobileCoin network sends out
periodic test transactions so this window for misleading is smaller than
minutes.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

As mentioned above, the timestamps in other blockchains are also an
approximation. 

For instance bitcoin's timestamps are documented as being within
a one to two hour window.

For Ethereum the node creating the block sets the timestamp based on its time.
Then the validators validate the timestamp as part of validating the block. This
approach doesn't work for SCP since all the nodes need to create the same block
and their system times may differ.

# Prior art
[prior-art]: #prior-art

This implementation was more or less derived from the [Stellar Consensus
Protocol whitepaper](https://www.stellar.org/papers/stellar-consensus-protocol).

> To combine candidate values, Stellar takes the union of their transaction sets
> and the maximum of their timestamps. (Values with invalid timestamps will not
> receive enough nominations to become candidates.)

# Unresolved questions
[unresolved-questions]: #unresolved-questions

None at this time

# Future possibilities
[future-possibilities]: #future-possibilities

None at this time
