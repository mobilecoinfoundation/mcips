```
  MCIP: ?
  Title: SCP-Based Hard Forks
  Authors: koe <ukoe@protonmail.com>
  Status: Draft (DO NOT USE)
  Type: Process
  Created: ?
  Requires: [Versatile SCP], [protocol version statement type]
```

## Abstract

MobileCoin is a blockchain-based cryptocurrency where nodes execute the Stellar Consensus Protocol (SCP) in collaboration with other nodes in the network to construct the blockchain.

If a change in consensus rules, which are the rules nodes use to assess potential blocks, is desired, then nodes in the network must decide *when* to start applying the new rules (an intentional change in consensus rules is called a 'scheduled hard fork'). If nodes don't transition at the same moment, then the network can diverge or get stuck as different rules are applied to the same blocks in the 'transition zone'.

This proposal describes a process for the network to execute a scheduled hard fork. Using MCIP-[versatile SCP] nomination filtering, the MobileCoin network may consense on new protocol versions while constructing blocks. If sufficient nodes vote for a new version, then the network will 'externalize' the new version as part of a block, and the network will begin using the new protocol rules in the next block.



## Motivation

At the time of writing this, the process for executing a MobileCoin scheduled hard fork is an 'actively coordinated network restart'. In that process, a fork coordinator defines an approximate time when the fork will occur, opens a communication channel with all node operators in advance of the fork, defines a fork height ad hoc when all node operators are prepared, and finally the node operators manually deactivate their nodes (shutting down the entire network temporarily) then restart them with new node software (which applies the new consensus rules).

While such an approach may be feasible when the number of nodes is small and coordination costs are low, as the number of nodes rises it will become more difficult to safely restart the network without leaving some nodes behind. It also forces the network to go offline temporarily and places a high burden of trust on the transition coordinator.

This proposal is infinitely scalable in the number of nodes, never requires a network restart (unless there is a concurrent network failure, or a proposed hard fork is so flawed that large numbers of nodes refuse to support it, causing the network to get stuck), and does not require a transition coordinator.



## Scheduled hard forks

### Defining a hard fork schedule

To schedule a hard fork, create a Process MCIP with the following details.

1. Define a sequence of version changes (the simplest case is one version change). Version number changes can only increase, so the first new version in a sequence must be greater than the version in blocks being actively created.
	- A Process MCIP based on this MCIP may not specify new version numbers less than or equal to version numbers specified by other Process MCIPs using this MCIP that have reached Proposed or Active status, even if those MCIPs failed (i.e. the hard fork was not actually executed). Without this constraint, there is a risk that nodes with a ruleset implementation for an 'old' MCIP will use that implementation if a 'new' MCIP is activated, if the two MCIPs specify the same version number.
1. For each version change, write a summary of new rules for transaction validation and block construction that will be applied when that version number is active (i.e. being recorded in blocks). This summary should include references to MCIPs and/or reference implementations for all the new rules. Independent node implementations that use those references to update their software should be interoperable after the new rules go into effect.
	- All referenced MCIPs must have 'Proposed' status before a hard fork MCIP can have 'Proposed' status.
1. Define a sequence of increasing activation dates, one for each version change.


### Hard fork process

To execute a hard fork, use the following process.

1. Create a Process MCIP (see previous section) for a hard fork sequence. It must reach 'Proposed' status before the next step can be done.
1. In the period before the first activation date, nodes prepare implementations supporting new rules.
1. Node operators locally define their 'preference' for each new version number (i.e. a vote for yes/no). This will be used in MCIP-[Versatile SCP] nomination filtering. Even if their preference is 'no' for a given version number, their node implementations should still support its new rules.
1. After the activation date for a new version number passes, nodes may issue nomination votes for MCIP-[SCP statement types] `0x01` (Protocol Version) statement types containing the new version number.
1. If node A sees a vote to nominate (or accept nominate) a `0x01` statement from node B (or it is proposed by the local implementation), it should use MCIP-[Versatile SCP] nomination filtering to decide whether to vote to nominate that statement. The 'test condition' is a comparison between a local clock and the scheduled activation date for that version number (assuming it is scheduled at all).
	- If a sequence of version numbers is scheduled, then MCIP-[versatile SCP] nomination filtering should only allow a version statement to be voted on if all prior version numbers in the sequence have been externalized (added to blocks). If a version number in a sequence fails due to too many nodes dissenting, then for the next version number in the sequence to be voted on, node operators must manually remove the failed version number from their hard fork sequence. This may mean the series of version numbers in the blockchain will contain 'gaps' corresponding to failed version numbers.
1. If a version number statement gets externalized as part of a slot, it should be recorded in the version field of the block created from that slot.
1. **Protocol Rule**: All new protocol rules, and any Peer Services transitions piggybacking on those rules, will only go into effect *after* the first block with a new version number has been externalized.



## Rationale

#### How is this different from protocol upgrades in the Stellar Network?

The Stellar Network is another blockchain technology that uses an implementation of the Stellar Consensus Protocol. Stellar has a scheduled hardfork procedure which they call 'protocol upgrades'. To upgrade Stellar's protocol, nodes may pass protocol upgrade statements through SCP, much like the hardfork statements discussed in this document. The critical difference, however, is Stellar [does not use nomination filtering](https://github.com/stellar/stellar-core/blob/e1acce46eace5db4547adeb2560f6089647d0ed3/docs/versioning.md).

If a Stellar node believes a protocol upgrade statement is invalid (e.g. it has appeared too soon), they will completely drop it from their slot (instead of allowing a v-blocking set to change their mind). This effectively means nodes who agree and disagree about a protocol upgrade, which they all concur is scheduled but dissent on the timing, will consider each other malicious (Byzantine). Once a protocol upgrade makes it through SCP, all nodes that dissented and hence did not participate in consensuating the relevant slot must re-synchronize with the network after the fact. Nomination filtering allows nodes to disagree about the timing of a scheduled hardfork without affecting the number of nodes that may participate in creating each block.

#### Why not use a pre-defined block height to coordinate protocol transitions?

This is the approach taken by [Monero](https://web.getmonero.org/library/Zero-to-Monero-2-0-0.pdf), which has successfully undergone at least [14 scheduled hard forks](https://github.com/monero-project/monero/blob/master/README.md) as of writing this. However, Monero is a Nakamoto-based cryptocurrency where the rate of block production is carefully controlled, whereas in SCP-based blockchains such as MobileCoin, blocks are created ad hoc. This mean a pre-defined block height in MobileCoin could appear this afternoon, or a year from now, depending how frequently transactions are submitted to the network.

Moreover, imagine only half of the network is willing to execute a hard fork. At the transition point, half of nodes will begin using new rules, while the other half retains the old rules. At that point the network will get stuck, unable to make blocks. If each half of the network reconfigures their quorum sets to get unstuck, then the network will diverge as each half works on a different chain (using different rules). The threat of divergence may force dissenters to support a hard fork even if they represent a substantial portion of the network, which puts a lot of power in the hands of those who propose hard forks (e.g. probably the MobileCoin Foundation in most or all cases).

This proposal allows the timing of transition points to be coordinated accurately and prevents the network from getting stuck if there is not sufficient agreement about a hard fork.

This proposal also encourages a certain negotiation context in the network. Nodes are empowered to safely dissent from hard forks, with the implicit assumption that if enough of the network decides to go through with a hard fork, that dissenters will go along with it. Nodes only need to risk network divergence (by considering new hard fork versions invalid) if they believe a hard fork is seriously flawed (this may include a bug in a proposed hard fork - better for the network to get stuck than create insecure/broken blocks).

#### Why is the first block with a new protocol version number based on the previous version's rules? It is confusing.

This is necessary since in this proposal new block versions are externalized alongside block contents created using old rules. Importantly, it is possible to change block header formats without any difficulties, because the format change goes into effect after the version number has been recorded in the chain.



## Backward compatibility

Assuming MCIP-[Versatile SCP] and MCIP-[protocol version statement type] are already Active, a hard fork using this MCIP can be implemented without requiring any special techniques for compatibility with old block formats or protocol rules. Block headers already contain a protocol version number field.



## Reference implementation

N/A



## References

- None
