- Feature Name: dcap-evidence-in-block-metadata
- Start Date: 2023-12-01
- MCIP PR: [mobilecoinfoundation/mcips#68](https://github.com/mobilecoinfoundation/mcips/pull/68)
- Tracking Issue: [mobilecoinfoundation/mobilecoin#2373](https://github.com/mobilecoinfoundation/mobilecoin/issues/2373)

# Summary
[summary]: #summary

Provide DCAP attestation evidence in the block metadata. This is the compliment
to the Attestation Verification Report (AVR) used in EPID.

# Motivation
[motivation]: #motivation

EPID SGX hardware is being phased out. The newer SGX machines use DCAP as the
attestation protocol. This MCIP is to provide the DCAP version of the evidence
in the block metadata so that chain integrity can be verified for blocks created
on the newer hardware.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The `BlockMetatdataContents` in block versions 4 or greater contain either a
`DcapEvidence` or a `VerificationReport`.
(The `BlockMetatdataContents` was added in [MCIP-0043](0043-block-metadata.md)).
The `DcapEvidence` contains attestation evidence returned by the DCAP
attestation process. The `VerificationReport` contains the attestatoin evidence
returned by the EPID attestation process.

In general, block version 4 or greater is expected to only contain
`DcapEvidence`, but the `VerificationReport` is included for backwards
compatibility.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The `BlockMetadataContents` uses a
[`oneof`](https://protobuf.dev/programming-guides/proto3/#oneof) to contain
either a `VerificationReport` or a `DcapEvidence`. This field is named
`attestation_evendence` to be generic enough to contain either type of evidence.

## Protobuf schema

```protobuf
message BlockMetadataContents {
    /// The Block ID.
    BlockID id = 1;

    /// Quorum set configuration at the time of externalization.
    QuorumSet quorum_set = 2;

    // The attestation evidence for the enclave which generated the signature.
    oneof attestation_evidence {
        external.VerificationReport verification_report = 3;
        external.DcapEvidence dcap_evidence = 5;
    }

    /// Responder ID of the consensus node.
    ResponderId responder_id = 4;
}
```

# Drawbacks
[drawbacks]: #drawbacks

This requires a block version bump since it changes the `BlockMetadataContents`.
Consumers that only know about block version 3 will be unable to verify
`BlockMetadataContents` or the attestation evidence it contains.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

The `VerificationReport` and `DcapEvidence` were disparate enough that it didn't
seem feasible to try and convert a `DcapEvidence` into a `VerificationReport`.

# Prior art
[prior-art]: #prior-art

None at this time.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

None at this time.

# Future possibilities
[future-possibilities]: #future-possibilities

None at this time.
