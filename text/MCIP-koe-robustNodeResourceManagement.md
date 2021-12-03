```
  MCIP: ?
  Title: Robust Node Resource Management
  Authors: koe <ukoe@protonmail.com>
  Status: Draft (DO NOT USE)
  Type: Informational
  Created: ?
  Requires: MCIP-[versatile scp], MCIP-[reducing nomination votes]
```

## Table Of Contents

- [Abstract](#Abstract)
- [Motivation](#Motivation)
- [Transaction queues](#Transaction-queues)
	- [Overflow reduction](#Overflow-reduction)
	- [Get tx from peer](#Get-tx-from-peer)
	- [Unvalidated tx](#Unvalidated-tx)
	- [Validated tx](#Validated-tx)
	- [Expired tx](#Expired-tx)
- [Encountering tx](#Encountering-tx)
	- [Client proposals](#Client-proposals)
	- [Peer broadcast](#Peer-broadcast)
	- [SCP messages](#SCP-messages)
- [Failure modes discussion](#Failure-modes-discussion)
	- [Storage failure](#Storage-failure)
	- [CPU limitations](#CPU-limitations)
	- [Invalid tx spam](#Invalid-tx-spam)
	- [Bandwidth/network limitations](#Bandwidthnetwork-limitations)
- [Threat model](#Threat-model)
- [Rationale](#Rationale)
- [Backward compatibility](#Backward-compatibility)
- [Reference implementation](#Reference-implementation)
- [References](#References)



## Abstract

Unlike PoW (and similar), SCP is a network coordination mechanism that requires active involvement from all/most participants/nodes. In a sense, these nodes construct a 'shared state' together. If a node encounters local resource constraints (memory exhaustion, CPU overload, or bandwidth/network bottlenecking), it can't prune local data or ignore transactions naively. Doing so may invalidate (or leave incomplete) that node's 'shared state', which would break the SCP 'eventual liveness' guarantee.

This proposal discusses a node infrastructure that, alongside various resource-management heuristics,

- separates 'shared state' data (i.e. transactions) from non-mandatory data, so if storage limitations are encountered the non-mandatory data can be safely discarded,
- defines the minimum transaction storage space required for nodes to be robust SCP participants in the face of overwhelming transaction volume (assuming MCIP-[reducing nomination votes] is implemented),
- prioritizes validating (and obtaining from peers) those transactions that are most likely needed for making progress in SCP,
- and describes heuristics to mitigate DDOS in the form of large volumes of invalid transactions.



## Motivation

The existing validator node implementation does not have any robust techniques for handling resource exhaustion (memory, CPU, or bandwidth). This means nodes are not hardened against a well-funded adversary (who may or may not have root access to a node) willing to overload the network with transactions (either valid or invalid). Related to this, the maximum network transaction throughput is not well-optimized.



## Transaction queues

Transactions will be stored in a variety of queues, carefully moved between the queues, and judiciously discarded when appropriate.


### Overflow reduction

The primary mechanism for discarding transactions is a function `tx_cache_overflow_reduction()`, which has the following behavior.

- If the overall transaction cache exceeds `MAX_TX_CACHE_SIZE_BYTES`, remove the lowest elements in tx queues until the cache no longer overflows.
	- `MAX_TX_CACHE_SIZE_BYTES`: A configuration value chosen by the node operator. It should be low enough that overflowing the cache by one or two max-size transactions won't cause catastrophic memory failure (the overflow reduction mechanism is to first overflow slightly, then discard tx as necessary).
	- **Removal priority**: `unvalidated_tx_queue_untrusted` -> `unvalidated_tx_queue_unreq` -> `validated_tx_queue_unreq`
		- If a queue is shorter than (or equal to) `TX_QUEUE_PROTECTED_LENGTH`, then it is 'protected' (tx can't be discarded), and the removal mechanism should look at the next queue.
			- `TX_QUEUE_PROTECTED_LENGTH = MAX_TRANSACTIONS_PER_BLOCK`
		- **Expected invariant**: If all queues are too short to discard any tx, then the tx cache should not be able to overflow. In concrete terms, this function should never try to remove a tx from `validated_tx_queue_unreq` and fail.

#### Rationale/comments

- Transactions can have large variations in size, so cache management is based mainly on bytes rather than transaction counts.
- Reduction priority starts with unvalidated client transactions, which are least trusted because a client-based adversary can't be easily blacklisted (they can just spoof infinite client identities). Next are unvalidated tx from peers that don't appear necessary for SCP. Peers that recommend an invalid tx can be easily blacklisted. Last are validated tx (that don't appear necessary for SCP). Validated tx are lowest priority for discarding so no adversary can submit/recommend tx in order to make a node discard tx it already spent time validating.
- As noted [below](#get-tx-from-peer), the queue of transactions to 'get' from other nodes is not touched by `tx_cache_overflow_reduction()` because tx hashes are much smaller than full transactions.
- The queues that can't be reduced here are `get_tx_queue_unreq`, `get_tx_queue_req`, `unvalidated_tx_queue_req`, `validated_tx_queue_req`, `expired_tx_queue_externalized`, and `expired_tx_queue_optional_tombstone`.
- Queues are protected below a certain length to avoid an edge case. Namely, since MCIP-[versatile scp] `nominate_fn` allows a node to consider tx 'valid but undesirable', it is conceivable for the `validated_tx_queue_unreq` queue to become full of undesirable tx. Those tx are unlikely to land in any slots, so won't be removed from queues until their tombstone block expires, and can only be replaced in the same queue by desirable inputs. However, if the cache is completely saturated by `validated_tx_queue_unreq`, then it is impossible to pass any unvalidated tx through the validation process, because `tx_cache_overflow_reduction()` will discard it the moment it is added to any unvalidated-tx queue.
- A node that implements this proposal cannot safely be deployed on a machine with too little cache available for transaction storage.
- This proposal only satisfies the goal of 'robustness' if, assuming a node has the recommended minimum storage, it is impossible for the node's tx cache to overflow when the discardable queues are below `TX_QUEUE_PROTECTED_LENGTH` and the MobileCoin network is [intact](https://www.stellar.org/papers/stellar-consensus-protocol) (in particular, if there are no full quorums of malicious nodes, nor a set that `v`-blocks all honest nodes). See the [threat model](#threat-model) for more precise expectations about this proposal.


### Get tx from peer

Given a tx hash, the node wants to get a fully copy of the transaction. While waiting for the tx to be obtained, its hash will be stored in one of the following queues.

Each element in these queues is a pair between a transaction hash and 'recommendation list'. The list contains all the peers who at one time recommended the transaction (e.g. via peer broadcast or as part of an SCP message). If at least one client has recommended a tx, then the list should contain one 'client recommendation' ID (no more than one).

- `get_tx_queue_unreq`: Transactions not required for SCP.
	- **element type**: A pair between a transaction hash and a 'recommendation list'.
	- **sort**: number of non-blacklisted recommendations (descending) -> transaction hash (ascending)
	- **max elements**: `5*MAX_TRANSACTIONS_PER_BLOCK`
		- **after insert**: If the queue is full, discard the lowest element.
	- **after tx obtained**: Insert the transaction to `unvalidated_tx_queue_unreq` and call `tx_cache_overflow_reduction()`.

- `get_tx_queue_req`: Transactions required for SCP.
	- **element type**: A pair between a transaction hash and a 'recommendation list'.
	- **sort**: last-in-first-out (LIFO)
	- **max elements**: unbounded
	- **after tx obtained**: Insert the transaction to `unvalidated_tx_queue_req` and call `tx_cache_overflow_reduction`.

#### Rules

- **prioritize**: `get_tx_queue_req` -> `get_tx_queue_unreq`
- **after slot is externalized**: move `get_tx_queue_req` to `get_tx_queue_unreq`
- Do not allow tx into queues if sourced from *only* blacklisted nodes.
- A node should prioritize setting up remote connections for obtaining tx from the following sources: `get_tx_queue_req` -> `get_tx_queue_unreq` -> [peer broadcast](#peer-broadcast) -> [client proposals](#client-proposals).

#### Rationale/comments

- Each element of these queues is small (in bytes) compared to a full transaction, so `get_tx_queue_unreq` is not in the path of `tx_cache_overflow_reduction()`, which could empty the entire queue just to compensate for one overflowing transaction. Instead, `get_tx_queue_unreq` has a maximum size and the lowest element is discarded on overflow.
- By storing a recommendation list alongside tx, nodes can both identify tx recommended by a blacklisted node (i.e. if a node becomes blacklisted after the recommendations are recorded) and disambiguate tx recommended by 'only' blacklisted nodes from tx recommended by a blacklisted node but also recommended by a non-blacklisted node and/or client. The latter is important so a malicious node can't as easily recommend a valid tx, then broadcast an invalid tx so other nodes will blacklist the node and think the valid tx needs to be discarded (if they haven't validated it yet).
	- Tx hashes in `get_tx_queue_req` also have a recommendation list in case they are moved into `get_tx_queue_unreq` for one reason or another (e.g. at the end of a slot, or due to MCIP-[reducing nomination votes] `nomination_reduction_fn`).
- These queues have no knowledge of transaction fees, so queue priority differs from other queue types.
	- The queue `get_tx_queue_unreq` first sorts by the number of non-blacklisted peers (including +1 if there is a 'client ID'). Transactions recommended by many peers are both more likely to be valid and more likely to be useful for making progress in the current SCP slot, than tx recommended by few peers.
	- The queue `get_tx_queue_req` is LIFO instead of FIFO because the most recent tx the node sees is required for making progress in SCP is probably most important for actually making progress. This is because a 'required' tx can become less important if newer tx overcome it (e.g. higher-fee tx, tx in ballots with better chances of succeeding). Note that this 'optimization' is only speculative and may not have any effects in practice.
- The act of getting a transaction from a peer should by asynchronous relative to other node operations (validating tx and executing SCP).


### Unvalidated tx

The node wants to validate a transaction. While waiting to be validated, the tx will live in these queues.

- `unvalidated_tx_queue_untrusted`: Transactions obtained from a client.
	- **element type**: A pair between a full transaction (unvalidated) and a 'recommendation list' (see [above](#get-tx-from-peer)).
	- **sort**: fees (descending) -> tombstone (ascending) -> transaction hash (ascending)
	- **max elements**: unbounded
		- protected by `TX_QUEUE_PROTECTED_LENGTH`
	- **after tx validated**: insert transaction to `validated_tx_queue_unreq`

- `unvalidated_tx_queue_unreq`: Transactions obtained from a peer node.
	- **element type**: A pair between a full transaction (unvalidated) and a 'recommendation list' (see [above](#get-tx-from-peer)).
	- **sort**: number of non-blacklisted recommendations (descending) -> fees (descending) -> tombstone (ascending) -> transaction hash (ascending)
	- **max elements**: unbounded
		- protected by `TX_QUEUE_PROTECTED_LENGTH`
	- **after tx validated**: Insert the transaction to `validated_tx_queue_unreq`.

- `unvalidated_tx_queue_req`: Transactions required for SCP.
	- **element type**: A pair between a full transaction (unvalidated) and a 'recommendation list' (see [above](#get-tx-from-peer)).
	- **sort**: last-in-first-out (LIFO)
	- **max elements**: unbounded
	- **after tx validated**: Insert the transaction to `validated_tx_queue_req`.

#### Rules

- **prioritize**: `unvalidated_tx_queue_req` -> `unvalidated_tx_queue_unreq` -> `unvalidated_tx_queue_untrusted`
	- Before converting an externalized slot into a block, the block contents must be fully re-validated. The validation thread may be commandeered for that purpose.
- **after slot is externalized**: Move `unvalidated_tx_queue_req` into `unvalidated_tx_queue_unreq`.
	- After `unvalidated_tx_queue_unreq` is updated and the slot index is incremented, delete all tx with `tombstone_block` less than the slot index.
		- (MCIP-[optional tombstone block]) Move all transactions with `temp_tombstone_block` less than the slot index into `expired_tx_queue_optional_tombstone`.
- Every tx is validated at most two times to optimize CPU usage. First, when the tx is acquired, and second, when the tx is externalized for a new block. Since a tx may become invalid while sitting in storage, the SCP `validity_fn` should perform consistency checks (instead of fully re-validating transactions).
	- **Consistency checks**: A transaction is inconsistent/invalid if it has a key image or txout public key that appears on-chain, if any of its membership proof root elements are inconsistent with the chain, if its tombstone block is invalid, or if its fee is invalid.
		- A membership proof root element will only become inconsistent (after being consistent) if the node operator rolls back part of their local ledger. Implementers must take care that any time an operator is allowed to roll back their local ledger, all validated transactions' membership proofs are accurate. The easiest way to do this is to move validated transactions into their corresponding unvalidated queues (`_req` or `_unreq`) during the roll-back.
		- A transaction's fee may become invalid if the hard-coded minimum fee changes due to a hard fork. It may be wisest to transfer all validated transactions to the unvalidated queues whenever a hard fork occurs.
- If a transaction is invalid, immediately blacklist all peers that recommended that transaction.
	- If a tx is invalid *only* because it fails a consistency check, then do not blacklist the peers (they may have recommended it before the tx became invalid).
	- Do not blacklist the special 'client ID', which is just a stand-in for 'some client out there recommended it'.
	- If, after blacklisting a peer, any of the tx recommended by that peer (in 'get' or 'unvalidated' queues) ends up with an empty recommendation list, then discard the transaction. If the only remaining recommendation is the 'client ID', move 'get' tx to `_unreq` and 'unvalidated' tx to `_untrusted`.
	- **Note**: If a peer is blacklisted, then the node operator should be notified in case they need to reconfigure their quorum set or investigate a bug.
- If a transaction is valid, rebroadcast it to the node's peers.
- The node should not create connections to blacklisted nodes, neither for obtaining transactions, peering transactions, nor obtaining/sending SCP messages.
- Nodes must rate-limit client connections so clients cannot submit transactions faster than the node can validate them. In practice, the client connection rate-limit can be unrestrictive if no invalid transactions are detected, then throttled down if they are detected.

#### Rationale/comments

- Suppose there is an adversary submitting many invalid transactions as a 'client'. Assume the rate of invalid tx submission is `INV_R`, the rate of valid tx submission is `VAL_R`, and the rate of validating tx is `VD_R`.
	- If `unvalidated_tx_queue_untrusted` uses sorting to decide which transaction should be removed first by `tx_cache_overflow_reduction()`, then the rate valid tx will succeed is `min(VD_R - INV_R, VAL_R)`.
		- Given this, if `VD_R <= INV_R`, then the node will be unable to validate any transactions submitted by honest users. This is because given enough time the adversary can fill up `unvalidated_tx_queue_untrusted` with invalid transactions, and continually push out honest tx by abusing the sorting scheme (or at the very least ensure there are always invalid tx with higher priority than valid tx).
		- Note that even randomly sampling `unvalidated_tx_queue_untrusted` to decide the order of tx to validate cannot change the rate valid tx will enter the validator (except the case `VD_R == INV_R`). If we assume the adversary will abuse the sorting scheme so only honest tx will be discarded on overflow, then the equilibrium point where honest tx are not pushed out any more is when the rate dishonest tx enter the validator equals the rate dishonest tx are submitted (i.e. when the queue's composition is `(INV_R/VD_R) x 100 %` invalid tx).
	- On the other hand, if `unvalidated_tx_queue_untrusted` is FIFO (i.e. on overflow, an effectively random tx will be discarded), then the rate honest tx are validated will be `min(VD_R * VAL_R/(INV_R + VAL_R), VAL_R)`. The queue can never be blocked. However, if `VD_R < VAL_R + INV_R`, then it will be possible for honest transactions with high fees to be discarded even though low-fee honest tx succeed! This is not acceptable from a user experience point of view, where users should be confident that successfully submitted high fee tx are more-or-less guaranteed to land in a block.
		- Instead, nodes must rate-limit client connections so the condition `VD_R <= INV_R` can never be true. Peers don't need to be rate limited since they can be blacklisted as soon as they recommend an invalid tx.
			- The client rate-limit can be increased by around 2x if the 'order of expensive validation operations' is randomized. Validating a tx requires a number of discrete steps. An adversary hoping to waste a node's CPU cycles will choose the most expensive way for a tx to fail (i.e. the very last validation step). If expensive validation steps (e.g. verifiying Bulletproofs and MLSAG signatures) are both randomized and executed at the end of the validation procedure, then the average validation time of an adversarially invalid tx will be reduced by 50%.
- The queue `unvalidated_tx_queue_unreq` sorts by tombstone block after fees. In the rare case that two transactions have the same fee, the tx closer to dying is considered higher priority for adding to a block.
- Validating transactions should by asynchronous relative to other node operations (obtaining tx and executing SCP).


### Validated tx

Transactions the node has validated at least once.

- `validated_tx_queue_unreq`: Transactions not required for SCP.
	- **element type**: A full transaction (validated).
	- **sort**: fees (descending) -> tombstone (ascending) -> transaction hash (ascending)
	- **max elements**: unbounded
		- protected by `TX_QUEUE_PROTECTED_LENGTH`

- `validated_tx_queue_req`: Transactions required for SCP.
	- **element type**: A full transaction (validated).
	- **sort**: none specified
	- **max elements**: unbounded

#### Rules

- **after slot is externalized**: Move each externalized tx into `expired_tx_queue_externalized` and set their `store_until` values equal to the externalized slot's index plus the constant `MAX_EXTERNALIZED_SLOTS`. Move `validated_tx_queue_req` into `validated_tx_queue_unreq`.
	- **Expected invariant**: All tx externalized in a slot should exist in `validated_tx_queue_req` when the slot is externalized.
	- After `validated_tx_queue_unreq` is updated and the slot index is incremented, remove all tx with `tombstone_block` less than the slot index.
		- (MCIP-[optional tombstone block]) Move all transactions with `temp_tombstone_block` less than the slot index into `expired_tx_queue_optional_tombstone`.
- During an SCP slot:
	- The node can propose transactions from `validated_tx_queue_unreq` to vote to nominate. Each time the node wants to propose transactions, it should propose the top `MAX_NOMINATION_VOTES` transactions from that queue (recall MCIP-[reducing nomination votes]).
		- This replaces `MAX_PENDING_VALUES_TO_NOMINATE` in the current implementation.
		- It is fine if a tx is proposed more than once. Duplicates should be ignored when adding tx to the list of tx to vote to nominate.
	- If a transaction is added to the list of transactions the node wants to vote to nominate (i.e. after MCIP-[versatile scp] `nominate_fn` recommends it, and assuming the transaction has been validated and passed a consistency check), then move it to `validated_tx_queue_req`.
		- **Expected invariant**: Only tx in `validated_tx_queue_unreq` should be added to the list of tx to vote to nominate.
	- In MCIP-[reducing nomination votes] `nomination_reduction_fn`, if a transaction is removed from the list of tx to vote to nominate, then that transaction should be moved from `validated_tx_queue_req` back to `validated_tx_queue_unreq`.

#### Rationale/comments

- It is safe for `nomination_reduction_fn` to remove transactions from `validated_tx_queue_req` because that function should only be called during the nomination phase.
	- During the nomination phase, the node has accepted no ballots, so no tx required for a ballot can be lost.
	- The list of tx to vote to nominate should not include any tx the node has accepted for nomination.
	- Therefore a tx that can be removed from `validated_tx_queue_req` by `nomination_reduction_fn` should be 'required' only because it is in the list of tx to vote to nominate. Nomination votes are safe to discard according to the [FBA model](https://www.stellar.org/papers/stellar-consensus-protocol).
- Moving transactions out of `validated_tx_queue_req` in `nomination_reduction_fn` prevents edge cases where `validated_tx_queue_req` could grow arbitrarily large. For example, if an adversary client/node recommends a series of sets of valid tx with increasing fees, and the node iteratively replaces its existing votes to nominate with tx from that series.
- Transactions in `validated_tx_queue_req` do not need to be proposed for nomination, since tx in that queue should already be part of the node's SCP state in some way (see [below](#SCP-messages).
- In the current implementation, `MAX_PENDING_VALUES_TO_NOMINATE` is a performance heuristic for limiting how many transactions can pass through an SCP slot. For simplicity, however, it is better if performance goals are met by setting the constant `MAX_TRANSACTIONS_PER_BLOCK` to a reasonable value. This is why `MAX_NOMINATION_VOTES`, which is a function of `MAX_TRANSACTIONS_PER_BLOCK`, replaces `MAX_PENDING_VALUES_TO_NOMINATE`.


### Expired tx

Transactions that have expired, but the node wants to keep around temporarily, are stored in these queues.

- `expired_tx_queue_externalized`: Tx externalized in a block, that the node wants to keep a record of temporarily in case a slow peer node wants to request a copy.
	- **element type**: A pair between a full transaction and a `store_until` value.
	- **sort**: none specified
	- **max elements**: unbounded

- `expired_tx_queue_optional_tombstone`: An MCIP-[optional tombstone block] transaction with `tombstone_block == 0` that has expired, but the node needs to keep a record temporarily in case there are inconsistencies in expiry block across the network.
	- **element type**: A pair between a transaction hash and a `store_until` value.
	- **sort**: none specified
	- **max elements**: unbounded

#### Rules

- At the end of a slot after the slot index has incremented:
	1. Remove from these queues all queue elements whose `store_until` values are less than the new slot index (i.e. the index of the slot that will be constructed next).
	2. (MCIP-[optional tombstone block]) Check all the 'unvalidated' and 'validated' queues introduced by this proposal.
		- If a tx has `tombstone_block == 0`, `temp_tombstone_block <= new slot index`, and the tx is not already in `expired_tx_queue_optional_tombstone`, then move the tx into `expired_tx_queue_optional_tombstone`. Set `store_until = temp_tombstone_block + TOMBSTONE_STORAGE_BUFFER`.
		- To comply with MCIP-[optional tombstone block], MCIP-[verstile scp] `nominate_fn` should not recommend a tx if its hash exists in `expired_tx_queue_optional_tombstone`.
		- A tx is allowed to move through the 'required' queues even if it exists in `expired_tx_queue_optional_tombstone`, since it may be required for SCP (sitting in `expired_tx_queue_optional_tombstone` just means the node shouldn't vote to nominate it).

#### Rationale/comments

- The queue `expired_tx_queue_optional_tombstone` does not need to be touched by `tx_cache_overflow_reduction()` because its maximum size is insignificant. A tx is added to the queue when it has lived in tx storage for `MAX_TOMBSTONE_BLOCKS` slots without being added to any block. If we assume tx storage is crammed full of minimum-size tx and all those tx are added to `expired_tx_queue_optional_tombstone`, the relative size of `expired_tx_queue_optional_tombstone` compared to the maximum tx storage will be `40 bytes / min tx size` (given an 8-byte `store_until` value and 32-byte tx hash). The minimum tx size is on the order of at least `4000 bytes`, so one full set of tx will compress down to 1% or less of maximum tx storage. Moreover, since a tx can only make it into `expired_tx_queue_optional_tombstone` after `MAX_TOMBSTONE_BLOCKS` blocks have passed, and tx in `expired_tx_queue_optional_tombstone` will be discarded after `TOMBSTONE_STORAGE_BUFFER` blocks, at most `ceil(TOMBSTONE_STORAGE_BUFFER / MAX_TOMBSTONE_BLOCKS)` full sets of transactions can be held in `expired_tx_queue_optional_tombstone` at once. Currently that ratio is 1.5, so the queue can occupy at most 2% of maximum tx storage.



## Encountering tx

A transaction can be encountered by a node in three ways. A client can propose it, a peer node can recommend it via broadcast, or a peer node can include it in an SCP message. Furthermore, transactions seen in SCP messages may become 'required' for local storage if the node is forced to accept them (either during nomination or as part of a ballot).


### Client proposals

- When a tx has been obtained from a client.
	1. If the tx is in `get_tx_queue_unreq`, remove it from that queue but save the attached recommendation list.
	1. If the tx is in `get_tx_queue_req`, remove it from that queue but save the attached recommendation list, insert the tx into `unvalidated_tx_queue_req`, and call `tx_cache_overflow_reduction()`.
	1. If the tx is not in any 'unvalidated', 'validated', or 'expired' queues, then add it to `unvalidated_tx_queue_untrusted` and call `tx_cache_overflow_reduction()`.
	1. If the tx is in any 'unvalidated' queues, add the 'client recommendation' ID and the saved recommendations (if they exist) to the tx's recommendation list (disallow duplicates).
	1. (MCIP-[optional tombstone block]) If `tombstone_block == 0`, `temp_tombstone_block == 0`, and the transaction is not in `expired_tx_queue_optional_tombstone`, then set `temp_tombstone_block` according to (MCIP-[optional tombstone block]).

#### Rationale/comments

- Transactions obtained from clients are considered 'untrusted', compared to those obtained from peers. This is because an adversary can easily spoof arbitrary numbers of client IDs, while peer IDs are hard to spoof and can be blacklisted aggressively if they misbehave.
	- Untrusted transactions are lowest priority for validating and highest priority for discarding tx when there is storage overflow.
- (MCIP-[optional tombstone block]) If a transaction is in `expired_tx_queue_optional_tombstone`, it is allowed to exist in one of the 'required' queues as well. However, it must be discarded after the slot ends if it fails to land in the block being constructed, since the node considers it 'expired' (undesirable). To make sure that happens, the rules around `temp_tombstone_block` are carefully defined in this proposal.
	- **Expected invariant**: When a new slot begins, no transactions in `expired_tx_queue_optional_tombstone` should exist in any other queues.


### Peer broadcast

- When a tx has been obtained via peer broadcast (or upon request due to the `get_` queues).
	1. If the tx is in `get_tx_queue_unreq`, remove it from that queue but save the attached recommendation list.
	1. If the tx is in `get_tx_queue_req`, remove it from that queue but save the attached recommendation list, insert the tx into `unvalidated_tx_queue_req`, and call `tx_cache_overflow_reduction()`.
	1. If the tx is in `unvalidated_tx_queue_untrusted`, move it to `unvalidated_tx_queue_unreq`.
	1. If the tx is not in any 'unvalidated', 'validated', or 'expired' queues, then add it to `unvalidated_tx_queue_unreq` and call `tx_cache_overflow_reduction()`.
	1. If the tx is in any 'unvalidated' queues, add the peer's ID and the saved recommendations (if they exist) to the tx's recommendation list (disallow duplicates).
	1. (MCIP-[optional tombstone block]) If `tombstone_block == 0`, `temp_tombstone_block == 0`, and the transaction is not in `expired_tx_queue_optional_tombstone`, then set `temp_tombstone_block` according to (MCIP-[optional tombstone block]).
- Given multiple peers trying to connect/send a message, randomly select which one to prioritize talking to.

#### Rationale/comments

- Randomly selecting which peer to connect to mitigates malicious nodes who spam connections.


### SCP messages

- When a tx hash is seen in an SCP message.
	1. If the tx is not in any queues, add it to `get_tx_queue_unreq`.
	1. If the tx is in any 'get tx' or 'unvalidated' queues, add the peer's ID to the tx's recommendation list (disallow duplicates).

- When a tx is accepted during SCP. Either it is accepted nominated, or a ballot containing the tx has been accepted prepared/committed. Allowing a node to detect all of these tx may require modifications to the SCP implementation.
	1. If the tx is not in any queues (except `expired_tx_queue_optional_tombstone`), add it to `get_tx_queue_req`.
	1. If the tx is in any 'get tx', 'unvalidated', or 'validated' queues, and is not in the `_req` variant, then move it into the `_req` variant.
	1. Do not add the tx to any of the node's SCP messages unless it is in `validated_tx_queue_req`.

#### Rationale/comments

- Transactions accepted during SCP should in *most* cases already be in a queue somewhere. At the very least, in `get_tx_queue_unreq`, because the node should look through new SCP messages before checking what statements can be accepted. The one edge case where that is not true is when a transaction was discarded due to storage overflow between the time it was last seen (i.e. assessed for inclusion in a queue) and the moment the node accepted it.
- **Optional optimization**: Instead of adding all accepted transactions to `_req` queues, only add 'state-changing' transactions. Specifically, if a tx is in a ballot that is accepted prepared/committed, only add that tx to a `_req` queue if having that tx in `validated_tx_queue_req` would affect the node's SCP message state (the B, P, PP, H, or C ballots).
- **Warning**: Since this proposal recommends an asynchronous node architecture, care must be taken that 'required' tx can't slip through any cracks and fail to land in a required queue (only tx in `_req` queues are guaranteed not to be discarded). The danger-points are transactions in the process of being requested from peers (in `get_tx_queue_unreq`) and transactions in the process of being validated (in `unvalidated_tx_queue_untrusted` and `unvalidated_tx_queue_unreq`).



## Failure modes discussion

There are four 'areas of failure' this proposal attempts to address. These are storage failure (out-of-memory errors), CPU bottlenecking, invalid transaction blockage, and bandwidth/network bottlenecking.


### Storage failure

A node's tx cache can get full if it is overwhelmed by client and peer broadcasts. Since a node needs a local copy of any transaction that the network 'might' add to a block (i.e. all tx required by SCP), they can't discard transactions naively.

In this proposal, a node will encounter an out-of-memory failure if the transaction cache is saturated, it tries to insert an element into a queue, and it is impossible to discard any data. Un-discardable data is as follows.

- `get_tx_queue_unreq`: up to `5 * MAX_TRANSACTIONS_PER_BLOCK * 432` bytes
	- `MAX_TRANSACTIONS_PER_BLOCK` = 5000 (in v0)
	- `432`: `32` bytes for a transaction hash plus `50 * 8 = 400` bytes for 50 8-byte recommendations (guesstimate for upper bound)
- `get_tx_queue_req`: assume it is empty if `validated_tx_queue_req` is at max size
- `unvalidated_tx_queue_untrusted`: up to `TX_QUEUE_PROTECTED_LENGTH * MAX_TX_SIZE` bytes
	- `MAX_TX_SIZE`: 600 kB (in v0, including both the tx and reference membership proofs)
- `unvalidated_tx_queue_unreq`: up to `TX_QUEUE_PROTECTED_LENGTH * MAX_TX_SIZE` bytes
- `unvalidated_tx_queue_req`: assume it is empty if `validated_tx_queue_req` is at max size
- `validated_tx_queue_unreq`: up to `TX_QUEUE_PROTECTED_LENGTH * MAX_TX_SIZE` bytes
- `validated_tx_queue_req`: up to `MAX_REQUIRED_TX_APPROX * MAX_TX_SIZE` bytes
	- `MAX_REQUIRED_TX_APPROX = 2 * MAX_NOMINATION_VOTES`
- `expired_tx_queue_optional_tombstone`: up to approximately `0.02 * TX_CACHE_BUFFER_BYTES` bytes
	- `TX_CACHE_BUFFER_BYTES`: the amount of memory allocated for storing transactions
- `expired_tx_queue_externalized`: up to `MAX_EXTERNALIZED_SLOTS * MAX_TRANSACTIONS_PER_BLOCK * MAX_TX_SIZE` bytes

Now define the minimum storage required to support this proposal.

```
STORAGE_SAFETY_FACTOR = 2

MIN_TX_CACHE_BUFFER_BYTES = STORAGE_SAFETY_FACTOR * (
		5 * MAX_TRANSACTIONS_PER_BLOCK * 432 +
		TX_QUEUE_PROTECTED_LENGTH * MAX_TX_SIZE +
		TX_QUEUE_PROTECTED_LENGTH * MAX_TX_SIZE +
		TX_QUEUE_PROTECTED_LENGTH * MAX_TX_SIZE + 
		MAX_REQUIRED_TX_APPROX * MAX_TX_SIZE +
		0.02 * TX_CACHE_BUFFER_BYTES +
		MAX_EXTERNALIZED_SLOTS * MAX_TRANSACTIONS_PER_BLOCK * MAX_TX_SIZE
	)

MIN_TX_CACHE_BUFFER_BYTES = STORAGE_SAFETY_FACTOR * (100 / 96) * MAX_TRANSACTIONS_PER_BLOCK * MAX_TX_SIZE * (
		2160 / MAX_TX_SIZE +
		3 * TX_QUEUE_PROTECTED_LENGTH / MAX_TRANSACTIONS_PER_BLOCK +
		2 + (NOMINATION_VARIANCE_FACTOR / 50) +
		MAX_EXTERNALIZED_SLOTS
	)

MIN_TX_CACHE_BUFFER_BYTES = 40 GB (with v0 of the protocol, and constants defined in this proposal and MCIP-[reducing nomination votes])
```

Currently, `MAX_TRANSACTIONS_PER_BLOCK = 5000` is an unrealistic number. If it were reduced to `500`, then `MIN_TX_CACHE_BUFFER_BYTES = 4.0 GB` would be much more reasonable. MCIP-[optimize membership proofs] would further reduce this to `1.6 GB` by reducing `MAX_TX_SIZE` from \~600 to \~235 kB.

#### Discussion

- The linchpin of storage management is the value `MAX_REQUIRED_TX_APPROX`. Since 'required' transactions can't be discarded, 'required' queues can theoretically grow to arbitrary size. Practically speaking, however, MCIP-[reducing nomination votes] puts strong limitations on how many tx can land in 'required' queues via `MAX_NOMINATION_VOTES`. For this analysis, it is assumed only `MAX_NOMINATION_VOTES` tx can be accepted during SCP, and only (at most) an additional `MAX_NOMINATION_VOTES` tx may exist in `_req` queues due to the node's locally-defined set of nomination votes.
- There are six rules and heuristics that facilitate robust storage management.
	1. 'Required' transactions are carefully segregated from 'unrequired' transactions, so the latter can be discarded if there is storage overflow.
	1. MCIP-[reducing nomination votes] `MAX_NOMINATION_VOTES` limits how many transactions nodes can vote to nominate at a time. Even if individual nodes change their nomination votes many times, it is unlikely that the overall network will end up accepting more than `MAX_NOMINATION_VOTES` nominated tx in the short time between a slot beginning and the nomination phase ending.
	1. Similarly, `MAX_TRANSACTIONS_PER_BLOCK` limits how many tx may be added to a ballot. This constant is essential for defining reasonable queue sizes that won't affect maximum transaction throughput (under non-adversarial conditions).
	1. The `_unreq` queues (other than `get_tx_queue_unreq`) have a minimum size in case `validated_tx_queue_unreq` overflows with tx that are 'undesirable' according to `nominate_fn`.
		- If there were no minimum size for other queues, then it would be impossible to add new transactions to storage that are 'desirable' according to `nominate_fn`. As a result, the node would get stuck, unable to vote to nominate transactions nor acquire transactions worth voting for. In the worst case, the entire network could get stuck.
	1. Externalized transactions are only stored for `MAX_EXTERNALIZED_SLOTS` slots, which is as small as possible to prevent externalized tx from overwhelming the tx cache.
	1. (MCIP-[optional tombstone block]) Transactions with `tombstone_block == 0` are pruned down to a 40-byte struct when they expire.


### CPU limitations

Nodes are bottlenecked on how fast they can validate transactions. This ultimately constrains the network's maximum transaction throughput.

#### Discussion

There are three rules and heuristics that facilitate robust use of CPU cycles.

1. When selecting a transaction to validate, nodes first prioritize tx that are 'required' for SCP, then tx that are recommended by many peers (presumably closest to being 'required').
1. The constant `MAX_TRANSACTIONS_PER_BLOCK` limits how many transactions can be added to a block. While this doesn't have a large impact on overall transaction throughput, it does impact the rate at which slots can be completed. If it's too large, then time spent validating all the transactions required to complete a slot (at maximum transaction volume) will be much larger than network-related latency effects.
	- Constants like `MAX_TRANSACTIONS_PER_BLOCK` and `NOMINATION_VARIANCE_FACTOR` are configuration values that must be defined based on empirical analysis of the MobileCoin network.
1. Transactions are only fully validated twice. First, when they are added to storage, and second, when they are externalized as part of a slot. This is the absolute minimum amount of validation to ensure both SCP liveness and SCP safety guarantees are supported.
	- Only tx that have been fully validated should be added to SCP messages. This way no tx can become accepted/confirmed if they are invalid, which could cause nodes that see invalid tx are being accepted/confirmed to think the entire network is malicious.
		- 'Consistency checks' have the effect of 'updating' a tx's validity state in the period between receiving a tx and externalizing it.
	- For robustness, transactions are always fully re-validated before being added to a block. If that is not done, then any small implementation bug could cause an invalid tx to be added to a block, which would violate the SCP safety guarantee.


### Invalid tx spam

If a node is inundated by invalid transactions, it should not be completely prevented from participating in SCP or acquiring transactions from honest users.

#### Discussion

There are four rules and heuristics for mitigating invalid tx spam.

1. Unvalidated transactions are split between, with increasing priority, 'untrusted' (from clients), 'unrequired' (from peers), and 'required' (necessary for SCP). Higher-priority tx are passed to the tx validator first.
	1. 'Unrequired' transactions are sorted primarily based on how many peers recommend them. This increases the chances that a given tx passed to the validator is in fact valid.
	1. Peers that recommend invalid transactions are aggressively blacklisted, so future tx recommended by those peers can be ignored.
1. Client connections must be rate-limited so the node cannot encounter invalid transactions faster than it can process them. In practice, the client connection rate-limit can be unrestrictive if no invalid transactions are detected, then throttled down if they are detected.
1. (Optional) High-cost validation steps can be performed in random order so a malicious tx author cannot, on average, construct invalid transactions that are maximally-expensive to validate.
1. (Optional) A node can limit client connections to known/trusted sources and blacklist those that misbehave.


### Bandwidth/network limitations

A node can get bottlenecked on downloading transactions from outsiders, and generally opening communication channels with those outsiders.

#### Discussion

There are five rules and heuristics for managing interactions with outsiders (clients and peers).

1. Obtaining tx (when the tx hash is known) has this priority order: 'required' tx -> 'unrequired' tx (sorted by number of recommending peers).
1. Peer nodes that recommend invalid txs are blacklisted aggressively.
1. The constant `MAX_TRANSACTIONS_PER_BLOCK` limits how many tx can plausibly be useful during a slot, which affects how many tx will pass through `get_tx_queue_req`. If the constant is too high then a node can be bottlenecked on obtaining tx.
1. A node should prioritize setting up remote connections in the following order: `get_tx_queue_req` -> `get_tx_queue_unreq` -> peer broadcast -> client proposals.
1. Given multiple peers trying to connect/send a message, randomly select which one to prioritize talking to. This handles malicious nodes who spam connections.



## Threat model

A brief discussion of the threat model for the node infrastructure discussed in this proposal.

#### Assumptions

1. Let there be `NUM_HONEST` honest nodes. Each honest node:
	- Has at least `MIN_TX_CACHE_BUFFER_BYTES` storage available for transactions.
	- Has equivalent CPU and bandwidth/network capabilities.
		- Worst-case (slowest) transactions can be validated at rate `HON_VALIDATE_PER_S`.
		- Client-node connections are rate-limitted to at most `HON_CONNECT_PER_S`.
1. The network is intact (there are enough honest nodes to make a quorum, and honest nodes are well-connected).
1. The adversary:
	- Has access to infinite IP addresses (client identities).
	- Can generate both legitimate and invalid transactions at an infinite rate.
		- The adversary is extremely well-funded (transaction fees are no concern).
	- Can create client-node connections at a rate `ADV_CLIENT_PER_S` to each honest node.
		- Connections from the adversary are indistinguishable from honest client connections until transactions have been validated.
	- Has root access on all dishonest nodes (there is at least one).
		- Can make infinite peer-to-peer connections.
1. Honest users:
	- Can make client-node connections (to submit transctions) at rate `HON_CLIENT_PER_S` to each node in the network.
	- Will always construct tx that pass `nominate_fn` in all honest nodes.
1. Assume that the rate an honest node can evaluate transactions that won't be added to a block is greater than the rate invalid transactions can appear.
	- `HON_VALIDATE_PER_S - (MAX_TRANSACTIONS_PER_BLOCK / mean block time) > HON_CONNECT_PER_S x (ADV_CLIENT_PER_S / (HON_CLIENT_PER_S + ADV_CLIENT_PER_S))`

#### Threat model

1. The adversary CAN:
	- Marginally slow down the rate of slot production by introducing more variations of valid SCP messages than would otherwise exist. In the worst case, slots may be stalled indefinitely if the 'preemption' attack mentioned in the SCP whitepaper is executed successfully.
	- Reduce the maximum average tx throughput of the network to `min((MAX_TRANSACTIONS_PER_BLOCK / mean block time), NUM_HONEST x HON_CONNECT_PER_S x (HON_CLIENT_PER_S / (HON_CLIENT_PER_S + ADV_CLIENT_PER_S)))`.
1. The adversary CANNOT:
	- Cause a node to crash or observe resource exhaustion.
	- Cause the network to become completely unresponsive (i.e. break the SCP 'eventual' liveness guarantee, nor the SCP safety guarantee).

**Note**: If `NUM_HONEST x min(HON_CONNECT_PER_S, HON_CLIENT_PER_S) x (HON_CLIENT_PER_S / (HON_CLIENT_PER_S + ADV_CLIENT_PER_S) x mean block time > MAX_TRANSACTIONS_PER_BLOCK`, then there will be a persistent fee market and valid tx submitted by honest clients will regularly fail.



## Rationale

- No additional rationale at this time.



## Backward compatibility

This proposal only changes resource management within nodes. It has no backward compability concerns.



## Reference implementation

Not completed yet.



## References

- [Stellar Consensus Protocol whitepaper](https://www.stellar.org/papers/stellar-consensus-protocol)
