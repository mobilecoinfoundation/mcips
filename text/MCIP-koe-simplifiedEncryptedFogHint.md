```
  MCIP: ?
  Layer: Consensus
  Title: Simplified Encrypted Fog Hint
  Authors: koe <ukoe@protonmail.com>
  Status: Draft (DO NOT USE)
  Type: Standards Track
  Created: ?
```

## Abstract

Remove the 16-byte MAC from encrypted fog hints.



## Motivation

MobileCoin blockchain data is stored permanently for all time. Reducing how much data is stored per-transaction improves the system's ability to support large transaction volumes, even if such improvements are only marginal (in this case, < 10% reduced storage per output on average).

A MAC in the encrypted fog hint is not strictly necessary, so it is harmless to remove.



## Encrypted fog hint

### Output format change

- Remove the 16-byte MAC component from encrypted fog hint fields in MobileCoin outputs (thereby shortening the field by 16 bytes).


### Block ID adjustment

New block IDs should be computed from outputs with the new format (an implicit semantic change).


### Encrypted fog hint handling

- The encryption scheme for fog hints may remain the same. To decrypt an encrypted fog hint, prepend an empty 16-byte fake MAC to it, and ignore the decryption success/failure result.
- If decryption succeeds, the decrypted message should be a compressed Ristretto point. Checking if it is a legitimate compressed Ristretto must be done in constant time.



## Rationale

#### Why was a MAC originally included with encrypted fog hints?

MACs are typically used to detect if decrypting a cyphertext succeeds or fails.

The Fog service's `fog-ingest` enclave decrypts fog hints in constant-time. If decrypting a fog hint fails, then the enclave will use a randomly generated Ristretto point to construct the corresponding output's `ETxOutRecord`. The failed decrypted bytes cannot be used directly for finishing the `fog-ingest` procedure because there will be a loss of consant-timeness if the decrypted bytes aren't a legitimate compressed Ristretto point.

Successful vs. failed decryptions can be detected in constant-time via a MAC.

#### Why is it safe to remove the MAC?

In the specific context of `fog-ingest`, instead of using a MAC to detect if decryption succeeded/failed, you can check if the decrypted bytes decompress into a Ristretto point with a constant-time decompression algorithm.

- If decompression succeeds, then use the decompressed point for the `fog-ingest` algorithm.
- If decompression fails, then use a pre-generated random Ristretto point for the `fog-ingest` algorithm.

If decryption failed (i.e. the original encrypted text isn't recovered), but the decrypted bytes just happen to decompress into a Ristretto point, this is acceptable from the perspective of `fog-ingest` (it should happen for 1 in 8 failed decryptions). The decompressed Ristretto point is effectively a random point, and `fog-ingest` wants to use a random Ristreto point after the decryption step if decryption fails.



## Backward compatibility

This proposal represents a ledger format change (new format for outputs) and block construction change (block IDs computed from new output format). It must be implemented via a hard fork of some kind.

The Fog service's `fog-ingest` component could retain backward compatibility with old outputs by reserializing them into the new output format before they are consumed by the ingest enclave (i.e. drop the MAC component of the encrypted fog hint field). Since Fog `TxOutRecords` do not include the encrypted fog hint, no changes are required for the `fog-view` service. However, the `fog-ledger` service would have to adjust its database to handle the new output format.



## Reference implementation

Not completed yet.



## References

- None
