- Feature Name: token_metadata_document
- Start Date: 2022-12-02
- MCIP PR: [mobilecoinfoundation/mcips#0059](https://github.com/mobilecoinfoundation/mcips/pull/0059)
- Tracking Issue: TBD

# Summary
[summary]: #summary

This MCIP specifies a means for clients to dynamically discover metadata about token ids
on the MobileCoin blockchain, so that they can then display the balance of a new token to customers
in an appropriate way.

This proposal differs from an earlier proposal MCIP 40, in that it is a stop-gap mechanism which
does not post metadata to the blockchain.

This proposal specifies how clients should dynamically discover token metadata, authenticate this data,
and how they can go on to use this data.

# Motivation
[motivation]: #motivation

When a client gets a `TxOut`, at the lowest level, both the `value` and `token_id` are `u64`
numbers which can be decrypted from the blockchain records. However, it isn't appropriate to
display the `token_id` number directly to the users, since they won't know what this means.

Metadata about a `token_id` consists of all the data which helps clients figure out how they
can appropriately represent value to the users. Tampering with this metadata can represent an attack
to confuse or steal from the users, so there is considerable interest in ensuring that there
is a robust way for clients to authenticate the metadata.

If we do not support dynamic discovery of token metadata, then it must always be baked into clients,
which is what we do today.

Because some clients have relatively long upgrade times, this means that new
tokens that are launched still cannot actually be used by users. This effectively hurts the
time-to-market on any new token that is created.

There are second order effects to this -- if tokens become "available" in clients at different
points in time, then users can get bitten in a bad way. For instance, if a Moby user sent eUSD
to a Signal user, and at the time, Moby supports eUSD but Signal does not, then the recipient
cannot see the payment in their app, or send it anywhere, or send it back to the Moby user.
This could lead to disputed payments, bad user experiences, etc.

At the same time, putting token metadata on chain will require an enclave upgrade, while creating
new tokens on the chain does not. So there is considerable interest in having an off-chain source
of token metadata that can be validated by clients.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

This proposal creates a source of truth for token metadata for tokens on the MobileCoin blockchain.

Two documents are hosted on the MobileCoin foundation website:

* `https://www.mobilecoin.com/token_metadata.json`: A JSON document containing token metadata
* `https://www.mobilecoin.com/token_metadata.sig`: An ed25519 signature over the bytes of the .json document. This is binary data.

The ed25519 signing key will be controlled by the mobilecoin foundation.
This MCIP will be updated, and the public key will be posted here, after the key is created,
if the proposal is accepted.

Token metadata ed25519 public key hex bytes: `TBD`

The `token_metadata.json` document has the following schema (example):

```
{
    "signing_timestamp": "10020032320032",
    "metadata": [
        {
            "token_id": "0",
            "currency_name": "MobileCoin",
            "ticker_symbol": "MOB",
            "decimals": 12,
            "logo_svg": "b64...==",
            "info_url": "https://www.mobilecoin.com"
        },
        {
            "token_id": "1",
            "currency_name": "Electronic Dollars",
            "ticker_symbol": "eUSD",
            "symbol": "$",
            "decimals": 6,
            "logo_svg": "b64...==",
            "info_url": "https://mobilecoin.com/blog/mobilecoin-launches-eusd"
        },
        {
            "token_id": "14",
            "currency_name": 'Meowblecoin',
            "ticker_symbol": 'MEOW',
            "decimals": 12,
            "logo_svg": "b64...==",
            "info_url": "https://www.meowblecoin.com"
        }
    ],
}
```

This metadata list is conceptually a map from `token_id` integers to the metadata
about the token of that id.

For example:

* A `TxOut` with token id of 0 and a (u64) value of `1.5 * 10^12`, may be displayed as "1.5 MOB" to the user, because the token id corresponds to MOB, and the number of decimals of MOB is 12.
* A `TxOut` with a token id of 1 and a  (u64) value of `5 * 10^8` may be displayed as `500 eUSD` to the user, because the token id corresponds to eUSD, and the number of decimals is 6.
* That `TxOut` might also be displayed as `$500` because the symbol of `eUSD` is specified as `$`.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The token-metadata json fields have the following semantics:

* `signing_timestamp`: A string representation of the unix timestamp at the time of creating the document's signature.
* `token_id`: A decimal string representation of the `u64` corresponding to the token id.
* `currency_name`: A UTF-8 string which is the full, official name of the currency. This is intended to be an english-language noun or noun phrase.
* `ticker_symbol`: A ticker symbol for the currency. This is appropriate to be displayed by an exchange, as a short form of the name of the currency. Not more than 12 printable ASCII characters, including `[A-Za-z0-9].-`.
* `symbol`: Most fiat currencies have a printable character such as $, £, ¥. Some cryptocurrencies do also, Bitcoin has ₿, and Ethereum has Ξ. This may be optionally specified as a UTF-8 string in the "symbol" field. How exactly it is displayed may be locale specific and we do not attempt to formally specify that at this time.
* `decimals`: An integer specifying how the `u64` integer in a TxOut in the blockchain is scaled to compute a user-displayed amount of the currency.
* `logo_svg`: An optional logo image. This is a base64-encoded SVG document. This is expected to have been sanitized using something like [svg-hush](https://github.com/cloudflare/svg-hush), to remove scripting, hyperlinks to other documents, and references to cross-origin resources. The full extent of such sanitization will not be specified here.
* `info_url`: A link to a website containing more information about the token that may be interesting to token holders. This should basic information about the purpose of the token, its supply, any utility that it has, or links to associated whitepapers.

Clients MUST download and validate the `token_metadata.sig` before attempting to process the `token_metadata.json`, and must reject the json with an error if the signature fails.

Clients SHOULD include a minimum for `signing_timestamp` as part of their build, and reject any `token_metadata.json` which is less than the baked-in `signing_timestamp`. This prevents replay attacks where an old `token_metadata.json` document is substituted for the latest one with the goal of confusing the users by making their tokens display differently or not at all. They can fetch the latest `token_metadata.json` at build time to get the latest `signing_timestamp`.

Clients MAY continue to bake token metadata into their build at their discretion, for instance, in order to support offline wallets or other cases where the client cannot access the internet. When both baked-in metadata and the `token_metadata.json` are available, the validated `token_metadata.json` should be considered a more reliable source of truth.

Maintainers of this document SHOULD NOT:

* Allow duplicate records for a given token id
* Delete a token id's record
* Modify data such as the `ticker_symbol` or `decimals` of a token, which may confuse users, and particularly, any exchanges that use this as a source of truth.
* There may be legitimate reasons to make changes the `currency_name` or `logo`, but this needs to be carefully considered.
* Allow two token ids with `currency_name`s which could be confused
* Allow the same `ticker_symbol` to appear, or, allowing two `ticker_symbols` which differ only in case, or the replacement of letters with similar numbers.

# Drawbacks
[drawbacks]: #drawbacks

This proposal has a few drawbacks relative to [MCIP 40](https://github.com/mobilecoinfoundation/mcips/pull/0040):

* MCIP 40 puts the token metadata on chain. This makes it possible to enforce protocol level rules around how the document can evolve. It also ensures the availability of the document.
* It also would take longer to develop since it would require an enclave upgrade.

This proposal's authentication scheme is a single ed25519 signature.

* As an alternative, we could have a signature chain, such as an x509. This allows for more sophisticated strategies around the keeping the key secure.
* As an alternative, we could have a k-of-n ed25519 multisignature. This would allow for the signing authority to be shared across several parties who can sign asynchronously.

Since this signature can easily be performed off-line anyways, and the document isn't expected to change often, we think a single ed25519 signature is fine for this stop-gap proposal.

In our design, we included SVG images inline in the metadata document to avoid this validation issue.
This will not scale well to e.g. millions of tokens, and we expect to transition to on-chain metadata before that becomes a problem.

This proposal does not envision having different token registries per network. It is believed that this complexity is unnecessary.
However, it means that just as clients need to hit Intel's IAS services as part of testing (in prod and dev), they will have to hit
the `mobilecoin.com` URLs as well, so this is another way in which testing a deployed network will not be completely isolated to the servers deployed as part of the network.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

We chose json instead of protobuf to make it easier for clients (which may be web browsers) to directly consume this document in e.g. javascript.

We chose to separate the signature from the document to avoid the complexity of storing serialized json within json.
This also allows that we can modify the signature scheme in the future while maintaining backwards compatibility, as long as we continue to serve both the old signature and the new signature,
at different url's.

We chose not to attempt to standardize the localization of any cryptocurrency names, since it does not seem conventional to do so in cryptocurrency wallets.

We chose to allow numbers and some punctuation to be used in ticker symbols.

This is based on looking at token lists on various cryptocurrency exchanges:

* [Binance.US token list](https://support.binance.us/hc/en-us/articles/360049417674-List-of-Supported-Assets)
* Kraken's exchange includes the following symbols, where suffixes such as `.S` suffix are used with staked tokens and similar.

```
[
  "1INCH",
  "AAVE",
  "ACA",
  "ACH",
  "ADA",
  "ADA.S",
  "ADX",
  "AED",
  "AED.HOLD",
  "AGLD",
  "AIR",
  "AKT",
  "ALCX",
  "ALGO",
  "ALGO.S",
  "ALICE",
  "ALPHA",
  "ANKR",
  "ANT",
  "APE",
  "API3",
  "APT",
  "ARPA",
  "ASTR",
  "ATLAS",
  "ATOM",
  "ATOM.S",
  "AUD.HOLD",
  "AUDIO",
  "AVAX",
  "AXS",
  "BADGER",
  "BAL",
  "BAND",
  "BAT",
  "BCH",
  "BICO",
  "BIT",
  "BLZ",
  "BNC",
  "BNT",
  "BOBA",
  "BOND",
  "BSX",
  "BTT",
  "C98",
  "CAD.HOLD",
  "CELR",
  "CFG",
  "CHF",
  "CHF.HOLD",
  "CHR",
  "CHZ",
  "COMP",
  "COTI",
  "CQT",
  "CRV",
  "CSM",
  "CTSI",
  "CVC",
  "CVX",
  "DAI",
  "DASH",
  "DENT",
  "DOT",
  "DOT.P",
  "DOT.S",
  "DYDX",
  "EGLD",
  "ENJ",
  "ENS",
  "EOS",
  "ETH2",
  "ETH2.S",
  "ETHW",
  "EUR.HOLD",
  "EUR.M",
  "EWT",
  "FARM",
  "FET",
  "FIDA",
  "FIL",
  "FIS",
  "FLOW",
  "FLOW.S",
  "FLOWH",
  "FLOWH.S",
  "FLR",
  "FORTH",
  "FTM",
  "FXS",
  "GAL",
  "GALA",
  "GARI",
  "GBP.HOLD",
  "GHST",
  "GLMR",
  "GMT",
  "GNO",
  "GRT",
  "GRT.S",
  "GST",
  "GTC",
  "HFT",
  "ICP",
  "ICX",
  "IDEX",
  "IMX",
  "INJ",
  "INTR",
  "JASMY",
  "JUNO",
  "KAR",
  "KAVA",
  "KAVA.S",
  "KEEP",
  "KEY",
  "KFEE",
  "KILT",
  "KIN",
  "KINT",
  "KNC",
  "KP3R",
  "KSM",
  "KSM.P",
  "KSM.S",
  "LCX",
  "LDO",
  "LINK",
  "LPT",
  "LRC",
  "LSK",
  "LUNA",
  "LUNA.S",
  "LUNA2",
  "MANA",
  "MASK",
  "MATIC",
  "MATIC.S",
  "MC",
  "MINA",
  "MINA.S",
  "MIR",
  "MKR",
  "MNGO",
  "MOVR",
  "MSOL",
  "MULTI",
  "MV",
  "MXC",
  "NANO",
  "NEAR",
  "NMR",
  "NODL",
  "NYM",
  "OCEAN",
  "OGN",
  "OMG",
  "ORCA",
  "OXT",
  "OXY",
  "PARA",
  "PAXG",
  "PERP",
  "PHA",
  "PLA",
  "POLIS",
  "POLS",
  "POND",
  "POWR",
  "PSTAKE",
  "QNT",
  "QTUM",
  "RAD",
  "RARE",
  "RARI",
  "RAY",
  "RBC",
  "REN",
  "REPV2",
  "REQ",
  "RLC",
  "RNDR",
  "ROOK",
  "RPL",
  "RUNE",
  "SAMO",
  "SAND",
  "SBR",
  "SC",
  "SCRT",
  "SCRT.S",
  "SDN",
  "SGB",
  "SHIB",
  "SNX",
  "SOL",
  "SOL.S",
  "SPELL",
  "SRM",
  "STEP",
  "STG",
  "STORJ",
  "STX",
  "SUPER",
  "SUSHI",
  "SXP",
  "SYN",
  "T",
  "TBTC",
  "TEER",
  "TLM",
  "TOKE",
  "TRIBE",
  "TRU",
  "TRX",
  "TRX.S",
  "TVK",
  "UMA",
  "UNFI",
  "UNI",
  "USD.HOLD",
  "USD.M",
  "USDC",
  "USDT",
  "UST",
  "WAVES",
  "WBTC",
  "WETH",
  "WOO",
  "XBT.M",
  "XCN",
  "XETC",
  "XETH",
  "XLTC",
  "XMLN",
  "XREP",
  "XRT",
  "XTZ",
  "XTZ.S",
  "XXBT",
  "XXDG",
  "XXLM",
  "XXMR",
  "XXRP",
  "XZEC",
  "YFI",
  "YGG",
  "ZAUD",
  "ZCAD",
  "ZEUR",
  "ZGBP",
  "ZJPY",
  "ZRX",
  "ZUSD"
]
```

Coinbase lists over 9000 tokens with letters, numbers, mixed case, and as many as 8 characters: `BabyDoge, LYXe, vBUSD, 7S, DRAGACE, TRINITY`.
Coinbase also lists the tickers `OXD V2`, `$ZOOM`, which would not be allowed in the above specification due to the space character and `$` character.

We do not recommend that space or `$` become allowed characters, as this creates the opportunity for confusion.

# Prior art
[prior-art]: #prior-art

We considered the metadata used in both Cardano native assets and Algorand standard assets when considering this proposal.
We also considered how tokens are assigned metadata in the Ethereum ecosystem.

In the Ethereum ecosystem, most tokens (fungible and non-fungible) have logos, but they are typically stored on centralized servers, and referred to via URLs.

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

In Cardano, native assets can be created before being [registered with the token registry](https://developers.cardano.org/docs/native-tokens/token-registry/How-to-prepare-an-entry-for-the-registry-NA-policy-script). Only a name is required to create the asset. When subsequently registering an asset, a "description" is required. Then a ticker, url, logo, and decimals are all considered optional. In the Cardano registry, metadata can be deleted and updated, but it requires a signature from the creator.

In a [blog post](https://moxie.org/2022/01/07/web3-first-impressions.html), Moxie Marlinspike criticized web3 technologies that commit url's pointing to images to blockchains, but don't include e.g. a hash of the image on the chain, which would allow the user to verify that the url resolved to the correct image. In this proposal, SVG images are inline in the metadata document.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

It is not clear to us if we should include more information about how the currency symbol is displayed, for example, before or after the number.
Is this a convention of currencies or of locales. More information about localization specifics for amounts of currencies would be helpful at this
time, and if e.g. a `symbol_localization` field should be added to the metadata to help clients handle this properly we should specify that. It may
also be fine to pin this down at a later time.

# Future possibilities
[future-possibilities]: #future-possibilities

In the future we hope to do on-chain token metadata as envisioned in MCIP 40.

This is likely required for doing user-created tokens, and will be required before there are too many tokens for this document to be manageable by humans.
