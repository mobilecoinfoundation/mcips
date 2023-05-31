- Feature Name: `dcap_attestation_evidence`
- Start Date: 2023-05-31
- MCIP PR: [mcips #65](https://github.com/mobilecoinfoundation/mcips/pull/65)
- Tracking Issue: [mobilecoin #3119](https://github.com/mobilecoinfoundation/mobilecoin/issues/3119)

# Summary
[summary]: #summary

Support for verifying both EPID and DCAP quotes in the same codebase.

# Motivation
[motivation]: #motivation

We want to provide uninterrupted client support of the MobileCoin services when
the services transition from EPID to DCAP attestation.

In order to support this uninterrupted transition, clients will need to have
logic for verifying both the EPID and DCAP flavors of attestation evidence.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

For the current EPID attestation the client provides the expected measurements
and advisories to a verifier. The verifier then checks the evidence against the
expected measurements and advisories.

A _Measurement_ is either a `MRENCLAVE` value or a `MRSIGNER` value set.

A `MRENCLAVE` value is a hash that uniquely identifies the enclave's code
binary. A `MRENCLAVE` measurement is a strict comparison, any change in the
enclave's code binary will result in a verification failure.

A `MRSIGNER` is a more lenient verification which allows for newer versions of
an enclave's code binary to be used. A `MRSIGNER` value set consists of three
values:

1. A hash that uniquely identifies the entity or group that signed the enclave.
   This is often refered to as the `MRSIGNER` value, not to be confused with the
   "`MRSIGNER` value set" being described here.
2. The enclave's security version number. This number is incremented when the
   enclave is updated with security fixes. This is refered to as the ISV SVN. An
   enclave that has a higher security version number will still pass
   verification as it is assumed to be more secure than the previous security
   version.
3. The enclave's product version number. This number varies based on the
   enclave's purpose. For example fog view has a different product ID than
   consensus. This is often refered to as the ISV product ID.

Advisories are a list of advisory IDs that an enclave is allowed to have. If
there is an advisory ID in the evidence that is not in the allowed list, the
verification will fail.

The verifier needs to be able to interpret the evidence and verify that the
evidence meets the requirements of the expected measurements and advisories.

The verifier also ensures that the evidence is signed by a trusted party. This
signature is provided in an X509 certificate chain that is verified by a trust
root.

The MobileCoin enclaves will contain the enclave's public key in the evidence.
The verification of the evidence proves the validity of the public key. A common
output of verifying an enclave's evidence is the enclave's public key.

```mermaid
flowchart TD
    C[Client] -->|Expected Measurements\nAdvisories| V[Verifier]
    subgraph EPID Specific
    V --> Vy{Verify}
    T[Trust Root] --> Vy
    E[Evidence] --> Vy
    Vy --> |No| F[Fail]
    Vy --> |Yes| Va[Valid Report]
    end
    Va --> |Get Enclave Key| K[Key]
    F --> Er[Error Handling]
```

From the application's perspective, the main difference between EPID and DCAP is the provided `Evidence` format.

Based on the diagram above we should be able to create a similar flow for DCAP. The evidence will differ and will be the deciding factor on which flow to take. We'll propagate the evidence with the client measurements down to the respective verifier.

```mermaid
flowchart TD
    C[Client] -->|Expected Measurements\nAdvisories| Ki
    E[Evidence] --> Ki{Evidence Kind}
    Ki --> Ev[EPID Verifier]
    subgraph EPID
    Ev --> EVy{Verify}
    EVy --> |Yes| Er[Valid EPID Report]
    end
    Ki --> Dv[DCAP Verifier]
    subgraph DCAP
    Dv --> DVy{Verify}
    DVy --> |Yes| Dr[Valid DCAP Report]
    end
    DVy --> |No| F[Error]
    EVy --> |No| F[Error]
    Er --> |Get Enclave Key| K[Key]
    Dr --> |Get Enclave Key| K[Key]
```

In order for this to work the client's expected measurements and advisories will need to be formatted the same.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## Advisories

The advisories are separated into two categories:

1. Software hardening: These are advisories that are mitigated by the software
   implementation.
2. Config: These are advisories that are mitigated by the specific hardware
   configuration. For example disabling of hyperthreading, disabling of onboard
   GPU, updates to microcode, etc.

The client provides two lists of advisory IDs, one for each category. The lists
contain the advisory IDs that the client is willing to accept. If the evidence
contains an advisory ID that is not in the list, the verification will fail.

At the time of this writing no config advisories are allowed. This means that clients expect all advisories to be mitigated by the correct configuration of the hardware. 

For software hardening advisories, the only way to ensure the advisory has been
mitigated is through the software implementation. This means that the evidence
the enclave provides will not be able to prove that the advisory has been
mitigated. Thus the evidence will list all software advisories which the enclave
may be suseptable to. It is up to the client to know which versions of the
enclave has mitigated which advisories.

For example if a new software advisory "INTEL-SA-12345" is published, enclave
version _n_  may not have mitigated this advisory. The client should *not* list
"INTEL-SA-12345" in the list of allowed advisories when talking to enclave
version _n_. The the client would provide "INTEL-SA-12345" for enclave version
_n+1_, assuming enclave version _n+1_ provides necessary software mitigations for the
advisory. The client would still not provide "INTEL-SA-12345" for enclave version _n_ since it does not have the necessary mitigations.

## Combined Measurement and Advisory Format

In JSON format for the combined measurment and advisories are of the form:

`MRENCLAVE`
```json
{
  "MRENCLAVE": "3370f131b41e5a49ed97c4188f7a976461ac6127f8d222a37929ac46b46d560e",
  "mitigated_config_advisories": ["INTEL-SA-XXXXX"],
  "mitigated_hardening_advisories": ["INTEL-SA-00334", "INTEL-SA-00615"]
}
```

`MRSIGNER`
```json
{
  "MRSIGNER": "2c1a561c4ab64cbc04bfa445cdf7bed9b2ad6f6b04d38d3137f3622b29fdb30e",
  "product_id": 1,
  "minimum_svn": 5,
  "mitigated_config_advisories": ["INTEL-SA-XXXXX"],
  "mitigated_hardening_advisories": ["INTEL-SA-00334", "INTEL-SA-00615"],
}
```

## Evidence Format

The term "Evidence" is used in this MCIP to refer to all the attestation data
that needs to be verified by the client. The terminology is  inconcistent in the
EPID and DCAP documentation. The EPID documentation refers to the data as
"report". While the DCAP documentation refers to the data as "quote" and
"collateral".

### EPID

The EPID evidence contains the following items:

- Signature over the HTTP body
- x509 certificate chain
- HTTP body (Attestation Verification Report)

### DCAP

The DCAP evidence contains the following items:

- Quote (Contains x509 certificate chain)
- Collateral
  - Certificate revocation list (CRL) for trust root
  - Certificate revocation list (CRL) for Quote x509 certificate chain
  - Trusted computing base (TCB) info
  - TCB x509 certificate chain
  - Quoting Enclave (QE) identity
  - QE x509 certificate chain

## Evidence Access

The entirety of the evidence is retrieved from the enclave host on a running
server. The enclave host will cache the evidence for the hosted enclave and
provide this cached evidence to any clients. The caching provides a reduced load
on the attestation service as well as robustness against transient network
issues between the server and the attestation service.

Since the enclave host communicates with both the enclave and the attesation
service, the enclave host will only need to support one of EPID or DCAP. The details of how the enclave host caches and retrieves the evidence from the attestation service is out of scope for this MCIP.

The clients only need to communicate with the enclave host to retriev the evidence. The trust root that the client has will be used to verify the evidence. The client will key off the type of evidence to determine how to verify the evidence.

```mermaid
sequenceDiagram
    Client->>+Enclave Host: Request attestation evidence
    box Running Server
    Participant Enclave Host
    Participant Enclave
    Participant Quoting Enclave
    end
    autonumber
    opt Evidence Not Cached
        Enclave Host->>+Enclave: Request enclave quote
        Enclave->>+Quoting Enclave: Enclave evidence
        Quoting Enclave->>+Enclave: Quote
        Enclave->>+Enclave Host: Quote
        Enclave Host->>+Attestation Service: Quote
        Attestation Service->>+Enclave Host: Attestation evidence
    end
    Enclave Host->>+Client: Attestation evidence
    Client->>Client: Verify attestation evidence
```

# Drawbacks
[drawbacks]: #drawbacks

- Many of the current interfaces would need to change. Currently they pass around a verifier with the expected measurements and advisories already set. This would need to be changed to pass around the expected measurements and advisories.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

TBD

# Prior art
[prior-art]: #prior-art

[Open Enclave](https://openenclave.io/sdk/) also keys off of the evidence type
to determine which path to take to verify the evidence. However, the Open
Enclave SDK does not provide a way to specify the expected measurements and
advisories. It uses a callback method to allow callers to further customize the
verification process
[`oe_verify_claims_callback_t`](https://openenclave.io/apidocs/v0.18/verifier_8h_a1894fea8701448f5cfcf18736faad3c3.html#a1894fea8701448f5cfcf18736faad3c3)