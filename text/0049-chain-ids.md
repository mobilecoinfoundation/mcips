- Feature Name: `chain_ids`
- Start Date: 2022-07-25
- MCIP PR: [mobilecoinfoundation/mcips#0049](https://github.com/mobilecoinfoundation/mcips/pull/0049)
- Tracking issue: [mobilecoinfoundation/mobilecoin#2311](https://github.com/mobilecoinfoundation/mobilecoin/issues/2311)

# Summary
[summary]: #summary

Each running MobileCoin consensus network will be started with a string, the "chain id".

Clients which make requests to consensus **may** pass the expected chain id as a header that goes with their request.
If this doesn't match the server, an error is returned, which includes the server-side chain id in the response.
Client-facing fog servers are similarly updated.

In a second step, "chain id" will be incorporated into the blockchain itself, and the enclaves will become chain-id aware.

# Motivation
[motivation]: #motivation

Two important and competing engineering goals for all financial software are:

* Enable Rigorous Testing: Properly test the code before it goes live in a test environment that matches production as closely as possible.
* Prevent Mixing of Testing and Production: Make it hard for users to think they are talking to a test environment when they are actually talking to production and vice versa.

The basic issue is, we **do** want the test servers and clients to match production as closely as possible, so that we test the code and the code paths
that we will actually ship. On the other hand, if there is **no** way for the software to distinguish the two environments, then there is no way to have
safeguards that prevent e.g. accidental sending of money or minting on Mainnet.

To date, we have mostly tried to solve the "mixing" problem in the following way:

* Testnet, Prod, and Dev staging environments all have different attestation trust roots
* Clients are either built with a hard-coded trust root, or at packaging time are supplied an appropriate css file.
* Clients will fail attestation if they connect to the wrong chain, so as a result they cannot mix networks.

There are a few problems with this:

* When mixing networks occurs, an attestation failure error message results, but this error message is very hard to understand.
* Clients typically don't clearly communicate to the user which chain they are on. So this doesn't ultimately fix the problem,
  and indeed sometimes users think they are using a testnet build but actually they are using a mainnet build.
* Now that we have the [MCIP #37](https://github.com/mobilecoinfoundation/mcips/pull/37) minting mechanism, we have more calls that clients can make to the consensus network,
  but minting commands are not attested, so there is no mechanism to prevent mixing of test and prod for that call.

To try to overcome the latter issue, one form of [MCIP #45](https://github.com/mobilecoinfoundation/mcips/pull/45) proposed things like, every token on mainnet will have a different token id
from testnet, and every devnet token id will also be different. This is roughly analogous to something like "in the stock exchange, in prod the ticker is GOOG, and in test the ticker is TESTGOOG".

Presumably, the next time we add a feature to consensus that adds a new API endpoint, we will need to add more configuration and more fragmentation of this
configuration across networks, to try to prevent mixing of test and prod when this endpoint is called.

This fragmentation is harmful for several reasons.

* Increases the complexity of properly configuring / deploying a network, which is already high.
* The intent of "GOOG" vs. "TESTGOOG" is that it prevents a user from being confused and thinking test data is production data,
  because they'll see the word "TEST", but testing isn't harmed much because the software otherwise works the same.
  However, for our software, lacking [MCIP #40](https://github.com/mobilecoinfoundation/mcips/pull/40) or any alternative proposal, client software needs to obtain metadata for each token
  id it encounters, or it cannot work properly. If this metadata doesn't match for GOOG and TESTGOOG then we may not properly be testing
  whether GOOG works properly before we go to prod. These same concerns aren't exactly true for a stock exchange.
* Lacking any per-network discoverability mechanism, clients are likely forced to hard-code the map from token ids to metadata.
  This raises serious complexity and maintainability concerns. Additionally, when we step back and look at the big picture, we can see that we are fighting ourselves:
  we are using server-side configuration to make all the token ids different on every network, arbitrarily, for "safety", but then we are hard-coding logic to basically
  undo that per-network mangling into each client so that we can test the clients properly.
* Ultimately, in a product-focused project, we want to be able to see in testing that "GOOG" displays correctly in the app when we test on testnet, and that all the correct
  UI elements work, and token-specific config is loaded correctly. This will likely motivate engineers higher up in the stack to make "TESTGOOG" and "GOOG" display in identical ways
  to the user of the app, so that they will know exactly how it will appear in real life in prod when they are testing in testnet.
  This completely defeats the purpose of "separate token ids per network", since the user will no longer be able to distinguish being on prod vs. being on testnet.

We propose to reduce complexity here and separate these concerns.

* "Chain id"-aware clients is a common pattern used by other blockchains to help clients (and users) be aware of what network they are talking to.
* Chain id can be easily displayed to the user if desired, which is something that none of our clients actually do today.
   * For example, when using Metamask, a user can select whether to talk to mainnet, an L2 network, or a test network, via a dropdown menu, and see what network they are currently talking to here.
     Even if we don't support dynamic switching of networks in a particular client, we can still display the network id somewhere, so that the users can discover TEST vs. PROD easily, without
     changing any of the other codepaths that we want to test.
   * This could be as simple as, if the chain id is not prod, then the chain id is displayed in a textbox in the upper right corner. Or, the chain id can be inspected from somewhere in the menu.
   * So instead of "TESTGOOG", you still see the ticker "GOOG", but "TEST" appears in the user interface somewhere else.
* All of our dev, test, and prod environments currently have names that can naturally be adopted for this purpose.
* Chain id can be our primary method of "de-conflicting" and preventing mixing of networks.
* We won't have to solve this again every time we add a new feature to consensus, and we can simplify future designs by reducing requirements. 
* This has the benefit that the error messages when networks are mixed will be consistent and much clearer.

Note that while we are partly motivated by reducing complexity and easing testing around [MCIP #45](https://github.com/mobilecoinfoundation/mcips/pull/45), it is possible that we will do [MCIP #45](https://github.com/mobilecoinfoundation/mcips/pull/45) as originally proposed
and this proposal -- they are orthogonal, strictly speaking. We could decide to adopt this proposal as a secondary layer of defense against mixing problems, or in service
of a new consensus feature in the future.



# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

A "chain id" is a human-readable string, which is part of the configuration of consensus and fog servers.
This configuration is now required for these servers.
All the servers in a given deployment are expected to match. Mainnet, testnet, and dev networks should all have different chain ids.

Servers with client-facing APIs now check for a `chain-id` GRPC header, which the client uses to say what network they are trying to talk to
when they make a request.

Servers **must** check this client-provided value against their configured chain id, and return an error if it doesn't match.

Clients **should**:

* Be aware of the chain id of the network they are configured to talk to
* Make this information discoverable to the user in some appropriate way. It is primarily important to highlight the fact that things are in dev or test networks rather than
  to add visual noise or untranslated strings in production -- it may be fine to display nothing in prod as a way of distinguishing it, depending on the application.
* Take advantage of these new headers to avoid mistakes

When the client is a cross-chain bridge, configuration should be organized so that chain-id is determined at the same time that the chain-id for the other chain is determined,
and there should be a single source of truth for this association. It should be made very hard to accidentally configure the bridge to talk to e.g. an Ethereum testnet and also a MobileCoin production environment.

For example, either a hard-coded table or a configuration toml file could be created for the bridge which associates to a MobileCoin chain id:

* URLs of MobileCoin services
* URLs of any other chain services
* Chain ids of any other networks
* Contract ids for any relevant contracts

This is better than each of these being independent configuration parameters to the software. They should all be changing in lock-step.

In block-version 3, chain-id will be incorporated to consensus in a more comprehensive way:

* The enclave will become `chain-id` aware, so that the network cannot peer if the chain-id's don't match.
* A `chain_id` field will be added to the block header in block-version 3.
* The `TxPrefix` of a transaction will include `chain_id`. This will be checked by transaction validation.

# Reference-level explanation
[guide-level-explanation]: #guide-level-explanation

A "chain id" is a string. The string must be a valid DNS label. In particular:

* Up to 63 bytes long
* Valid characters are `a-z`, `0-9`, and `-` (hyphen). Capital letters are not allowed here.
* You can't specify a hyphen at the beginning or end.

Any deployed network must have a specified chain id.

* The chain id of mainnet **shall** be `main`
* The chain id of testnet **shall** be `test`
* Dev networks and any other kind of test networks **should** use strings other than these
* For local testing a chain id of `local` is recommended.

Chain id's are intended to be human readable, but primarily this is a concern of test builds.
They aren't expected to be treated as "translatable strings" for purposes of internationalization.

Consensus and fog servers are passed this string on startup by either a flag `--chain-id` or an environment variable `MC_CHAIN_ID`.

* It is an error if it is missing.
* This is called the "configured chain id".

Client facing APIs are extended to include an optional "chain-id" header.

* For GRPC connections, the header is spelled `chain-id`
* For HTTP connections, the header is spelled `Chain-Id`
* If present, the untrusted server code checks this against the configured chain
  id immediately upon deserializing the response.
* If it is not a match, then an error is returned to the client.
* The response details must include a string of the form "chain-id mismatch: 'foo'", where foo is the server's configured chain id, to help debug clients.

This header is optional to support backwards compatibility.

In block-version 3, the following changes will occur:

* The enclave will become `chain-id` aware. The `BlockchainConfig` will include chain-id, so that consensus peers can only attest if they have matching chain-ids.
* A `chain_id` field will be added to the block header in block-version 3. The enclave fills this in from its config when forming blocks.
* The `TxPrefix` of a transaction will include `chain_id`. Transaction validation will be updated to check that this chain-id matches the enclave's chain id.

Additionally, more follow-on changes may include things like:

* The ledger will check that it does not consume blocks that don't match its chain id, after block version 3.
* Fog-ingest will check that blocks that it consumes have the expected chain id.

# Drawbacks
[drawbacks]: #drawbacks

This adds more configuration to the network.

Clients may send a few more bytes with each request.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

The primary alternative is, do nothing:

* Continue to use ad-hoc configuration differences like attestation trust roots and token ids as the primary means to distinguish dev, test, prod environments.
* Continue to introduce more such ad-hoc configuration differences as more features are added.
* Cross-chain bridges primarily lean on attestation roots to avoid e.g. accidentally minting on the wrong network, but it is a bit harder to manage the configuration,
  because attestation roots live in files.

Another alternative: The Ethereum [JSON-RPC API](https://ethereum.org/en/developers/docs/apis/json-rpc/) returns the network id as a string, but in practice all the ids are actually integers.
The mapping from network names to ids is available at [chainlist.org](https://chainlist.org/).

* Assigning consecutive numbers instead of names creates additional work around an additional service and is somewhat unfriendly to ad-hoc dev networks and local networks.
* One reviewer suggested that, instead of strings these should be 8-byte numbers, whose bit representation in ASCII is a human-readable string, so that no lookup service is needed.
  However, this will actually double the size of these strings on the wire (and in block headers) in mainnet, since the string "main" is only 4 bytes.
  The author feels that representing human-readable text using numbers is un-KISS.
* Size on the wire is likely not a serious concern here. Each block contains at least three TxOut's, and including each fog hint and the key images, we will always be at about 1KB per block
  at a minimum. Four to eight bytes is less than one percent of this, and this overhead should only be of concern in mainnet.

Ethereum actually has two ids like this: A network id, and a chain id.

* Peer-to-peer communication between nodes uses the network id
* Transaction signatures use the chain id

For most networks the two values are the same.
Chain ID was introduced in [EIP-155](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-155.md) when the Ethereum Classic fork occurred, to prevent mixing of these two networks.

* Putting the chain id within the transaction signature makes it easier for an off-line signer to
  have visibility on what network the transaction is submitted to. If your transaction is submitted
  via a proxy, the proxy would control which network id is expected.
* In the status quo, the off-line signer has no visibility into this unless the proxy pipes an attested
  connection through to them, which isn't possible for some types of off-line signers.

In the Cosmos ecosystem, a [chain registry](https://github.com/cosmos/chain-registry) is hosted on github.
Chains have both "names" which are human-readable strings, and "ids" which are also strings.

Cosmos registry enforces some chain id [best practices](https://github.com/cosmos/cosmos-sdk/issues/5363).
They expect chain-ids to match a pattern `[-a-zA-Z0-9]{3,47}`, otherwise they replace those chain-ids with a hash.
This resembles the rules we proposed except that we are case-sensitive and only lower-case letters are allowed.

# Prior art
[prior-art]: #prior-art

* [EIP-155](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-155.md)
* [Cosmos chain registry](https://github.com/cosmos/chain-registry)
* [Cosmos chain id best practices](https://github.com/cosmos/cosmos-sdk/issues/5363)

# Unresolved questions
[unresolved-questions]: #unresolved-questions

None at this time.

# Future possibilities
[future-possibilities]: #future-possibilities

In the future, clients like `mobilecoind` and `full-service` could check that ledgers that they download have the expected chain id.

More parts of fog could also check the chain id during their operations.
