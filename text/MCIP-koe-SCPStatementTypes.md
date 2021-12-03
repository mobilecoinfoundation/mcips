```
  MCIP: ?
  Layer: Peer Services
  Title: SCP Statement Types
  Authors: koe <ukoe@protonmail.com>
  Status: Draft (DO NOT USE)
  Type: Standards Track
  Created: ?
```

## Abstract

This proposal changes the 'statement value' passed between SCP nodes in MobileCoin from 32-byte transaction hashes to 34-byte typed statements. It also lays out a process and timeline for transitioning the existing network between the two value regimes.



## Motivation

Currently, MobileCoin nodes can only consense on transaction hashes (i.e. transaction hashes are the statements voted on during nomination, and ballots are collections of transaction hashes). It is not possible to consense on any other kind of statement, greatly reducing the versatility of slot contents. In the spirit of, and to support use of, MCIP-[Versatile SCP], it is useful to consense on arbitrary statements.



## Statement type specification

- Define the SCP statement value as a 34-byte byte sequence.
- Let bytes [0-31] be the statement payload.
- Let bytes [32-33] be the statement type.
- Statement types can be implemented as enums.
- Let type `0x00` be 'Transaction Hash', defined as (it's the same as old transaction hashes)
```
Tx.digest32::<MerlinTranscript>(b"mobilecoin-tx")
```
- Nodes should ignore unknown statement types (e.g. with MCIP-[Versatile SCP] nomination filtering), rather than considering them invalid.

**Note**: The same statement type 'Transaction Hash' can be used for different transaction versions, since transaction details can be handled independent of transaction hashes.


### SCP considerations

If MCIP-[Versatile SCP] is implemented, then for 'optional ballot creation' at least one `0x00` statement is required to create a ballot.



## Regime transition process

To transition an active network between the statement value regimes, there are two methods. The first method involves restarting all nodes in the network simultaneously, with the new value regime used post-restart. Since a network restart is impractical, a more gradual approach is warranted.

1. **Roll out node implementation update**: The node implementation handles typed statements only. Incoming SCP messages will contain old transaction hashes. These should be converted to 'Transaction Hash' statement types before being consumed by the node implementation. If a 'Transaction Hash' is sent out from a node implementation, it should be converted to an old transaction hash before being transmitted to another node.
1. **New messages contain Transaction Hashes**: New messages created by nodes should contain typed statements (Transaction Hashes). In this phase nodes can hear messages from both value regimes.
1. **Reject old value regime**: Messages containing old transaction hashes should be considered invalid. Nodes can update their node implementation to cut out parts supporting old transaction hashes.


### Transition timeline

[[[TODO: set dates]]]

1. **Roll out**: Date A
1. **Message transition phase**: max(Date B, Date A + 10 blocks)
1. **Finalize transition**: max(Date C, **Message transition phase** + 10 blocks)



## Adding statement types

To add a new statement type, it should be specified in a Peer Services Standards Track MCIP that references this MCIP. New statements types should be registered and tracked in the following table for ease of reference.

New statement types can be rolled out ad hoc. Nodes that care about a new statement type can support it before it starts being seen on the network. Nodes that don't care will ignore those statements when they appear.

The 'active/inactive' status of statement types in this table should be determined by which types the MobileCoin Foundation nodes support (and when they are/were supported).

Type   | Name             | MCIP     | Dates/protocol versions active
-------|------------------|----------|----------------------------------------
`0x00` | Transaction Hash | MCIP-[statement types] | not active



## Rationale

#### Why use a phased rollout?

Since the network is asynchronous, nodes may disagree about the timing of events. A phased rollout lets nodes have large disagreements on timing without causing problems for the network.

#### Why not use a hard fork to implement this proposal?

MCIP-[SCP Based Hard Forks] is a Process MCIP for hard forks that depends on this proposal. Ideally, this proposal would be implemented using the transition process defined here, then all new hard forks could use that hard fork MCIP.



## Backward compatibility

The transition process explicitly phases out the old SCP statement regime. After this proposal is Active (when all MobileCoin Foundation nodes have reached phase 3 of rollout), nodes using the old and new regimes will not be able to talk to each other.



## Reference implementation

Not completed yet.



## References

- None
