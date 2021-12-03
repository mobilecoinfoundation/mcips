```
  MCIP: ?
  Layer: Peer Services
  Title: Consensing on Block Timestamps
  Authors: koe <ukoe@protonmail.com>
  Status: Draft (DO NOT USE)
  Type: Standards Track
  Created: ?
  Requires: [Versatile SCP], [statement types], [canonical block timestamps]
```

## Abstract

MCIP-[canonical block timestamps] added a timestamp field to MobileCoin block headers, but did not describe how to populate that field. In this proposal, nodes in the network use SCP to collaboratively decide each block's timestamp, with strong Byzantine resistance.



## Motivation

MobileCoin nodes use [SCP](https://www.stellar.org/papers/stellar-consensus-protocol) to consense on block contents. Each node independently creates a block out of a set of statements it obtains from executing SCP in conjunction with other nodes. If nodes don't create exactly the same block for each block index, then they will diverge and become unable to interoperate.

This means nodes must all add the same timestamp to a given block header. Since nodes already use SCP to consense on other block contents, it makes sense to also use SCP for block timestamps.



## MCIP-[statement types] statement type `0x02`

Define an MCIP-[statement types] statement type, which this MCIP revolves around.

- Let type `0x02` be 'Time Range'.
  - serialize as two 8-byte unsigned integers
  	- a range (inclusive) of Epoch/Unix timestamps: low to high
  	- **invariant**: low timestamp <= high timestamp
  - record the range in the least significant 16 bytes of the SCP statement type
  	- timestamps are little-endian
  	- low timestamp in least significant 8 bytes of the SCP statement type
- A vote to nominate a Time Range (or accept a nomination) is a vote to nominate all timestamps in that range (inclusive of the bounds).
- A Time Range is an MCIP-[Versatile SCP] 'statement clump'.



## SCP considerations

### Validity

A Time Range that appears in a slot as part of any node's message is invalid if its lower bound is not equal to the previous block's timestamp plus one second (i.e. plus epsilon).
	- If a node witnesses a Time Range from another node, whose upper bound is over 5 minutes away from the node's local time, the node should log it. Unreasonable Time Ranges may be created by Byzantine nodes.


### MCIP-[versatile scp]

#### Statement clumps

MCIP-[versatile scp] encourages SCP implementers to define three behaviors for each statement type.

- `statements_intersect_fn`: Given two Time Ranges, return their intersection.
	- For example, given ranges [1, 3] and [2, 4], return [2, 3].
- `statements_not_intersect_fn`: Given two Time Ranges, return the part of the second range that does not intersect with the first.
	- For example, given ranges [1, 3] and [2, 4], return [4, 4].
	- If the result is disjoint, return all resulting ranges. For example, given [3, 3] and [2, 4], return [2, 2] and [4, 4].
		- **Note**: Disjoint Time Ranges should only appear in practice if there is an upstream bug (assuming MCIP-[versatile scp] and this MCIP are correctly implemented).
- `simplify_statements_fn`: Given two Time Ranges, if they intersect then return their union.
	- For example, given ranges [1, 3] and [2, 4], return [1, 4].

#### Optional ballot creation

- A ballot may only be created if there is a confirmed nominated Time Range whose lower bound equals the previous block's timestamp plus one second.
	- By implication, a ballot is invalid if it does not contain exactly one valid Time Range statement.
- When creating a ballot, use the valid Time Range with the highest possible upper bound.
	- If an SCP implementation is well formed, then the `combine_fn` for converting confirmed nominated statements into a ballot should only encounter at most one Time Range per invocation. It should only encounter valid Time Ranges.

#### Nomination filtering

A node may only vote to nominate a Time Range statement if that statement was proposed by the local node. The node may vote for its own Time Range statement even if it isn't selected to be a federated leader.
	- **Best practice**: Nodes should only propose one Time Range statement per slot. A good time to do this is the first time the node casts a vote to nominate a statement. Ideally, each node's first Time Range nomination vote appears in the same SCP message as its first nomination vote for a Transaction Hash.



## Creating a block

When creating a block from a confirmed committed ballot, set the timestamp field equal to the upper bound of the ballot's Time Range statement.



## Rationale

#### Why does nomination filtering only allow votes to nominate a node's own timestamp proposals? Why are nodes encouraged to only propose one Time Range statement per slot?

These are optimizations to reduce the number of messages sent between nodes, and reduce how long it takes for slots to be completed. It is an open question how much real impact they have on network performance.

#### How are the timestamps in this proposal strongly Byzantine resistant?

1. Strongly Byzantine resistant here means that most timestamps added to blocks were created by honest nodes with realistic local clocks.
1. When an intact node (see the [SCP whitepaper](https://www.stellar.org/papers/stellar-consensus-protocol) for the meaning of 'intact') adds a timestamp to a ballot, they select the highest timestamp confirmed nominated. By implication, it is the lowest accepted timestamp in the quorum used to confirm it.
1. For a timestamp confirmed by an intact node to be erroneously 'high', there must be a full quorum accepting it. In other words, a full Byzantine quorum, contradicting the assumption our node is intact. Therefore intact nodes will never confirm erroneously high timestamps (similarly, they will never externalize ballots containing erroneously high timestamps).
1. There is a weaker guarantee for erroneously 'low' timestamps. Each quorum containing a Byzantine node is in a race condition with quorums containing only honest nodes (there is always at least one honest quorum, otherwise the SCP safety/liveness guarantees are broken). Imagine all Byzantine nodes accept timestamp nominations with the minimum timestamp range. This minimum range could eventually be confirmed nominated, because honest nodes also vote for timestamps in that range. If no higher timestamps are confirmed nominated, then the minimum range could be added to ballots and ultimately externalized. However, honest quorums will constantly be working to confirm higher (more honest) timestamp nominations, which can usurp the erroneous range at any time. It is an open question what the probability of an erroneously low timestamp appearing is, given different amounts of Byzantine nodes and different network topologies.
1. Honest nodes set timestamps early during a slot. This, in combination with the weak guarantee for erroneously 'low' timestamps and strong guarantee preventing 'high' timestamps, means canonical block timestamps will always lag behind real block times.

#### Why can't nodes prevent erroneously low timestamps by raising the lower bound of their timestamp ranges?

If nodes are not guaranteed to have a path to confirming a timestamp nomination (e.g. by always agreeing about the lower bound of timestamp ranges, so as soon as a quorum of timestamp votes appears a timestamp can be accepted), then since timestamps are required to create a ballot, it is possible a slot will be greatly delayed or even completely stuck if it just so happens no quorum intersects on any timestamp range.



## Backward compatibility

Blocks currently don't have timestamps.



## Reference implementation

Not completed yet.



## References

- [Stellar Consensus Protocol whitepaper](https://www.stellar.org/papers/stellar-consensus-protocol)
