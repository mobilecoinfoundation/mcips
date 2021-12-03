```
  MCIP: ?
  Layer: Consensus
  Title: Fee Epochs
  Authors: koe <ukoe@protonmail.com>
  Status: Draft (DO NOT USE)
  Type: Standards Track
  Created: ?
  Requires: MCIP-[dual rule blocks]
```

## Abstract

This proposal reduces the frequency of fee output creation by recording block fees in block headers and only creating fee outputs every 1000 blocks.



## Motivation

In the current (v0) protocol, each block has a fee output containing all of that block's transactions' fees. If the number of transactions in a block tends to be low, then the ratio of fee outputs to non-fee outputs in the blockchain can be very high. Most transactions have two outputs (one for the recipient, and one for change), so if a block has one transaction, then the fee output represents one third of the block's outputs.

Not only do fee outputs bloat the blockchain, they also 'pollute' ring signatures. Users cannot easily filter out fee outputs when constructing ring signatures for their transactions, so inevitably their rings will contain many fee outputs if fee outputs represent a large share of all outputs (i.e. given random sampling of outputs from the chain).

There is no real reason for every block to have a fee output. This proposal greatly reduces the frequency of fee output creation.



## Block format

Let the new block header format be as follows. The only difference compared to the MCIP-[dual rule blocks] header format is the addition of two 9-byte `fees` fields. Block IDs should be computed from this new header format in the same fashion as old block IDs.

Fields                 | Description
-----------------------|-----------------
`BlockID`              | This block’s ID
`version`              | This block's protocol rules version
`parent_id`            | Parent block’s ID
`index`                | This block’s index
`cumulative_txo_count` | Total txo count including this block
`root_element`         | Root element of txo set before this block
`fees_old`             | **Total fees paid out by transactions in this block (from old ruleset)**
`fees_new`             | **Total fees paid out by transactions in this block (from new ruleset)**
`contents_hash`        | Hash of block content components (see MCIP-[dual rule blocks])


### New protocol rules

- A block's `fees_xx` field must record the sum of fees paid out by transactions in the block validated by a specific ruleset (see caveat [below](#backward-compatibility)). MCIP-[dual rule blocks] introduced blocks containing transactions validated by up to two rulesets.
	- If one ruleset has no transactions, the corresponding fee field should be `0`.
- Let the `contents_hash` be computed as (modified from MCIP-[dual rule blocks]):
	- Domain separation string: "mc_contents_hash_v3" (or v2 if this MCIP is activated at the same time as MCIP-[dual rule blocks])
```
contents_hash = H(contents_hash_old_rules, contents_hash_new_rules, contents_hash_fee_outputs)
```
- Let `contents_hash_fee_outputs` be computed the same way as other content hashes, except only using fee outputs (see [below](#fee-epoch)). If there are no fee outputs, set it to `0`.
- When appending outputs to the store of on-chain outputs, fee outputs should have higher indices than all other outputs from a block. Fee outputs should be distinguishable from other outputs so `contents_hash` can be reconstructed properly, and so observers can recreate them for auditing purposes.


### Block signatures

In MCIP-[dual rule blocks], enclaves sign block contents relevant to the transactions they validate. Since fee amounts will be stored outside of enclaves instead of converted directly to fee outputs, the fees produced by an enclave must be signed by that enclave.

The message signed by new enclaves should be as follows (modified from MCIP-[dual rule blocks]).

```
msg = H(enclave_version, parent_id, index, root_element, fees, contents_hash_this)
```

- `H()`: Hash function (digest32 from MerlinTranscript) [[[TODO]]]
	- Domain separation string: "mc_block_signature_v2"
		- **Note**: The current block signature message is not domain separated, which seems to be an oversight.
- `enclave_version`: This enclave's version (corresponds 1:1 with MRENCLAVE value)
- `parent_id`: Parent block’s ID
- `index`: This block’s index
- `root_element`: Root element of txo set before this block
- `fees`: Fees provided by transactions validated by this enclave
- `contents_hash_this`: Hash of block contents (txos and key images) validated by this enclave



## Fee epoch

A fee epoch is the 'number of blocks' that fees should accumulate before making a fee output.
	- Let the fee epoch be 1000 blocks in this proposal (future MCIPs may change it by 'superceding' this MCIP).

When constructing a block, if the block's index is a multiple of the current fee epoch (i.e. `index % fee_epoch == 0`), then create fee outputs by summing up the prior epoch's fees (including the current block's fees).


### Sum of fees from prior epoch

The fee epoch may change at any time due to a hard fork, so the following algorithm may be useful (in Rust-like pseudocode). Nodes can store a convenience variable `previous_fee_epoch_index` recording the index of the last block where fee outputs were created.

```
fn fees_from_epoch(BlockIndex block_index, Fees current_block_total_fees, ProtocolVersion protocol_version) -> Option<Fees>
{
	if (block_index % get_fee_epoch(protocol_version) == 0)
	{
		assert(previous_fee_epoch_index < block_index);

		return [sum of fees from (previous_fee_epoch_index + 1) up to and including current block (both _old and _new fees in the same sum)];
	}
	else
		return None();
}
```

Rules for `previous_fee_epoch_index`:
- It may only be updated if a block is successfully appended to the blockchain.
	- Its initial value should equal the index of the block just prior to the first block whose header has `fees_xx` fields (i.e. when this MCIP is activated on the network).
- It must be deterministically reproducible when synching a ledger from scratch (i.e. reproducible from knowledge of each protocol version's rules, combined with information stored in block headers).
	- Similarly, a watcher node synching the ledger from scratch should be able to recreate all fee outputs and fully account for all block fees stored in block headers.
- It must be reset correctly if part of the chain is discarded. An easy way to do this is to search back through the blockchain record until you find the most recent block with a fee output (or the variable's initial value, if necessary).


### Fee output construction

If the sum is greater than 8 bytes, then create multiple fee outputs. A representative algorithm follows:

```
fn outputs_from_fees(Fees total_fees, BlockID parent_id, Hash contents_hash_rulesets) -> Vec<Outputs>
{
	int num_outputs{0};
	Fees fee_amount;
	Vec<Outputs> results;

	while (total_fees > 0)
	{
		if (total_fees > [8 bytes max])
			fee_amount = [8 bytes max];
		else
			fee_amount = total_fees;

		results.push_back(make_fee_output(fee_amount, num_outputs, parent_id, contents_hash_rulesets));

		total_fees -= fee_amount;
		num_outputs++;
	}

	return results;
}
```

Define `contents_hash_rulesets = H(contents_hash_old, contents_hash_new)`.
	- `H()`: < hash function > [[[TODO]]]
	- Domain separation string: "mc_combined_contents_hash"

Let the function `make_fee_output()` construct an output, where:
- Recipient address is the MobileCoin Foundation (the same address used in v0).
	- *Public view key*: `5222a1e9ae32d21c23114a5ce6bb39e0cb56aea350d4619d43b1207061b10346`
	- *Public spend key*: `26b507c63124a2f5e940b4fb89e4b2bb0a2078ed0c8e551ad59268b9646ec241`
- The txout private key is defined as:
	- `H(parent_id, fee_output_number, contents_hash_rulesets)`
		- `H()`: hash function [[[TODO]]]
		- Domain separation string: "mc_fees_output_private_key"
	- The `fee_output_number = num_outputs` in the above algorithm.
- The output amount is the fee amount to be stored in the output.
- **Note**: This is equivalent to `mint_aggregate_fee()` used by MobileCoin's reference implementation in v0.

Sort the fee outputs based on their `fee_output_number` before adding them to the blockchain (the lowest number has lowest index) and before computing `contents_hash_fee_outputs`.



## Rationale

#### Why are fees recorded in 9 bytes instead of 8?

A MobileCoin amount is 8 bytes, which can hold 18.4mill MOB, but the maximum supply is 250mill MOB. Since it is at least possible for accumulating fees to exceed 8 bytes (although extremely unlikely in practice), we use 9 bytes for robustness.

#### Why are two fee amounts stored in block headers?

A core part of auditing the MobileCoin blockchain is verifying block signatures, which are produced by SGX secure enclaves. It is important for all non-trivial block information to be signed by enclaves. This means if a block header records fees, then those fee amounts must be traceable back to an enclave, which obtained the fee amounts from transactions it securely validated.

Since MCIP-[dual rule blocks] allows two enclaves to validate transactions in the same block, two fee amounts are necessary so all enclave signatures can be reproduced from blockchain data (block headers and transaction artifacts [txos and key images]).

#### Fee outputs are not currently reproducible by observers, doesn't this proposal reduce the privacy of MobileCoin by allowing outputs to be categorized?

Fee outputs are identifiable in MobileCoin's current protocol by implication. Normal outputs can be identified because their txout public keys are transmitted outside enclaves for duplicate checks. Outputs that aren't normal outputs must be fee outputs.

#### Why does this proposal depend on MCIP-[dual rule blocks]?

MCIP-[dual rule blocks] specifies a new way to define the `contents_hash` field of block headers, which is useful for incorporating fee outputs. To support both that MCIP and this proposal, it is also necessary to add two fee fields to block headers. Depending on that MCIP is more convenient than adding clauses like 'if MCIP-[dual rule blocks] is implemented, then do X'.



## Backward compatibility

If this proposal is active while an old enclave that doesn't support it is being used (e.g. because the active protocol version supports MCIP-[dual rule blocks] and uses two enclaves, and the 'old' enclave doesn't support this proposal), then the corresponding `fees_xx` header field should be set to `0` and the old enclave should create its own fee outputs.

This proposal represents a ledger format change (block header format change, must store fee outputs separately). It must be implemented via a hard fork of some kind.



## Reference implementation

Not completed yet.



## References

- None
