- Feature Name: sorted_transaction_outputs
- Start Date: 2022-04-04
- MCIP PR: [mobilecoinfoundation/mcips#34](https://github.com/mobilecoinfoundation/mcips/pull/34)
- Tracking Issue: original PR: https://github.com/mobilecoinfoundation/mobilecoin/pull/561

# Summary
[summary]: #summary

For blocks with a version >= 3, all the transaction's outputs MUST be sorted by the public key value.

# Motivation
[motivation]: #motivation

The main idea behind this feature is to prevent establishing the order of outputs that were added to the 
transaction, thus making hard or impossible to identify the source.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The enclave transaction validation will now check if the TxOuts are sorted.
It will return a transaction validation error if not.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

Before block version 3, the transaction validator was checking on the following checks
* number of inputs
* number of outputs
* ring sizes
* uniqueness of ring elements
* sorted order of the ring elements
* sorted order of inputs
* membership proof
* signature
* transaction fee
* uniqueness of key images
* uniqueness of outputs public keys
* tombstone

With block version >= 3 the validator will also check whether the transaction outputs are sorted. 

When library is compiled with `test-only` feature flag, there is a possibility to provide a custom trait for the transaction
builder to be used for sorting the transaction outputs.

# Drawbacks
[drawbacks]: #drawbacks

There are no drawbacks.

