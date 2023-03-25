```
  MCIP: ?
  Layer: Consensus
  Title: Optional Tombstone Block
  Authors: koe <ukoe@protonmail.com>
  Status: Draft (DO NOT USE)
  Type: Standards Track
  Created: ?
  Requires: MCIP-[versatile scp]
```

## Abstract

Allow transactions to set an 'optional' tombstone block. The lifetime of such transactions (i.e. when waiting to be added to a block) can be managed locally by nodes.



## Motivation

In the current protocol, a transaction cannot be added to a block if its `tombstone_block <= block_index` (i.e. the block is being made after the tx has 'died') or `tombstone_block > block_index + 100` (i.e. the block being made isn't within 100 blocks of the tx dying). Moreover, if a node encounters a tx whose `tombstone_block` is invalid (based on the index of the block under construction), then the node will discard it (the node won't wait until the second condition is satisfied, even if reasonable to do so).

As a consequence, it is not possible to construct 'slow' transactions (e.g. for multisig/MPC, some atomic swap methods, and most payment channel approaches), which begin construction an arbitrary amount of time before they are submitted to the network. This is because an appropriate `tombstone_block` cannot be predicted in advance.

This proposal allows 'slow' MobileCoin transactions by not requiring that tx authors specify a `tombstone_block` value for every transaction.



## Tx construction/validation changes

- **Construction**: A tx author may set `tombstone_block` to either 0 or non-zero.
- **Validation**: When validating a transaction, `tombstone_block` fields set to 0 are always considered valid. Non-zero `tombstone_block`s will be evaluated with the existing tombstone block rules.



## Tx lifetime management

If a `tombstone_block` is set to 0, then nodes must locally manage the transaction's lifetime. The strategy recommended here has three parts.


### Part 1: `temp_tombstone_block`

When a node hears about a tx from a client or peer, and the node doesn't already have a local copy of the tx, then, if `tombstone_block == 0`, set a variable `temp_tombstone_block` and attach it to the local tx record, and rebroadcast the tx.

- `temp_tombstone_block = current slot + MAX_TOMBSTONE_BLOCKS`
	- Here, `current slot` is the slot the network is currently working on (not the local node's current slot - the local node might be behind the network).
	- `MAX_TOMBSTONE_BLOCKS`: Currently 100 blocks (v0 of protocol), this is the maximum window of blocks a tx is considered 'ok to add to a block'.

This value is only for local management of the tx, and should not be sent to other nodes or used in any context where inter-node determinism is required. If a node fully discards a tx then hears about it again, it will set a new value for `temp_tombstone_block` (unlikely in practice, but acceptable).

**Note**: When constructing a ballot in SCP with `combine_fn`, it is necessary to 'sort' the input statements first, which requires comparing transactions. The `temp_tombstone_block` should never be used for that purpose. Instead, if `tombstone_block == 0`, the transaction should be considered lower priority than tx with non-zero tombstone blocks that are otherwise 'equal' (e.g. subtract 1 [with integer overflow] from all tombstone values before comparing them).


### Part 2: nomination filtering

During MCIP-[versatile scp] nomination filtering, nodes should evaluate the `temp_tombstone_block` if it is non-zero.

Only vote for a transaction if its `temp_tombstone_block` passes the normal validity check for tombstone blocks. In other words, it must be in the range `[current_slot_index, current_slot_index + MAX_TOMBSTONE_BLOCKS]`. To be clear, a tx with non-zero `temp_tombstone_block` is *not* invalid if outside that range. It just shouldn't be voted for.


### Part 3: tx storage with buffer

Nodes store transactions in a tx cache while waiting for them to be put in a block. Currently they are purged from the cache when their `tombstone_block` values are `<=` the current slot being worked on (plus some default buffer blocks related to node operations called `max_externalized_slots`).

Since `temp_tombstone_block` is set locally by nodes, it can't be used naively for deciding whether or not to purge a tx from the cache. The reason is different nodes could have different `temp_tombstone_block` values, leading to a 'ping-pong' situation where one node purges a tx, a second node proposes it as part of SCP, the first node re-acquires the tx, the second node purges it, the first node proposes it as part of SCP, the second node re-acquires it, etc. (unlikely to occur in practice unless tx volume is so high that low-fee tx end up stuck in the tx cache for long periods of time).

When deciding whether a tx should be purged, use the following method:

```
#define TOMBSTONE_STORAGE_BUFFER = 1.5 * MAX_TOMBSTONE_BLOCKS;

fn should_purge_tx(Tx tx, Index slot_in_progress) -> Bool {
	Index tombstone_limit = slot_in_progress.saturating_sub(get_max_externalized_slots());
	Index tombstone = tx.tombstone_block;

	if (tx.temp_tombstone_block != 0) {
		tombstone_limit.saturating_sub(TOMBSTONE_STORAGE_BUFFER);
		tombstone = tx.temp_tombstone_block;
	}

	return tombstone < tombstone_limit;
}
```

With this approach, nodes should only purge a tx with non-zero `temp_tombstone_block` after all other nodes have stopped voting for it, since tx that are ineligible for voting will linger in the tx cache longer than the period of eligibility.

**Note1**: To optimize for storage, transactions with non-zero `temp_tombstone_block` that have died (and passed the `max_externalized_slots` mark) can be pruned down to the pair [transaction hash, `temp_tombstone_block`]. Nodes that receive a transaction from client/peer broadcast, or encounter it during nomination filtering, should not vote for or rebroadcast tx in the list of pruned transactions.

**Note2**: The buffer cannot prevent ping-pong if there are malicious nodes that disregard tombstone lifetime conventions for non-zero `temp_tombstone_block`s. Node operators must detect malicious nodes and blacklist them locally.



## Rationale

#### Why does this proposal use nomination filtering (MCIP-[versatile scp]), instead of letting nodes consider an optional tombstone invalid based on locally-defined ideas about when its tx expires?

Any difference in validation rules between nodes can cause nodes to get stuck during SCP. Nomination filtering allows nodes to disagree about something (when a tx with optional tombstone expires), without risking liveness.

#### Why is the tombstone block part of transactions in the first place?

The tombstone block has three purposes.

1. Tx authors can have definite knowledge of tx failure, i.e. if they see the blockchain height exceeds their tx's `tombstone_block`, then the tx definitely failed.

2. Nodes need to coordinate how long tx live in the tx cache. Since MobileCoin uses SCP, nodes cannot selfishly discard tx that they think are failures. A different node may try to vote to nominate it, forcing nodes that discarded it to re-acquire the tx. Coordinating tx purges is much more efficient (note that nomination filtering was not invented when MobileCoin was first launched).

3. Most importantly, `tombstone_block` helps tx authors coordinate with Fog ingress key expiration. A tx author must set the tx's `tombstone_block` to no greater than the lowest Fog ingress key expiration of the outputs created by the tx. Otherwise, a tx might be added to a block after an ingress key expired, causing one of the tx's outputs to be ignored by Fog instead of recorded properly. The intended recipient of the output, who presumably relies on Fog to find their outputs, wouldn't realize they own the output, and it may be very difficult/annoying for them to recover it with view-key scanning.

#### What is the impact of optional tombstone blocks on Fog outputs?

A transaction with `tombstone_block` set to 0 should **not** contain outputs destined to a Fog user, as a matter of best practice. Implementers that disregard this rule must take responsibility for the attendant UX difficulties of 'lost' Fog outputs.

Unfortunately, this means Fog cannot reliably be used in conjunction with 'slow' transactions (multisig/etc.). That is unless Fog ingress keys can be given substantially longer expiration dates than the expected duration of constructing a 'slow' tx.

The alternative methods for recovering outputs are:
- *view-key scanning*: Directly scanning on-chain outputs with the user's private view key.
	- View-key scanning can theoretically be optimized by setting a 'block range' limitation (i.e. the range of blocks where an owned output is expected to exist).
- `Receipts` (better named as `PaymentNotices`): If the recipient knows exactly what output to look for on-chain, then it is easy to find.



## Backward compatibility

This proposal only expands the interface for transaction-building, so it is backward compatible with existing transaction-builder implementations.

Since the proposal requires a transaction validation rule change (adds optional tombstone blocks), it must be implemented via a hard fork of some kind. It also recommends new SCP behavior and tx cache management, which nodes should implement for optimal performance.



## Reference implementation

Not completed yet.



## References

- None
