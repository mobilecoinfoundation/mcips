- Feature Name: Dynamic Fees V1 (Configurable Fees)
- Start Date: (2021-04-20)
- RFC PR: [mobilecoinfoundation/rfcs#0000](https://github.com/mobilecoinfoundation/rfcs/pull/0000)
- MobileCoin Epic: MCC-2301

# Summary
[summary]: #summary

Decrease the minimum fee to 0.0004 MOB (400uMOB, which is 0.02 USD/tx at a 50 USDMOB exchange rate) per board instruction, allow node operators to override the hard-coded minimum fee, and prevent divergent fees by including fee agreement in the node-to-node attestation / key exchange protocol.

# Motivation
[motivation]: #motivation

MobileCoin launched with a hard-coded minimum fee of 0.01 MOB. When this constant was set, the value of MobileCoin was assumed to be slightly below 1 USD per coin, resulting in a per-transaction fee of under 0.01 USD (1c). In the intervening 6 months, the value per-coin increased to 50 USD, resulting in a per-transaction fee of 0.50 USD.

This fee is too high to reasonably support the cannonical use-case---purchasing a cup of coffee---and we would like to lower the fee to $0.04 to bring it back in line with that use-case.

However, this creates up a different problem. If the consensus system is roughly capable of completing 50tps (transactions per second), then the cost of conducting a denial of service attack is directly related to the size of the fees. At 0.50 USD/tx, consuming the entire consensus power of the network would cost over 2 MUSD/day.

At the desired 0.04 USD/tx, however, the cost of conducing a denial of service attack would be less than 175 kUSD/day. This may seem like a lot of money, but recall that at peak trading on FTX, 80 MMOB or 4 GUSD changed hands, which gives some sense of scale to those costs (i.e. they are too low).

The second issue is that changing the hard-coded fee requires going through the full release cycle: testnet deployment and testing, mainnet enclave signing, client updates to trust the new enclave, a grace period for client updates, then finally a coordinated mainnet deployment by operators.

In our current environment, it takes at least a month to change the fee, and the fees are simultaneously too high and too low. This must change.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

When a node operator starts the consensus service, it may optionally provide a configuration option or command-line argument overriding fee. The fee will be appended (as a string) to the existing to the key exchange "prologue" (a series of bytes that both sides agree upon in advance) when performing node-to-node attestation.

If two nodes with different fees attempt to communicate, the result will be an `AeadError` during the handshake phase.

The currently-configured fee should be made available to consensus clients directly by extending the `BlockchainAPI.GetLastBlockInfo` response. This gRPC call is already polled by thick clients to orient themselves with relation to the state of the network.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The `consensus_service` application will be extended to add a new `--minimum-fee` command-line argument and `MC_MINIMUM_FEE` environment variable. These values will be merged (environment-variable, then command-line option). This value will contain a string indicating the quantity and SI-scale of the minimum fee. Fees with out units will be considered to be denominated in picoMOB. Therefore, all these values are equivilent fees:

- 10mMOB
- 10000uMOB
- 10_000_000_000

SI units will be parsed in a case-sensitive manner following the [NIST SI Prefixes](https://www.nist.gov/pml/weights-and-measures/metric-si-prefixes). Most importantly, GMOB will be correctly pronounced as "JIG-uh MOEB" (with a soft "g").

Fractional fees (e.g. "0.01MOB") will be rejected, to avoid the potential for floating-point ambiguity in implementation. Additionally, the consensus service will reject any fees larger than 1 MOB or smaller than 0.00000001 MOB (0.005 USD @ 50 kUSD), unless the operator also includes an `--allow-any-fees` command-line option. This is intended to mitigate some of the risk of SI unit confusion (e.g. "MMOB", indicating millions of MOB, vs. "mMOB", indicating thousandths of a MOB.

The given minimum fee value, if it is provided, will be injected into the enclave at startup and stored for future use when validating inbound transactions. As is the case today, fees will be validated initially, on transaction submission.

This fee value will be injected into the attested key exchange by the enclave, by constructing a new UTF-8 string of the format: `"%s-%u"` using the existing `ResponderId`, and the u64 picoMOB-denominated representation of the new fee. This value is injected into the hashing scheme built onto the key exchange protocols in use.

Because of this, two nodes with different fees will produce different encryption keys, which will lead to a failed decyption during the key exchange setup (specifically, the initiator will receive an `AeadError` when attempting to decrypt the initial message received from the responder).

# Drawbacks
[drawbacks]: #drawbacks

There are several drawbacks to 

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?

# Prior art
[prior-art]: #prior-art

Discuss prior art, both the good and the bad, in relation to this proposal.
A few examples of what this can include are:

- For consensus and fog proposals: Does this feature exist in other systems, and what experience have their community had?
- For community proposals: Is this done by some other community and what were their experiences with it?
- For other teams: What lessons can we learn from what other communities have done here?
- Papers: Are there any published papers or great posts that discuss this? If you have some relevant papers to refer to, this can serve as a more detailed theoretical background.

This section is intended to encourage you as an author to think about the lessons from other systems, provide readers of your RFC with a fuller picture.
If there is no prior art, that is fine - your ideas are interesting to us whether they are brand new or if it is an adaptation from other systems.

Note that while precedent set by other systems is some motivation, it does not on its own motivate an RFC.
Please also take into consideration that MobileCoin sometimes intentionally diverges from common cryptocurrency features.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature before stabilization?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?

# Future possibilities
[future-possibilities]: #future-possibilities

Think about what the natural extension and evolution of your proposal would
be and how it would affect the project as a whole in a holistic way. Try to
use this section as a tool to more fully consider all possible interactions
with aspects of the project in your proposal. Also consider how this all
fits into the roadmap for the project and of the relevant team.

This is also a good place to "dump ideas", if they are out of scope for the
RFC you are writing but otherwise related.

If you have tried and cannot think of any future possibilities,
you may simply state that you cannot think of anything.

Note that having something written down in the future-possibilities section
is not a reason to accept the current or a future RFC; such notes should be
in the section on motivation or rationale in this or subsequent RFCs.
The section merely provides additional information.

