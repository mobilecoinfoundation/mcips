- Feature Name: `reserved_token_ids`
- Start Date: 2022-07-14
- MCIP PR: [mobilecoinfoundation/mcips#0045](https://github.com/mobilecoinfoundation/mcips/pull/45)
- Tracking Issue: N/A

# Summary
[summary]: #summary

MobileCoin 1.2 introduced the ability for the network to operate on different assets based on an unsigned 64-bit integer token ID. This MCIP exists primarily to maintain a canonical mapping of token IDs to underlying assets.

# Motivation
[motivation]: #motivation

We will need to allocate token IDs for development and testing purposes on our various networks. We also wish to ensure a given token ID is unique for an underlying asset across our networks: development and test network(s) (e.g. AlphaNet, TestNet, etc.), and our main network (i.e. MainNet).

It is also critically important that a given token ID not be re-used within a network, because (presumably) the new token is not a direct successor token to the previously existing token, and it is not considered reasonable to expect the original token to be completely burned via transaction to the verifiable burn address. If a holder of a particular token loses their private key, then the contents of their account cannot be verifiably burned.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

As a result, the primary purpose of this change is to ensure that the same token ID is not re-used on a given network, but secondarily to ensure that a given token ID is also not re-used *between* networks. 

This is done primarily to aid engineers in their debugging and testing by creating a central repository they can use to provide something akin to an order-of-magnitude style check on tokens in use in a particular environment.  For example, a token ID in the MainNet reserved range should never appear on TestNet, and similarly, a token ID in the TestNet reserved range should never appear on MainNet.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

This is a list of token ID ranges, and what we intend to use them for. This list should be updated by future MCIPs which add additional token types to public networks such as MainNet and TestNet.

| Start      | End        | MCIP                                                                                               | Use                      | MainNet | TestNet | DevNet(s) | Comments                                                                                   |
|------------|------------|----------------------------------------------------------------------------------------------------|--------------------------|---------|---------|-----------|--------------------------------------------------------------------------------------------|
| `0`        |            | [#25](https://github.com/mobilecoinfoundation/mcips/blob/main/text/0025-confidential-token-ids.md) | MOB                      | Yes     | Yes     | Yes       | Original Token                                                                             |
| **1**      | **2,047**  |                                                                                                    | **Reserved for MainNet** | Yes     |         |           | Tokens intended to be used on MainNet, with fees (potentially) payable in the token itself |
| **2,048**  | **8,191**  |                                                                                                    | **MainNet**              | Yes     |         |           | Tokens intended to be used on MainNet, with fees payable only in MOB.                      |
| **8,192**  | **10,239** |                                                                                                    | **TestNet**              |         | Yes     |           | Tokens intended to be used on TestNet exclusively.                                         |
| **10,240** | **16,383** |                                                                                                    | **AlphaNet**             |         |         | Yes       | Tokens intended to be used exclusively on persistent testing networks                      |
| **16,384** | **32,767** |                                                                                                    | **Developer Testing**    |         |         | Yes       | Tokens intended to be used on ephemeral "testing" networks and deployments.                |
| **32,768** | **2^64**   |                                                                                                    | **Reserved**             |         |         |           | Reserved for future use                                                                    |

# Drawbacks
[drawbacks]: #drawbacks

There are several potential drawbacks to this approach to allocation of token IDs:

1. By their nature, ranges chosen prior to widespread adoption will always be inflexible. For example, if we later decide that we need more than 2,047 tokens which can pay fees in the token itself, we will need to allocate additional ranges from the last "reserved" block (32,768-2^64).
2. While intended, requiring an additional MCIP for each new token that will live on the chain is overhead. It is the opinion of this document that this amount of overhead is likely acceptable given the permanent nature of token selection.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

The primary alternative to this proposal would be to simply not do this, and decide which tokens are given which TokenID on an ad-hoc basis---relying on the `tokens.toml` configuration to be the source of truth. A secondary alternative to this proposal would be to simply use a unified set of tokens across all networks.

Neither option is considered viable because neither one handles de-confliction of tokens between networks (the secondary option, intentionally). This is considered vital as a basic integrity check which users and developers.

# Prior art
[prior-art]: #prior-art

In the purest sense, any registry of assigned numbers could be considered prior art, so they will not be enumerated here. 

# Unresolved questions
[unresolved-questions]: #unresolved-questions

None

# Future possibilities
[future-possibilities]: #future-possibilities

The most obvious future possibility would be the automated provisioning of new Tokens (and IDs) from the reserved range based on payments made to the MobileCoin Foundation.
