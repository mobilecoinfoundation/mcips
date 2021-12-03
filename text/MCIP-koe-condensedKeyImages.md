```
  MCIP: ?
  Layer: Consensus
  Title: Condensed Key Images
  Authors: koe <ukoe@protonmail.com>
  Status: Draft (DO NOT USE)
  Type: Standards Track
  Created: ?
```

## Abstract

Store a 16-byte hash of each key image instead of as a compressed Ristretto point.



## Motivation

MobileCoin blockchain data is stored permanently for all time. Reducing how much data is stored per-transaction improves the system's ability to support large transaction volumes, even if such improvements are only marginal (in this case, < 10% reduced storage per output on average).



## Condensed key images

A condensed key image is computed as follows:
```
H16("mc_condensed_key_image", KI)
```

- `H16()`: hash function with 16-byte message digest [[[TODO: construction]]]
  - Domain separation string: "mc_condensed_key_image"
- `KI`: the key image in canonical compressed Ristretto point format



## Block IDs

Key images created before this proposal have been stored in the ledger as compressed Ristretto points. These are used to compute each block's block ID (as part of the `contents_hash` sub-field in block headers), and cannot be discarded.

To support this proposal:

1. New block headers should have a `contents_hash` based on condensed key images instead of the compressed Ristretto point versions.
2. Old key images should still be stored in compressed Ristretto point format for reconstructing old block IDs.
  - The compressed Ristretto form of new key images does not need to be recorded.
3. Old key images should be duplicated, converted to condensed key image format, and stored alongside new condensed key images for double-spend checks.



## Double-spend checks

To perform a double-spend check for key images in a new transaction:

1. Convert the transaction's key images to condensed key image format.
2. Check the ledger's condensed key image storage for duplicates.



## Rationale

#### Why is it safe to discard the compressed Ristretto point form of new key images?

In MobileCoin, most transaction data is discarded once a transaction is added to a block. After that point, there is no possibility or expectation of re-validating transactions. Therefore, the full version of a key image is not strictly useful after the key image has been added to the blockchain.

Key images cannot be completely discarded, because they are used to prevent double spends. Future transactions must prove the outputs they are spending haven't been spent before by creating key images that haven't appeared on-chain yet. Comparing deterministic representations of key images (e.g. hash digests or bit-wise truncations) accomplishes that task.

#### Why use 16 bytes for condensed key images?

The most important property for key images is collision-resistance. Users should never encounter a situation where they can't spend an output because a different output's key image matches their output's key image (assuming one-time addresses [i.e. 'target' addresses] are unique on-chain).

If the total number of variations of a number is 2^N, then, according to the birthday problem, there is a 50% chance of a collision existing if you randomly sample from that number-space 2^(N/2) times.

It is safe to assume the total number of key images on-chain will not exceed 2^56 (7.2 quadrillion). If N = 128 (i.e. if key images are uniformly mapped to a 128-bit number-space), then at 2^56 key images there should be a `(2^56)^2 / 2*2^128 = 2^(-17) = ~0.001%` chance that a collision exists (using the square approximation for the birthday problem).

This proposal asserts that a 0.001% chance of a collision appearing in the entire lifetime of MobileCoin is sufficiently low to justify 16-byte condensed key images.



## Backward compatibility

This proposal represents a ledger format change (new storage for condensed key images), block construction change (new block IDs computed from condensed key images), and a new transaction validation rule (check condensed key images for duplicates). It must be implemented via a hard fork of some kind.



## Reference implementation

Not completed yet.



## References

- None
