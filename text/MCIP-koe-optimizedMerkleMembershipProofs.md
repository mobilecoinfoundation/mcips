```
  MCIP: ?
  Layer: Consensus
  Title: Optimized Merkle Membership Proofs
  Authors: koe <ukoe@protonmail.com>
  Status: Draft (DO NOT USE)
  Type: Standards Track
  Created: ?
```

## Abstract

Reduce the size of MobileCoin membership proofs by replacing 'ranges' with a set of bit flags recording which nodes are on the left/right side of each branch intersection. Further reduce the size of transactions when stored by validator nodes by requiring one 'reference' membership proof per transaction, instead of one per ring member per input per transaction.



## Motivation

One factor in MobileCoin network performance is the size of transactions, both as broadcast client-to-peer and peer-to-peer, and as stored in nodes while waiting to be added to a block. Reducing those sizes as much as possible can improve the maximum transaction through-put of the system, and reduce the resources required to handle any given transaction.

Currently (protocol v0), Merkle membership proofs constitute the bulk of transaction size. A 1-input/1-output transaction is \~21 kB, with membership proofs taking up \~17 kB (assuming membership proofs with 32 elements, representing a Merkle tree 31 layers deep). In a 16-input/16-output transaction (the maximum size of a transaction), membership proofs occupy \~275 kB out of \~325 kB.

When a node stores a transaction, it also stores 'reference membership proofs' for each ring member in each input (used to verify that the ring member's membership proof is legitimate). This doubles the storage required for membership proofs, so a 1-input/1-output tx will require \~38 kB to store, and a 16-input/16-output will require \~600 kB.

This proposal reduces storage requirements to:

- **membership proofs**: \~1 kB per proof (down from \~1.6 kB); \~11.5 kB per input with 11 ring members
- **1-input/1-output**: \~15.5 kB per transaction; \~16.5 kB per transaction stored by a node
- **16-input/16-output**: \~235 kB per transaction; \~236 kB per transaction stored by a node



## Left/right node assignments

Assume a Merkle membership proof is a sequence of node hashes. The first node is hashed with the second, and the result of that hash is hashed with the third node, etc. When only one hash remains, that is the root element of the proof.

Adjacent nodes/hashes are not always hashed together in the same order. The node that represents 'lower' elements in the Merkle tree is canonically on the 'left' side of the hash, while the one representing 'higher' elements is on the 'right' side.

Currently (protocol v0), left/right assignments are determined by 'ranges'. Each element in the proof has an attached 'range' indicating what leaf-node-layer elements it represents. Comparing ranges between adjacent elements indicates which should be on the left or right side.


### Assignments with bit flags

Instead of using 'ranges', record the left/right assignment of all nodes in a Merkle membership proof as bit flags in an 8-byte variable `node_assignment_flags`.

- The first node in a proof does not have a left/right assignment (it is inferred from the assignment of the second node).
- A bit flag value of `0x00` means a node is on the 'left' side when hashed with the node/hash below it (after all lower nodes have been hashed together), while `0x01` means it is on the 'right' side.
- The least-significant bit of `node_assignment_flags` records the left/right assignment of the second proof node. The next least-significant bit records the third proof node's assignment, and so on.

**Note**: Replacing 'ranges' with big flags requires some minor adjustments to the root element computation's implementation, which must be constant-time.

#### Transaction structure

A `node_assignment_flags` field must be added to all membership proofs. Add it to the `TxOutMembershipProof` struct as follows.

Fields                  | Description
------------------------|-----------------
`index`                 | The index (in the proof's data set) of the element being proven to exist in that data set.
`node_assignment_flags` | **Left/right assignment flags for proof elements**
`elements`              | Proof elements

**Note**: The `highest_index` is removed from this struct [below](#reference-proofs).


### Simulated proofs

A Merkle membership proof in MobileCoin is a proof that the element at `index` existed in an append-only data set whose highest element at that time was at `highest_index`. This proof can be converted to a 'simulated proof' for the data set that existed when the `index` element was the highest member (i.e. you can obtain the root element of that old data set via the simulated proof).

A simulated data set may have a smaller Merkle tree than the original proof's data set, meaning the simulated proof has fewer elements. To simulate a proof after this proposal (where 'ranges' can't be used to discern which nodes to include in the simulated proof), use the following procedure.

1. Let `simulated_proof_size = log2([the next power-of-2 above 'index']) + 1`.
2. Let the first element of the simulated proof equal the first element of the original proof.
3. Iterate through the remaining elements of the original proof until the simulated proof has `simulated_proof_size` elements.
    1. If the element is on the 'left' side, then append it to the simulated proof.
    1. If the element is on the 'right' side, then append a 'nil' node to the simulated proof.
    1. Copy the relevant left/right assignment bit flag from the old proof to the new proof's `node_assignment_flags` variable.



## Reference proofs

Currently (protocol v0), if a node wants to validate the membership proofs for each ring member in a transaction, then it needs to obtain a separate 'reference' membership proof for each of those proofs. This reference proof sets `index` equal to the proof-to-be-validated's `highest_index`. Simulating that reference proof should give a simulated proof whose root element equals the proof-to-be-validated's root element.

A separate reference proof is required for each ring member because the protocol allows each ring member's membership proof to have a different `highest_index` value. Instead, add the following rule.

- **Rule**: All membership proofs in a transaction must have the same `highest_index`.

With this rule, nodes only need to (per transaction): obtain one reference proof, construct one simulated proof, and compute one root element.

**Note**: A side consequence of this rule is that `tx_is_well_formed()` (in the current reference node implementation) does not need to check that all reference proofs for a transaction have the same root element.

#### Transaction structure

Since only one `highest_index` value is needed per transaction, only one such value should be stored in each transaction.

- Remove the `highest_index` field from the `TxOutMembershipProof` struct.
- Add a `highest_index` field to the `TxPrefix` struct, as follows.

Fields                 | Description
-----------------------|-----------------
`inputs`               | This transaction's inputs (`vec<TxIn>`)
`outputs`              | This transaction's outputs (`vec<TxOut>`)
`highest_index`        | **The highest index of the data set used to compute all membership proofs in this transaction's inputs.**
`fee`                  | The fee provided by this transaction
`tombstone_block`      | The tombstone block of this transaction



## Rationale

#### What are some optimizations left out of this proposal?

- Instead of recording left/right node assignments in a separate `node_assignment_flags` variable, the most-significant-bit of each node's hash could store that information.
    - This would require a reset of the Merkle tree in the ledger, which would prevent/make-very-difficult an MCIP-[dual rule] + MCIP-[scp hard forks] hard fork.
    - Combining left/right node assignments with how nodes are constructed increases the complexity of membership proofs.
    - The relative reduction in transaction size would be less than 1%, an amount too negligible to justify the above costs.
- Left/right node assignments could be inferred from the proof's element index and highest index, rather than stored in a bit flag.
    - Computing root hashes in constant-time would be more difficult, because inferring left/right assignments is not purely trivial.
    - The relative reduction in transaction size would be less than 1%, an amount too negligible to justify the above cost.
- Don't record the first leaf node hash, since it can be recomputed from the corresponding output.
    - This would be inconvient to implement, and would only reduce proof sizes by \~3%. Transactions are not stored in the blockchain, so reductions of this magnitude are not meaningful.
- Reduce node hashes from 32 to 16 bytes.
    - **Bad idea!** Collision-resistance for a 16-byte hash is only at a 64-bit security level due to the birthday problem, and 64-bit problems are considered computationally feasible. If node hashes were 16-bytes, an attacker could mint coins with the following collision-based attack. Let the attacker try to find a certain kind of leaf node collision. They randomly generate outputs whose amounts are split 50/50 between `2^0` and `2^64 - 1`. A collision between outputs with different amounts should appear in `<< 2^128` operations. After a useful collision is obtained, the attacker can add the minimum-amount output to the blockchain as part of a transaction, then 'spend' it by instead spending the maximum-amount output. The associated membership proof will not reveal that you are spending an output that isn't on-chain, because the relevant leaf node represents both the real and malicious outputs! This attack works because validator enclaves cannot access the blockchain directly to check if a referenced output matches a real on-chain output.

#### Why is `node_assignment_flags` 8 bytes?

- A 4-byte flag variable would permit Merkle proofs with up to 32 nodes, which can represent a Merkle tree with up to 4.3 billion elements. It is easy to imagine the MobileCoin blockchain containing more than 4.3 billion outputs in the long run, so 4 bytes is insufficient.
- The next highest standard data size above 4 bytes is 8 bytes. An 8-byte flag variable would permit Merkle proofs with up to 64 nodes, which can represent a Merkle tree with up to 1.8e19 elements. It is not reasonable to expect the MobileCoin blockchain will ever store that many outputs, which would require around 4 Zettabytes of storage (i.e. 4 million Petabytes, assuming \~200-byte outputs).

#### What are the consequences of using one `highest_index` for all inputs?

- It is not feasible to cache membership proofs before constructing a transaction. All membership proofs must be obtained at the same time with respect to the same ledger state. This may or may not put limitations on the performance of wallets and the Fog service.

#### What are the advantages of using one `highest_index` for all inputs?

- Transactions with only have to store one `highest_index` 8-byte value, instead of one per ring member per input. Since each ring member's membership proof is on the order of 1 kB, this only reduces each proof's size by \~1%.
- Validator enclaves only have to obtain one reference membership proof from the untrusted ledger per transaction, instead of one per ring member per input per transaction. This reduces the size of stored transactions (i.e. those waiting to be added to a block) by around 40-45%, and marginally improves validation times for transactions with many inputs by reducing how many root elements need to be computed from simulation proofs.



## Backward compatibility

Since the proposal requires transaction format changes and a validation rule change, it must be implemented via a hard fork of some kind.



## Reference implementation

Not completed yet.



## References

- None
