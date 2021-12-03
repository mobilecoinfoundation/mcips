```
  MCIP: ?
  Layer: Consensus
  Title: View Tags
  Authors: koe <ukoe@protonmail.com>
  Status: Draft (DO NOT USE)
  Type: Standards Track
  Created: ?
```

## Abstract

Increase the per-output speed of view-key-scanning by adding a 1-byte 'view tag' to output data.



## Motivation

The rate that users can view-key-scan sets of outputs for owned outputs is bottlenecked on per-output elliptic curve operations. Reducing the average number of operations to assess unowned outputs will increase average scan speeds (especially since most outputs that exist are unowned).

View-key-scanning is a contributing factor to the maximum transaction throughput of the MobileCoin network. If outputs are added to the chain faster than an average/low-end user can scan them, then MobileCoin would become unusable for a non-trivial portion of the userbase.



## View tags

### New output field

Outputs should contain a new 1-byte field (i.e. the `TxOut` data structure in the MobileCoin reference implementation).

```
uint8 view_tag
```


### `view_tag` construction

When constructing an output for a transaction:

1. Define the sender-receiver Diffie-Hellman derivation (unchanged).

```
derivation = r Kv
```

2. Define the view tag as a domain-separated hash of the derivation, truncated to 1 byte.

```
view_tag = H("view_tag", derivation).truncate(1)
```


### Output scanning

When an output is encountered and the user wants to know if they own it:

1. Compute the nominal sender-receiver DH derivation from the txout public key (unchanged).

```
derivation_nom = kv * output.public_key
```

1. Compute the nominal view tag.

```
view_tag_nom = H("view_tag", derivation_nom).truncate(1)
```

1. Use the view tag to branch the execution flow.

```
if (view_tag_nom == output.view_tag)
    // continue with normal view-key scanning (can re-use 'derivation_nom' here instead of recomputing it)
else
    // ignore this output
```



## Rationale

#### What is the practical benefit of view tags?

View tags allow a user to skip the computation of `spend_key_nominal = output.target_key - shared_secret*G` for all outputs that fail the view tag test. Avoiding these elliptic curve operations contributes the bulk of time savings.

[One estimate](https://github.com/monero-project/research-lab/issues/73#issuecomment-634915828) showed a 25% increase in scan speed due to view tags.

#### Why are view tags 1 byte instead of e.g. 4 bits or 2 bytes?

- Full bytes are a standard unit of measurement in the digital world. Partial-byte data types would probably end up being serialized in full bytes, so there is no benefit to using them.

- The marginal gain in scan speed when going from 1-byte to 2-byte view tags is, relative to the maximum possible gain, (1/256) - (1/65536) = 0.4%. This is too insignificant to justify using more bytes.

#### Why are view tags domain-separated? Why not just truncate the normal sender-receiver secret `H(r Kv)`?

Leaking bits of the shared secret violates best practice for cryptographic protocol design.

#### How can you do constant-time output scanning in the context of this proposal?

One approach would be collecting large sets of outputs before scanning them. If the number of owned outputs is not a significant proportion of the set to be scanned, then timing analysis will not reveal anything.



## Backward compatibility

This proposal represents a ledger format change (new output format) and new transaction construction rules. It must be implemented via a hard fork of some kind.

A user can assume they own all outputs obtained from the Fog service, so view tags don't need to be returned to users by Fog. This means no changes to the Fog `TxOutRecord` struct are required. To compensate for the changed output format, the `fog-ingest` service can either reserialize new outputs into the old output format (i.e. drop the view tag field), or make a new enclave that can handle the new output format.



## Reference implementation

Not completed yet.



## References

- [Reduce scan times with 1-byte-per-output 'view tag'](https://github.com/monero-project/research-lab/issues/73)
