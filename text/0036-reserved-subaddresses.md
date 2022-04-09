- Feature Name: Reserved Subaddresses
- Start Date: (2022-04-10)
- MCIP PR: [mobilecoinfoundation/mcips#36](https://github.com/mobilecoinfoundation/mcips/pull/36)
- Tracking Issue: TODO

[summary]: #summary

Subaddress indices are unsigned 64-bit numbers.

The indices with high bit set are reserved for "special purposes".

[mobilecoinfoundation/mcips#4](https://github.com/mobilecoinfoundation/mcips/pull/4) will be updated,
changing the change subaddress from `1` to `u64::MAX`.
Subsequent reserved indices will count down from there.

# Motivation
[motivation]: #motivation

In developing new features, we have discovered that sometimes, we want to reserve particular subaddress
indices for specific purposes. This is particularly helpful for mobile wallets which are designed to
be used with "linked devices" that share your private keys. By using the subaddress index to represent
information about a `TxOut`, they can ensure that other linked devices will all see this information,
without needing to create another synchronization mechnaism.

A good example of this is the "change subaddress" introduced in
[mobilecoinfoundation/mcips#4](https://github.com/mobilecoinfoundation/mcips/pull/4).

However, this is at odds with how the desktop wallets use subaddresses. The desktop wallets
assign different subaddresses to different contacts, counting up from 1.

To avoid collisions and promote compatibility, we want to ensure that the change subaddress and other
special subaddresses will never be used for contacts by desktop wallets. This will help desktop wallets
to be able to make as much sense of this information as possible if a user imports their mobile wallet keys
into a desktop wallet.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Subaddress indices with the high bit set are reserved for special purposes.

Examples:
* The "change subaddress" from [MCIP #4](https://github.com/mobilecoinfoundation/mcips/pull/4)
* The "gift code subaddress" from [MCIP #32](https://github.com/mobilecoinfoundation/mcips/pull/32)

These will be assigned numbers counting down from `u64::MAX`. This avoids collisions
with subaddresses already assigned by the desktop wallet.

# Drawbacks
[drawbacks]: #drawbacks

This is reserving a lot of subaddresses. However, if later it turns out we need to
give most of them back, that should be fine.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

None at this time.

# Future possibilities
[future-possibilities]: #future-possibilities

None at this time.
