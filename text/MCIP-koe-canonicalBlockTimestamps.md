```
  MCIP: ?
  Layer: Consensus
  Title: Canonical Block Timestamps
  Authors: koe <ukoe@protonmail.com>
  Status: Draft (DO NOT USE)
  Type: Standards Track
  Created: ?
```

## Abstract

The MobileCoin blockchain is a record of currency events, but timestamps indicating when those events occurred are currently not recorded in the chain. This is a proposal to add a 'canonical' timestamp field to each block. The process for populating that field is not the subject of this MCIP.



## Motivation

Currently, each node in the MobileCoin network records their own local timestamp for each block that is created. For observers to gain confidence about the timing of a transaction event, they must obtain timestamps from many nodes and use statistical analysis decide when the event occurred. This is inefficient and can lead to disagreements between users about the timing of events.

Adding a canonical timestamp to blocks allows users to easily see the timing of events. Users who require more control and reliability over event timings can operate their own node and record their own timestamps, or use the old method of asking many nodes their opinions about when an event occurred.



## Block format

Let the new block format be as follows. The only difference compared to the current (v0) block format is the addition of an 8-byte `timestamp` field. Block IDs should be computed from this new header format in the same fashion as old block IDs.

Fields                 | Description
-----------------------|-----------------
`BlockID`              | This block’s ID
`version`              | This block's protocol rules version
`parent_id`            | Parent block’s ID
`index`                | This block’s index
`timestamp`            | **Approximate time this block was created (UTC)**
`cumulative_txo_count` | Total txo count including this block
`root_element`         | Root element of txo set before this block
`contents_hash`        | Hash of block contents (txos and key images)

A timestamp should be recorded in Epoch/Unix time.

- **Protocol Rule**: A block's timestamp must be greater than the previous block's timestamp.


### Block signatures

Currently (v0), enclaves sign the entire block header for each block. New enclaves should also sign the entire block header (including timestamps), unless MCIP-[dual rule blocks] is implemented. If so, new enclaves should not sign timestamps.



## Rationale

None at this time.



## Backward compatibility

If MCIP-[dual rule blocks] is implemented, a dual-rule block is active (with two enclaves), and the 'old' enclave does not support this proposal, then block signatures from the old enclave will be on the 'old' block header format. This is not a problem for consensus, but must be supported by node software.

This proposal represents a ledger format change (block header format change). It must be implemented via a hard fork of some kind.



## Reference implementation

Not completed yet.



## References

- [Stellar Consensus Protocol whitepaper](https://www.stellar.org/papers/stellar-consensus-protocol)
