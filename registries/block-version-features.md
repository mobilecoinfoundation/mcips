- Registry: Block version features

# Summary
[summary]: #summary

This table tracks what **features** are tied to what **block version numbers**, and
provides links back to approved MCIPs that proposed those features.

See [MCIP #26 Block Version-based Protocol Evolution](../text/0026-block-version-based-protocol-evolution.md) for more context.

This table is supposed to reflect the current state of the world according to
all the MCIPs that have been merged, and not a historical state.

# Motivation
[motivation]: #motivation

This helps everyone to keep track of what breaking changes occurred across
a block version number bump.

# Table

| Feature name                       | Block Version | Description                                     | Relevant MCIP(s)                                                    | Relevant schema additions |
| ------------                       | ------------- | -----------                                     | ----------------                                                    | ------------------------- |
| Encrypted Memos                    | 1             | A standard format for encrypted memos           | [MCIP #3](https://github.com/mobilecoinfoundation/mcips/pull/0003)  | `TxOut::e_memo`           |
| Confidential Tokens                | 2             | Support for secret token ids on TxOuts          | [MCIP #25](https://github.com/mobilecoinfoundation/mcips/pull/0025) | `Amount::masked_token_id` |
| Mint Transactions                  | 2             | Support for configuring, minting new token ids  | [MCIP #37](https://github.com/mobilecoinfoundation/mcips/pull/0037) | `MintTx`, `MintConfigTx`  |
| MLSAG sign extended message digest | 2             | Cryptographic hardening around MLSAG signatures | [MCIP #25](https://github.com/mobilecoinfoundation/mcips/pull/0025) | -                         |
| Enforce Sorted TxOuts              | 3             | `TxPrefix.outputs` must be sorted by public key | [MCIP #34](https://github.com/mobilecoinfoundation/mcips/pull/0034) | -                         |
| Mixed Transactions                 | 3             | Transactions may use more than one token id     | [MCIP #31](https://github.com/mobilecoinfoundation/mcips/pull/0031) | `SignatureRctBulletproofs::{range_proofs, pseudo_output_token_ids, output_token_ids}` |
| Signed Input Rules                 | 3             | MLSAG's may sign a `TxIn` with a set of `InputRules` instead of the extended message. During validation, any `InputRules` are validated against the whole `Tx`. An atomic swap protocol is specified using a new `SignedContingentInput` object which uses this. | [MCIP #31](https://github.com/mobilecoinfoundation/mcips/pull/0031) | `TxIn::InputRules, SignedContingentInput` |
| Block Metadata                     | 3             | Block metadata is specified for ArchiveBlock format, containing signatures and AVRs | [MCIP #43](https://github.com/mobilecoinfoundation/mcips/pull/0043) | `ArchiveBlockV1::BlockMetadata` |
| Masked Amount V2                   | 3             | Masked Amount key derivation is changed to include an "amount shared secret". | [MCIP #42](https://github.com/mobilecoinfoundation/mcips/pull/0042) | `MaskedAmount => OneOf { MaskedAmountV1, MaskedAmountV2 }` |
| Partial Fill Rules                 | 3             | Extend 'Signed Input Rules' feature to support partial fill rules. This enables partial fill atomic swaps. (Note: In code this support is tied to the 'Masked Amount V2' feature.) | [MCIP #42](https://github.com/mobilecoinfoundation/mcips/pull/0042) | `InputRules::{partial_fill_change_output, partial_fill_outputs, min_partial_fill_value}` |
| Nested Signer Sets for Minting                  | 3             | Extend `SignerSet` to support nesting. This enables hierarchical multi-sigs | [MCIP #55](https://github.com/mobilecoinfoundation/mcips/pull/0055) |  `SignerSet::{individual_signers, multi_signers}` |
