```
  MCIP: ?
  Layer: Consensus
  Title: Dual-Rule Blocks
  Authors: koe <ukoe@protonmail.com>
  Status: Draft (DO NOT USE)
  Type: Standards Track
  Created: ?
```

## Abstract

This is a proposal to change MobileCoin's block format so a block can contain transaction artifacts from transactions validated against two sets of protocol rules. These transaction artifacts are segregated into ruleset-specific groups, to facilitate compatibility with validating transactions in secure enclaves, a process that does not allow rulesets to be easily merged together.

This proposal also introduces protocol rules for supporting dual-rule blocks in MobileCoin.



## Motivation

To change MobileCoin's protocol rules (the rules by which transactions are validated, and blocks constructed), a hardfork is necessary. A hardfork is an event where the network's nodes begin using new protocol rules to validate new transactions, and to make new blocks.

Hardfork transitions occur at a point in time that may or may not be accurately predictable in advance, so users may be inconvenienced. When a user constructs a transaction, if the user doesn't know exactly what rules will be in effect when the transaction is assessed by the network, they may create a transaction that gets rejected unexpectedly.

Wallet developers are also burdened, since if they want their users to have a good experience, it may be necessary to support multiple sets of protocol rules at once. Otherwise, if they only support one set of rules at a time, then users would need to coordinate their software updates with each hardfork transition point.

We can avoid all those coordination problems by allowing a grace period in hardfork transitions where two sets of protocol rules are active in parallel (implementing those transitions is not the subject of this proposal). However, MobileCoin transactions are currently validated in secure enclaves.

When a user submits a transaction to the network, they submit it in encrypted form to an SGX secure enclave running a specific set of protocol rules. This secure enclave can send copies of the transaction to other secure enclaves in the network running the same set of protocol rules. The system is designed so only enclaves with the same exact software can ever see the user's transaction.

Therefore, to support an old and new set of protocol rules, it is necessary to support two enclaves validating transactions in parallel. A new enclave with 'merged' rules would not work because users would have to update their software to correctly send transactions to the new enclave.

Since enclaves sign the transaction artifacts (e.g. outputs and key images) from transactions that they validate (allowing block contents to be audited by observers, to some extent), signatures from multiple enclaves can only be verified if the signature contents are recoverable. To that end, in the blockchain, transaction artifacts must be segregated based on which enclave validated them.

This proposal describes how to segregate transaction artifacts, and how to safely construct a block that contains artifacts validated by two separate protocol rulesets.



## Block format

This proposal does not change the format of block headers, but does slightly change how they are constructed. Specifically, instead of computing `contents_hash` directly from block contents (txos and key images, currently), it is computed from a composition of parts.

Fields                 | Description
-----------------------|-----------------
`BlockID`              | This block's ID
`version`              | This block's protocol rules version
`parent_id`            | Parent block's ID
`index`                | This block's index
`cumulative_txo_count` | Total txo count including this block
`root_element`         | Root element of txo set before this block
`contents_hash`        | **Hash of block content components**

The `contents_hash` should be computed as follows.
```
contents_hash = H(contents_hash_old_rules, contents_hash_new_rules)
```

- `H()`: hash function [[[TODO]]]
	- Domain separation string: "mc_contents_hash_v2"
- `contents_hash_old_rules`: Hash of block contents (txos and key images) validated by old ruleset.
- `contents_hash_new_rules`: Hash of block contents (txos and key images) validated by new ruleset.

If an active protocol version only permits one enclave to validate transactions, then `contents_hash_old_rules` should be set to `0`, and `contents_hash_new_rules` should represent transactions validated by that enclave.



## Supporting dual-rule blocks

In the current (v0) protocol, all transaction rules are implemented in SGX secure enclaves. This protocol allows two enclaves to validate transactions in parallel.


### Block signatures

Currently (v0), enclaves sign block headers (a block header represents the entire block). If a block has transaction artifacts from two enclaves, it doesn't make sense for an enclave to sign artifacts it didn't help validate. Instead, enclaves should sign only their own artifacts.

The message signed by new enclaves should be as follows. We also introduce an 'enclave version' number for tracking different enclaves.

```
msg = H(enclave_version, parent_id, index, root_element, contents_hash_this)
```

- `H()`: Hash function (digest32 from MerlinTranscript) [[[TODO]]]
	- Domain separation string: "mc_block_signature_v2"
		- **Note**: The current block signature message is not domain separated, which seems to be an oversight.
- `enclave_version`: This enclave's version (corresponds 1:1 with MRENCLAVE value)
- `parent_id`: Parent block's ID
- `index`: This block's index
- `root_element`: Root element of txo set before this block
- `contents_hash_this`: Hash of block contents (txos and key images) validated by this enclave


### Block creation

Currently (v0), enclaves create blocks and define the `cumulative_txo_count`. Instead, blocks should be constructed outside of enclaves, and `cumulative_txo_count` defined outside enclaves.


### Fee outputs

Currently (v0), enclaves create a fee output for every block. This behavior should remain unchanged in this proposal (a different proposal may change how fee outputs are created). If two enclaves process transactions for a block, then there will be two fee outputs.


### Transactions in the network

Currently (v0), when a node receives transactions from clients or the network, they are passed to a single enclave for validation. In this proposal, a node must operate two enclaves concurrently, and route transactions to the appropriate enclave for validation.

- **Note**: Transactions are validated with respect to the existing blockchain, so if there are two concurrent validator enclaves, if both are given access to the same information (the same blockchain record), then this proposal causes no conceptual problem for consensus.


### Block construction

To make sure the transaction artifacts produced by each enclave coexist properly (e.g. transaction outputs and key images), a careful block construction procedure is required. These rules are applied during the 'externalize' part of SCP, when a set of transactions (which may be validated by one or two enclaves/rulesets) are processed into a block for addition to the blockchain.

1. Verify the transactions contain no duplicate transaction hashes, key images, or txout public keys.
1. Sort the transactions into groups based on what enclave/ruleset must be used to validate them. Validate all transaction groups and sign them (e.g. with the enclave signature scheme [above](#enclave-signatures)).
	- The active MobileCoin protocol version from the previous block will define which enclaves/rulesets may be used to validate transactions for a given block.
	- If there are no transactions to pass to an enclave, then do not call into that enclave. The corresponding `contents_hash_xx` should be set to `0`.
1. If there is more than one enclave signature, verify the signed messages contain the same parent ID, block index, and root element.
1. Construct the block header and publish the block.

- **Note**: Block contents validated by different rulesets should be stored separately in the blockchain database, to facilitate recreating `contents_hash` fields and block signatures properly.
- **Protocol Rule**: When appending outputs to the store of on-chain outputs, the outputs from lower-versioned enclaves/rulesets should have lower indices than higher-versioned enclaves/rulesets.



## Rationale

None at this time.



## Backward compatibility
 
1. In the current (v0) protocol, enclaves create blocks and sign block headers directly. If a dual-rule block wants to include an old enclave that doesn't support this MCIP (e.g. to populate the `contents_hash_old_rules` field), then that enclave should sign a 'modified block header', as follows (differences from this proposal's block header are bolded).

Fields                 | Description
-----------------------|-----------------
`BlockID`              | This block's ID
`version`              | **The protocol version when this enclave was introduced**
`parent_id`            | Parent block's ID
`index`                | This block's index
`cumulative_txo_count` | Total txo count including **outputs added by this enclave**
`root_element`         | Root element of txo set before this block
`contents_hash`        | **Hash of block contents (txos and key images) validated by this enclave**

The 'modified block header' is only a semantic for interpretting blockchain contents and block signatures. It does not require any implementation changes.

2. This proposal represents a ledger format change (must store old/new ruleset tx artifacts separately). It must be implemented via a hard fork of some kind.



## Reference implementation

Not completed yet.



## References

- None
