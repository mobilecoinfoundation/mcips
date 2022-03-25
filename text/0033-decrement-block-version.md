- Feature Name: Decrement-block-versions
- Start Date: Mar 25, 2022
- MCIP PR: [mobilecoinfoundation/mcips#0033](https://github.com/mobilecoinfoundation/mcips/pull/0033)
- Tracking Issue: https://github.com/mobilecoinfoundation/mobilecoin/issues/1699

# Summary
[summary]: #summary

Correct block version numbers in [mobilecoinfoundation/mcips#0026](https://github.com/mobilecoinfoundation/mcips/pull/0026),
and some related MCIPs.

Correct the description of the roll out plan for [mobilecoinfoundation/mcips#0003](https://github.com/mobilecoinfoundation/mcips/pull/0003)

# Motivation
[motivation]: #motivation

During compatibility testing, we discovered that our assumptions in MCIP #26 were wrong.
The status quo block version is block version 0, and all clients based on release 1.1.0 are
writing block version 0. Their value of `mc_transaction_core::BLOCK_VERSION`
is 0 and not 1, and they are incompatible with blocks of version 1.
MCIP #26 had assumed that the status quo is block version 1.

Therefore, that and some other MCIPs need to be updated.

Each planned future block version will be decremented, so that MCIP #3 features appear in block version 1 rather than 2,
and MCIP #25 features appear in block version 2 rather than 3, and only block version 0 is the status quo.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

None at this time.
