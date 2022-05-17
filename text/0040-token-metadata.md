- Feature Name: `token_metadata`
- Start Date: May 16 2022
- MCIP PR: [mobilecoinfoundation/mcips#0040](https://github.com/mobilecoinfoundation/mcips/pull/0040)
- Tracking Issue: (none yet)

# Summary
[summary]: #summary

Specify a source of truth for metadata about new tokens, and a way for clients to discover this,
by querying the consensus nodes.

# Motivation
[motivation]: #motivation

[MCIP #25](https://github.com/mobilecoinfoundation/mcips/pull/0025) and [MCIP #37](https://github.com/mobilecoinfoundation/mcips/pull/0037)
have specified ways for new tokens to exist on chain. However, for user-facing applications to make effective use of this feature,
they need more information about a token than just a token id -- they ultimately need to know how they should present balance information
to the user.

One way to do this is to hard-code all metadata into clients, which might work initially. After this however, this will mean that new tokens
that are created on chain can't actually be used by users until a client update has been released and the users have updated.

(In the [MCIP #31](https://github.com/mobilecoinfoundation/mcips/pull/0025) implementation PR, we did add a proto enum called `KnownTokenIds`
which can be shipped with clients, as a stop-gap measure.)

If the clients can dynamically query this information, then a client update is not required for clients to make use of new tokens which are
created, which significantly reduces the amount of time it takes for a new token to be available to users.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Token metadata includes the following:

* Short (exchange ticker) name of a token, e.g. MOB. (This is not expected to be translated.)
* Long (english) name of a token, e.g. MobileCoin. (This is expected to be translated, but the client must maintain the translation list.)
* Display conversion factor, e.g. 10^12. This conversion factor converts from the smallest representable units (`u64`) which appear in the blockchain,
  to the user-displayable unit. (Note that this is possibly not a power of 10.) `u64` amounts
  which are obtained from `TxOut`'s on chain should be divided by this number, for the token id, before being displayed to the user.
* (Optional) URL pointing to an SVG logo of the token. (This is expected to be scalable and appropriate for display on devices of many form factors.)
* (Optional) URL pointing to an MCIP which proposed the token.

The consensus network will provide an endpoint at which this metadata can be requested for one or more token ids.

This data will be provided at startup time via a file to the consensus node. (In the future, we may consider to store this data on chain.)
The node should require this data for any token which is configured to exist (has a minimum fee in `tokens.toml`).

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

A token metadata proto has these fields

```
message TokenMetadata {
    // The short name is a standard ticker symbol for the token
    String short_name = 1;
    // The long name is the standard english name for the token
    String long_name = 2;
    // The conversion factor is an unsigned 64 bit number expressed as a decimal string
    String display_conversion_factor = 3;
    // If present, a url pointing to a logo for the token
    String logo_url = 4;
    // If present, a url pointing to an MCIP which proposed the token
    String mcip_url = 5;
}
```

A consensus network client-facing API will be exposed to query for these objects given one or more token ids.

The `tokens.toml` file schema is extended to contain these fields per token. The minting trust root should sign
these fields.

# Drawbacks
[drawbacks]: #drawbacks

This proposal

* Adds to the complexity of configuring a network
* Does not specify a mechanism that forces different nodes to have matching values
  (we could choose to hash this data into responder id for instance)
* Does not commit this data to the blockchain
* Does not prevent the metadata from being changed -- only trust in the foundation not to sign another value
* URL fields cannot be validated by users. For instance, we could include hash of the logo svg, and then they
  could confirm the validity of the file they download.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

The main rationale to justify the proposal is to try to make incremental progress increasing the discoverability
of this important metadata.

In the status quo, where different clients would have to hard-code this data, and then release it according to their
release schedule, this may lead to a negative user experience. For example, if one user's wallet supports a new token id,
and they decide to send it to another wallet of theirs that hasn't updated, or hasn't made a release that supports it yet,
the new wallet will likely decide not to show this token to the user since it can't discover the metadata. From the user's
point of view, their tokens have disappeared and they don't know how to get them back, so this is a negative user experience.

With this proposal, all clients would be able to display all tokens with or without an update. So this seems to avoid
a significant negative user experience.

There are a number of ways this proposal could be strengthened however, described in the drawbacks section, around
ensuring the integrity of this data.

The main reasons not to do any of these is if we have uncertainty around whether / how this data should be prevented
from being changed. For instance, it seems quite bad if the display conversion factor is changed after a token is launched.
It might be desirable somehow for the logo or long name to be adjusted though.

We could also allow this data to be committed to the chain, and for desired parts of it to be updateable, as part of a `MintConfigTx`
as specified in MCIP #37. This would be a ledger format change, and so would require an enclave upgrade, increasing the amount of
time it would take us to deliver such a change (and to not mitigate the negative user experience scenario described).

Thus, the main rationale behind this form of the proposal would be as a stopgap that we can implement it quickly without an enclave change,
and then provide an API to clients that improves a user experience, and makes it faster for clients to support new tokens.
In a later proposal we could nail down the details of whether this data lives on or off chain, the ledger format change details if any,
the consistency enforcement mechanism, the verifiability of the data by the end user, and so on.

# Prior art
[prior-art]: #prior-art

In the Ethereum ecosystem, most tokens (fungible and non-fungible) have logos, but they are typically stored on centralized servers,
and referred to via URLs.

In [EIS-2569](https://eips.ethereum.org/EIPS/eip-2569) it was proposed to actually store SVG data on chain.

Typically, wallets like metamask use services like Infura to access names and logos of tokens. There is not a standard way outside of these
services to get this data for an arbitrary ERC-20.

In Algorand, any standard asset that is created has [required parameters](https://developer.algorand.org/docs/get-details/asa/#asset-parameters)

> These eight parameters can *only* be specified when an asset is created.
>
>    Creator (required)
>    AssetName (optional, but recommended)
>    UnitName (optional, but recommended)
>    Total (required)
>    Decimals (required)
>    DefaultFrozen (required)
>    URL (optional)
>    MetaDataHash (optional)

In Cardano, native assets can be created before being [registered with the token registry](https://developers.cardano.org/docs/native-tokens/token-registry/How-to-prepare-an-entry-for-the-registry-NA-policy-script).

Only a name is required to create the asset. When subsequently registering an asset, a "description" is required. Then a ticker, url, logo, and decimals are all considered optional.

In the Cardano registry, metadata can be deleted and updated, but it requires a signature from the creator.

In a [blog post](https://moxie.org/2022/01/07/web3-first-impressions.html), Moxie Marlinspike criticized web3 technologies that commit url's pointing to images to blockchains, but don't include e.g. a hash of the image on the chain, which would allow the user to verify that the url resolved to the correct image.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

* Should we attempt to solve this problem in two steps (one without an enclave change, and one which may have an enclave change),
  or just wait and do it all in one step.
* Should we allow metadata to be changed at all.
* Should we require a different set of fields.

# Future possibilities
[future-possibilities]: #future-possibilities

Depending on the outcome of discussion, we could increase the scope of this proposal significantly, or scope it more narrowly and make a follow-up MCIP.

We anticipate that we may want more metadata items than this as well, or to change the names of them.

It is also possible that we may want to handle internationalization differently, either requiring that there is a source of truth for how to translate token names
into other languages, or not providing a long name for a token at all, and leaving that entirely to the clients.

We suspect that in the end we will want all of this metadata to go onto the chain, and the main barrier to that is implementation complexity. But this may be considerably
simpler now that we have `MCIP 37` `MintConfigTx`'s.
