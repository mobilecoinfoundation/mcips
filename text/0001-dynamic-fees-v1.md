- Feature Name: Dynamic Fees V1 (Configurable Fees)
- Start Date: (2021-04-20)
- MCIP PR: [mobilecoinfoundation/mcips#1](https://github.com/mobilecoinfoundation/mcips/pull/1)
- MobileCoin Epic: MCC-2301

# Summary
[summary]: #summary

Decrease the minimum fee to 0.0004 MOB (400uMOB, which is 0.02 USD/tx at a 50 USDMOB exchange rate) per board instruction, allow node operators to override the hard-coded minimum fee, and prevent divergent fees by including fee agreement in the node-to-node attestation / key exchange protocol.

# Motivation
[motivation]: #motivation

MobileCoin launched with a hard-coded minimum fee of 0.01 MOB. When this constant was set, the value of MobileCoin was assumed to be slightly below 1 USD per coin, resulting in a per-transaction fee of under 0.01 USD (1c). In the intervening 6 months, the value per-coin increased to 50 USD, resulting in a per-transaction fee of 0.50 USD.

This fee is too high to reasonably support the cannonical use-case---purchasing a cup of coffee---and we would like to lower the fee to $0.04 to bring it back in line with that use-case.

However, this creates a different problem. If the consensus system is roughly capable of completing 50tps (transactions per second), then the cost of conducting a denial of service attack is directly related to the size of the fees. At 0.50 USD/tx, consuming the entire consensus power of the network would cost over 2 MUSD/day.

At the desired 0.04 USD/tx, however, the cost of conducing a denial of service attack would be less than 175 kUSD/day. This may seem like a lot of money, but recall that at peak trading on FTX, 80 MMOB or 4 GUSD changed hands, which gives some sense of scale to those costs (i.e. they are too low).

The second issue is that changing the hard-coded fee requires going through the full release cycle: testnet deployment and testing, mainnet enclave signing, client updates to trust the new enclave, a grace period for client updates, then finally a coordinated mainnet deployment by operators.

In our current environment, it takes at least a month to change the fee, and the fees are simultaneously too high and too low. This must change.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

When a node operator starts the consensus service, it may optionally provide a configuration option or command-line argument overriding fee. The fee will be appended (as a string) to the existing key exchange "prologue" (a series of bytes that both sides agree upon in advance) when performing node-to-node attestation.

If two nodes with different fees attempt to communicate, the result will be an `AeadError` during the handshake phase.

The currently-configured fee should be made available to consensus clients directly by extending the `BlockchainAPI.GetLastBlockInfo` response. This gRPC call is already polled by thick clients to orient themselves with relation to the state of the network.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The `consensus_service` application will be extended to add a new `--minimum-fee` command-line argument and `MC_MINIMUM_FEE` environment variable. These values will be filled in order of precedence by the command-line option, environment variable, and hard-coded minimum. Failure to parse the value being used will result in an error, and will not "fall through" to the next source. The value will contain a string indicating the quantity and SI-scale of the minimum fee. Fees with out units will be considered to be denominated in picoMOB. Therefore, all these values are equivilent fees:

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

The primary drawback to this approach is that it requires a restart of a quorum of the network in order to adjust to a new fee. In the intended common-case (low fees generally, higher fees in response to an active denial-of-service), this will entail restarting the network twice, and when a node is restarted, any pending transactions which did not make it into the current SCP slot will be discarded. That said, if node operators follow the existing enclave upgrade procedures (using firewalls to prevent new inbound transactions, allowing the network to reach a quiescent state with no pending values, then restarting), no pending transactions will be lost.

Another drawback is the increased debugging difficulty in the event of a failed attestation. In the current system, a mismatched `ResponderId` (prologue) is the most likely source of an `AeadError` during authentication (the other cause would be an active MiTM attack replacing a public key). In the proposed system, because a configured fee is appended to the `ResponderId` to produce the prologue, a mismatch of either value would cause the `AeadError`, and thus both must be checked.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

## Alt: Just Lower The Fee

The first alternative considered was simply lowering the fees to 400uMOB, or 0.02USD @ 50USD/MOB. As noted above, while this accomplishes the initial goal of lowering the fees, it does not provide the ability to raise them without another enclave upgrade, which is costly, time-consuming, and will take upwards of 120 days to complete.

## Alt: Fee Schedule

The next alternative was using a "fee schedule", or "fee calendar"---that is, rather than having a single fee which is constructed at startup time, use a configuration file which provides a series of "fee ranges" `(fee, startBlock, endBlock)` that are hashed into the key exchange.

This alternative scheme does have the advantage of allowing a reconfiguration to "script" the denial-of-service response, and it's subsequent unwind, in a single restart. The devil, however, is in the details: how should pending and future transactions be handled?

This is actually quite tricky to get right: obviously you want to immediately reject any transactions which do not meet the currently configured fee requirements, but it's entirely possible that your node is already queuing transactions when this comes in.

In this scenario, a client submits a transaction, and it goes into a queue behind hundreds of other transactions. The fee is "correct" for the current block, but the transaction will not actually be entertained for acceptance until after the fee has been raised, and the fee is no longer valid.

Choosing to simply accept that if any node has accepted a proposed transaction at a lower fee, that fee is (or would have been) valid for all nodes is an option, but this does make the ability to enforce fees dependent on the union of all SGX functionality, which is a change from our ad-hoc norms around the use of SGX as a privacy-enhancement technology, rather than a security-critical technology.

The second option is to only check fees lazily, which will produce the scenario where a client could submit a transaction, and have it appear to "time out" later, because it was accepted into the queue at a low fee, then later judged to have an insufficient fee.

The "restart for each change" system will have similar properties for users as an emergency restart to introduce new fees---when a node is restarted without following the upgrade procedure, any pending transactions it has are lost---but is dramatically simpler to implement, and does not require special thought about validity periods of a particular transactions' fee-paid status.

## Alt: Allow Divergent Fees

The last alternative scheme is to remove agreement on minimum fee amounts outside of consensus altogether. In this version, each node can configure it's own minimum fee, and use it when verifying a transaction submitted by clients, but simply ignore the fee when treating transactions received by peers. This deviates in spirit, though not form, from the earlier "accept a node enforced its fees before proposing to the wider network" above, in that it removes consideration of fees entirely from the consensus.

In other words, if you decide the fee amount is not a criteria for a valid transaction, then the question of trusting SGX to enforce the fee is no longer a coherent objection. The nature of the alternative warrants deeper consideration and wider discussion, but that discussion would delay the immediate reduction of fees, which is believed to be a higher priority.

## Rationale

The primary rationale, then, is that this is "good enough" for right now, and we will follow this up with a "dynamic fees v2" that settles all questions properly under easier time constraints.

# Prior art
[prior-art]: #prior-art

Unfortunately, due to the limited scope of this PR, prior art is going to be difficult to discuss. There is obviously a much larger conversation to be had about Etherium gas, Bitcoin transaction fees, etc., and how MobileCoin fees differ from them, but that honestly belongs in a future MCIP. This is about doing the minimum viable to get us to a baseline fee-adjustment strategy, without compromising our ability to respond to denial of service attacks.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- There are no unresolved questions which should be resolved by this MCIP.

# Future possibilities
[future-possibilities]: #future-possibilities

One desirable end-state is a world where all transactions are algorithmicly sorted in some fair manner, and the minimum fee is set to something truly deminimus. The consensus nodes could aggregate and publish the minimum fee necessary to get a transaction onto the blockchain, queue depth, etc. This information can feed back into users who wish to propose transactions.
