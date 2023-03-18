```
  MCIP: ?
  Layer: Peer Services
  Title: Reducing Nomination Votes
  Authors: koe <ukoe@protonmail.com>
  Status: Draft (DO NOT USE)
  Type: Standards Track
  Created: ?
```

## Abstract

Allows an SCP node to constrain how many statements they vote to nominate with a new `nomination_reduction_fn`. Defines a constant `MAX_NOMINATION_VOTES`, which limits how many statements a node may vote to nominate at one time.



## Motivation

Currently, SCP nodes may vote to nominate an unlimited number of statements. A malicious node could cause the network to crash or become unresponsive by voting to nominate a huge number of statements.

Given a large set of statements to nominate, a node cannot prune it down naively. Unless there is a coordination mechanism between nodes, it may be possible for two (or more) nodes to prune their statement sets and end up without any intersections. Basically, the SCP 'eventual liveness' guarantee would be violated.

Instead, if nodes define a common reduction method in `nomination_reduction_fn` (i.e. sorting transactions and only reduction the lowest ones), then, at steady state, reduced sets will not be disjoint.



## Reducing nomination votes

### Nomination reduction

**Current behavior**: During the nomination phase, nodes collect statements to vote to nominate by looking at locally proposed statements and by 'mirroring' statements that other nodes have voted to nominate (or accepted nominated).

**New behavior**: During the nomination phase, nodes collect statements to vote to nominate by looking at locally proposed statements and by 'mirroring' statements that other nodes have voted to nominate (or accepted nominated). After collecting those statements, the node tries to 'reduce' them to a smaller set.

#### Implementation

Nomination reduction can be added to an SCP implementation with an undefined virtual function called `nomination_reduction_fn()`. Concrete instantiations of the SCP protocol implementation must define that function. It takes a set of SCP statements (or 'values') as input, and outputs a set of SCP statements that the node should vote to nominate.

```
nomination_reduction_fn(Vec<StatementType> statements) -> Vec<StatementType>;
```

**Note**: If MCIP-[scp statement types] and MCIP-[versatile scp] are implemented, then some statements could be non-removable due to optional ballot creation rules. Those statements should never be removed. In general, only transaction-referencing statements should be removed.


### Reduction rule: `MAX_NOMINATION_VOTES`

Define two constants:

- `NOMINATION_VARIANCE_FACTOR = 20`
- `MAX_NOMINATION_VOTES = ((100 + NOMINATION_VARIANCE_FACTOR) * MAX_TRANSACTIONS_PER_BLOCK) / 100`

In `nomination_reduction_fn`, sort all transaction statements by:

- transaction fee (descending)
- tombstone block (ascending)
- transaction hash (ascending)

If the total number of statements exceeds `MAX_NOMINATION_VOTES`, then remove the lowest transaction statements until that condition is met.

**Note**: If MCIP-[statement types] is implemented, then the above semantics are important. The entire set of statements is limited to `MAX_NOMINATION_VOTES`, but only transaction-referencing statements should be removed.

When receiving an SCP message from another node, consider it invalid if they voted to nominate more than `MAX_NOMINATION_VOTES` statements.



## Rationale

#### What is the `NOMINATION_VARIANCE_FACTOR`?

The variance factor describes the expected variance across the network in nomination sets, as a percentage of the maximum transactions that can be stored in a block (as enforced by the SCP `combine_fn` in MobileCoin).

In other words, given a quorum where each node votes to nominate `MAX_NOMINATION_VOTES` statements, the intersection of all those nodes' nomination sets is expected to be no less than `MAX_TRANSACTIONS_PER_BLOCK`. This is to prevent variance between nodes from disallowing construction of completely full blocks.

A variance of 20% was selected arbitrarily. It can be easily updated/revised based on real-world observations.



## Backward compatibility

This proposal should be rolled out alongside a hard fork to ensure all/most nodes update concurrently.



## Reference implementation

Not completed yet.



## References

- [Stellar Consensus Protocol whitepaper](https://www.stellar.org/papers/stellar-consensus-protocol)
