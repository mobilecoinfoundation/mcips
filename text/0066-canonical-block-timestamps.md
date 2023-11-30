- Feature Name: canonical-block-timestamps
- Start Date: 2023-11-30
- MCIP PR: [mobilecoinfoundation/mcips#0066](https://github.com/mobilecoinfoundation/mcips/pull/0066)
- Tracking Issue: [mobilecoinfoundation/mobilecoin#2813](https://github.com/mobilecoinfoundation/mobilecoin/issues/1617)

# Summary
[summary]: #summary

The MobileCoin blockchain is a record of currency events, but timestamps
indicating when those events occurred are currently not recorded in the chain.
This is a proposal to add a _canonical_ timestamp field to each block.

# Motivation
[motivation]: #motivation

Currently, each node in the MobileCoin network records their own local timestamp
for each block that is created. For observers to gain confidence about the
timing of a transaction event, they must obtain timestamps from many nodes and
use statistical analysis to decide when the event occurred. This is inefficient
and can lead to disagreements between users about the timing of events.

Adding a canonical timestamp to blocks allows users to easily see the timing of
events. Users who require more control and reliability over event timings can
operate their own node and record their own timestamps, or use the old method of
asking many nodes their opinions about when an event occurred.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Blocks contain a dedicated timestamp field.  Each block's timestamp is greater
than the previous block's timestamp. To prevent being limited to one block a
second, the timestamp is in milliseconds since Unix epoch (UTC). 

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The block structure includes a `timestamp` field. The timestamp is a u64
representing milliseconds since Unix epoch (UTC). This timestamp is included in
the computation of the block's `id`. 

Block versions 3 and below have no timestamp field. When these earlier versions
are used in block version 4 or higher code:
- the timestamp will be defaulted to 0. 
- the timestamp field will **not** be used in computing the block's `id`.

## Block format

Fields                 | Description
-----------------------|-----------------
`id`                   | This block’s ID
`version`              | This block's protocol rules version
`parent_id`            | Parent block’s ID
`index`                | This block’s index
`cumulative_txo_count` | Total txo count including this block
`root_element`         | Root element of txo set before this block
`contents_hash`        | Hash of block contents (txos and key images)
`timestamp`            | Approximate time this block was created (UTC)


# Drawbacks
[drawbacks]: #drawbacks

- Adding a timestamp field to the block requires a block version change.
- Using milliseconds for the timestamp deviates from block sigantures which use
  seconds. Looking back in the first 2 million blocks of the blockchain we've
  had about 28,000 blocks with the same signed at timestamp.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

None at this time.

# Prior art
[prior-art]: #prior-art

- Ethereum has block timestamps. They are Unix timestamp as 256 bit value in seconds.
- Bitcoin has block timestamps. They are Unix timestamps as a 32 bit value in
  seconds. They will overflow in 2106.
- Much of this MCIP was taken from [MCIP 11](https://github.com/mobilecoinfoundation/mcips/pull/11).

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- How the timestamp is populated is not the subject of this MCIP.

# Future possibilities
[future-possibilities]: #future-possibilities

None at this time.
