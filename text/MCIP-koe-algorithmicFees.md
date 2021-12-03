```
  MCIP: ?
  Layer: Consensus
  Title: Algorithmic Fees
  Authors: koe <ukoe@protonmail.com>
  Status: Draft (DO NOT USE)
  Type: Standards Track
  Created: ?
  Requires: MCIP-[versatile scp]
```

## Abstract





## Motivation





## spec

- needs research!

- design considerations
	- an honest user should always be able to succeed in adding a high-priority tx to the chain (easily met with fee-based sorting)
	- low fees at steady state
		- minimum fee to discourage nanotransactions (coffee is canonical 'expected tx magnitude')
	- attacker who wants to bloat the chain should incur high costs
		- no big bang
			- must allow for wildly varying normal volume
				- min fee should not become prohibitive
		- evaluated in terms of 'relative to non-attacker tx volume'
			- a slow bloat attack is 'weak', and not designed against (it is indistinguishable from non-attacker txs)
			- fast bloat attack should be expensive
			- accelerating bloat attack should be expensive (i.e. long term with rising relative volume)
	- when selecting tx fee, should understand risk of tx failing based on different fee levels/priorities
		- complicated by tombstone block
		- bounded fee change over period of blocks
	- how to handle natural change of purchasing power over time?
		1. manual changes to constants
		2. partly algorithmic; falling min fees as volume rises
	- difficult to control block sizes, because blocks are not periodic (~ 5s blocktime; ad hoc creation)
		- controlling block sizes may allow an attacker to impact avg completion time for honest users
		- how to allow higher fee tx to bid up block size?

- output fee and weight of tx

- min fee increases 'between' and 'within' slots
	- between: base min fee/weight, block size bounds
	- within: min fee/weight scales up as more tx are added to X
		- risk: no tx gets a quorum because by the time tx get sent between nodes, each node has locally increased their min fee too high
			- use nominate_reduction_fn to actively prune X after updating it with nominate_fn calls (these only assess tx based on min fee)
				- fee escalation rule: discard low-priority tx that violate fee escalation rule (i.e. add tx fees together starting with highest fee/weight ratio until next_priority_tx.fee/weight < f(weight total, min fee))
				- nodes will implicitly coordinate consensus-success by constructing X with hightest-fee tx, passing these between each other, voting for high-fee tx, etc.; no matter what realistic differences in min fee exist between nodes (e.g. < 1k MOB), a tx with high enough fee will always be voted on by all nodes that see it
		- min fee can't be fixed for a block, because then an attacker can submit many low fee tx for block, wait for inter-slot fee to fall, and repeat

- alt idea
	- A) record fee in block header (consensed on OR determinisitic from timestamps), B) deterministically decide tx for block with combine_fn (which implements within-block fee scaling)


- tx cache management (optimization)
If a node obtains a transaction from a client or peer, they should perform two tests.

1. Is the fee high enough that the transaction could plausibly be added to a block before its tombstone block expires? In other words, given the current state of the blockchain, might the minimum (algorithmic) fee/weight ratio fall below this transaction's fee/weight ratio before the transaction dies? This test has one caveat, namely if the [tx cache is full](#tx-cache-cleaning), then only compare the tx's fee/weight ratio to the current minimum fee/weight ratio instead of projecting the absolute minimum ratio over the tx's lifetime.

2. Is the transaction valid? This is the normal validity check already performed.

If both tests pass, then store the transaction in the node's transaction cache and rebroadcast it to the node's peers.


## Rationale

#### Why not use SCP to consensus on minimum fees, and write them directly in block headers?

- Doing so introduces the risk of an absurdly high minimum fee being set, causing the network to get stuck. Mitigating this would require heuristics, and it is best to minimize heuristics wherever possible.
- It is necessary to actively increase the minimum fee as a slot progresses. Minimum fees written in block headers would be misleading to users.
- Minimum fees do not provide useful information to future auditors of the blockchain, so recording them in block headers is superfluous. Users can query a trusted node to learn what the minimum fee is, and nodes can keep track of the fee expectations of other nodes (they should all be using the same fee algorithm unless there is a serious community bifurcation).



## Backward compatibility

This proposal requires a transaction validation rule change (removes), so it must be implemented via a hard fork of some kind. It also recommends new SCP behavior, which nodes should implement for optimal performance.



## Reference implementation

Not completed yet.



## References

- None
