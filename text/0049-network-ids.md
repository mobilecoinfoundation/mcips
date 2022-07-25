- Feature Name: `network_ids`
- Start Date: 2022-07-25
- MCIP PR: [mobilecoinfoundation/mcips#0049](https://github.com/mobilecoinfoundation/mcips/pull/0049)
- Tracking issue: None Yet

# Summary
[summary]: #summary

Each running mobilecoin consensus network will be started with a string, the "network id".

Clients which make requests to consensus MAY pass the expected network id with their request.
If this doesn't match the server, an error is returned, which includes the server-side network id in the response.

# Motivation
[motivation]: #motivation

Two important and competing engineering goals for all financial software are:

* Enable Rigorous Testing: Properly test the code before it goes live in a test environment that matches prod as closely as possible.
* Prevent Mixing of Test and Prod: Make it hard for e.g. users to think they are talking to testnet when they are actually talking to production and vice versa.

The basic issue is, we DO want the test servers and clients to match production as closely as possible, so that we test the code and the code paths
that we will actually ship. On the other hand, if there is NO way for the software to distinguish the two environments, then there is no way to have
safeguards that prevent e.g. accidental sending of money or minting on mainnnet.

To date, we have mostly tried to solve the "mixing" problem in the following way:

* Testnet, Prod, and Dev staging environments all have different attestation trust roots
* Clients are either built with a hard-coded trust root, or at packaging time are supplied an appropriate css file.
* Clients will fail attestation if they connect to the wrong network type, so as a result they cannot mix networks.

There are a few problems with this:

* When mixing networks occurs, an attestation failure error message results, but this error message is very hard to understand.
* Clients typically don't clearly communicate to the user which network they are on. So this doesn't ultimately fix the problem,
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

* "Network id"-aware clients is a common pattern used by other blockchains to help clients (and users) be aware of what network they are talking to.
* Network id can be easily displayed to the user if desired, which is something that none of our clients actually do today.
   * For example, when using Metamask, a user can select whether to talk to mainnet, an L2 network, or a test network, via a dropdown menu, and see what network they are currently talking to here.
     Even if we don't support dynamic switching of networks in a particular client, we can still display the network id somewhere, so that the users can discover TEST vs. PROD easily, without
     changing any of the other codepaths that we want to test.
   * This could be as simple as, if the network id is not prod, then the network id is displayed in a textbox in the upper right corner. Or, the network can be inspected from somewhere in the menu.
   * So instead of "TESTGOOG", you still see the ticker "GOOG", but "TEST" appears in the user interface somewhere else.
* All of our dev, test, and prod environments currently have names that can naturally be adopted for this purpose.
* Network id can be our primary method of "de-conflicting" and preventing mixing of networks.
* We won't have to solve this again every time we add a new feature to consensus, and we can simplify future designs by reducing requirements. 
* This has the benefit that the error messages when networks are mixed will be consistent and much clearer.

Note that while we are partly motivated by reducing complexity and easing testing around MCIP #45, it is possible that we will do MCIP #45 as originally proposed
and this proposal -- they are orthogonal, strictly speaking. We could decide to adopt this proposal as a secondary layer of defense against mixing problems, or in service
of a new consensus feature in the future.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

A "network id" is a string. The string must be a valid DNS label:

* Up to 63 bytes long
* Valid characters are `a-z`, `0-9`, and `-` (hyphen).
* You can't specify a hyphen at the beginning or end of a label.

Consensus servers are passed this string on startup by either a flag `--network-id` or an environment variable `MC_NETWORK_ID`.
* It is an error if it is missing.
* This is called the "configured network id".

Client facing APIs are extended to include an OPTIONAL "expected_network_id" argument.
* If present, the untrusted server code checks this against the configured network
  id immediately upon deserializing the response.
* If it is not a match, then an error is returned to the client which includes the configured network id.

This argument is optional to support backwards compatibility.

Clients SHOULD:

* Be aware of the id of the network they are configured to talk to
* Make this information discoverable to the user in some appropriate way. It is primarily important to highlight the fact that things are in dev or test networks rather than
  to add visual noise or untranslated strings in production -- it may be fine to display nothing in prod as a way of distinguishing it, depending on the application.
* Take advantage of the `expected_network_id` parameter to help prevent mistakes

When the client is a cross-chain bridge, configuration should be organized so that network-id is determined at the same time that the network-id for the other chain is determined,
and there should be a single source of truth for this association. It should be made very hard to accidentally configure the bridge to talk to e.g. an Ethereum testnet and also Mobilecoin Prod.

For example, either a hard-coded table or a configuration toml file could be created for the bridge which associates to a Mobilecoin network id:

* URLs of mobilecoin services
* URLs of any other chain services
* Network ids of any other chain networks
* Contract ids for any relevant contracts

This is better than each of these being independent configuration parameters to the software. They should all be changing in lock-step.

# Drawbacks
[drawbacks]: #drawbacks

This adds more configuration to the network.

Clients may send a few more bytes with each request.

This proposal does NOT, at this time, propose to hash the network id into the responder id. While that would logical, it is also an enclave change.

This means that if there is a configuration problem, it's possible that not all nodes in the network have the same network id, but in this case,
we can fall-back to the attestation trust roots to prevent a problem, and no harm was done relative to status quo. Eventually users who connect
to the misconfigured servers will be unable to make progress and report an error, and the configuration can be fixed.

This does not, at this time, propose to make the same API change in the Fog servers. Putting this in the Fog servers would mean that balance checks
cannot succeed without getting this error if there is mixing of test and prod, so Fog clients would fail faster in case of mixing networks.
But just putting it in consensus seems higher priority because that prevents loss of funds, and minting problems.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

The primary alternative is, do nothing:

* Continue to use ad-hoc configuration differences like attestation trust roots and token ids as the primary means to distinguish dev, test, prod environments.
* Continue to introduce more such ad-hoc configuration differences as more features are added.
* Cross-chain bridges primarily lean on attestation roots to avoid e.g. accidentally minting on the wrong network, but it is a bit harder to manage the configuration,
  because attestation roots live in files.

Another alternative: The Ethereum [JSON-RPC API](https://ethereum.org/en/developers/docs/apis/json-rpc/) returns the network id as a string, but in practice all the ids are actually integers.
The mapping from network names to ids is available at chainlist.org.

* Integers may be a bit smaller on the wire.
* Assigning consecutive numbers instead of names creates additional work around an additional service and is somewhat unfriendly to ad-hoc dev networks and local networks.
* Size on the wire is likely not a serious concern. But if it were, another approach would be to send a hash of the network id string, so that we can ensure that the on-the-wire size is bounded,
  and the error message in case of a mismatch sends back the full string. OTOH it seems that in real life the hash would probably not be shorter than the string network id.

Ethereum actually has two ids like this: A network id, and a chain id.

* Peer-to-peer communication between nodes uses the network id
* Transaction signatures use the chain id

For most networks the two values are the same.
Chain ID was introduced in EIP-155 when the Ethereum Classic fork occurred, to prevent mixing of these two networks.

* Putting the chain id within the transaction signature makes it easier for an off-line signer to
  have visibility on what network the transaction is submitted to. If your transaction is submitted
  via a proxy, the proxy would control which network id is expected.
* In the status quo, the off-line signer has no visibility into this unless the proxy pipes an attested
  connection through to them, which isn't possible for some types of off-line signers.
* Putting a chain id in the transaction signature cannot be done without an enclave change if we want this.

# Prior art
[prior-art]: #prior-art

* [EIP-155](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-155.md)

# Unresolved questions
[unresolved-questions]: #unresolved-questions

What exact network id strings should be used for the existing mainnet and testnet?

# Future possibilities
[future-possibilities]: #future-possibilities

In the future, we could:

* Specify that Fog network servers also have an `expected_network_id` parameter and `--network-id` configuration.

In a future enclave update, we could:

* Specify that network-id is part of the blockchain-config, and so hashed into the responder id, so that consensus nodes will fail to attest if they disagree about the network id.
* Specify that the network-id is under the transaction signature, or, another parameter like chain-id which is under the transaction signature.
* Decide that there should be a chain id which is actually part of the block header.
