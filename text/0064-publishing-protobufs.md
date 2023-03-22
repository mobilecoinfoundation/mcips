- Feature Name: Publishing Protobuf Definition Files
- Start Date: 2023-03-22
- MCIP PR: [mobilecoinfoundation/mcips#0064](https://github.com/mobilecoinfoundation/mcips/pull/0064)
- Tracking Issue: [mobilecoinfoundation/mobilecoin/3071](https://github.com/mobilecoinfoundation/mobilecoin/issues/3071)

# Summary
[summary]: #summary

A more ergonomic way for clients to use the 
[Protobuf](https://protobuf.dev/) definition files used for communicating with
consensus and fog services.

# Motivation
[motivation]: #motivation

Protobuf definition files are often used to generate code for communicating with
services. The Protobuf files define the binary data representation and the
generated code provides a way to convert between a common code interface and the
binary data representation.

The current Protobuf definitions for communicating with fog and consensus are
defined in the main 
MobileCoin Foundation git [repo](https://github.com/mobilecoinfoundation/mobilecoin). 

Most clients end up submoduling the MobileCoin Foundation repo in order to
generate code at build time.  The MobileCoin Foundation repo is fairly
large and submoduling it to gain access to a few files is not very ergonomic.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Protobuf definition files are made available in the following ways:

1. [Buf Schema Registry](#buf-schema-registry)
2. [Rust crates](#rust-crates)
3. [The Protobuf git repo](#the-protobuf-git-repo)

## Buf Schema Registry

The [Buf Schema Registry](https://buf.build/explore/) is an online hosting
platform of Protobuf defintions. It provides documentation hosting and a command
line tool to detect breaking changes in Protobuf definitions.

The MobileCoin Foundation Protobuf files are available at
<https://buf.build/mobilecoinfoundation>.


For most clients, the Buf Schema Registry is the recommended way to use the
MobileCoin Foundation's Protobufs.

## Rust Crates

The MobileCoin Foundation Protobuf defintions are available via
[Rust](https://www.rust-lang.org/) crates.  The crates use
[prost](https://docs.rs/prost/latest/prost/) generated messages. Both
[tonic](https://docs.rs/tonic/latest/tonic/) and
[grpcio](https://docs.rs/grpcio/latest/grpcio/) are supported for the services.


These crates can be found on
<https://crates.io> by searching for one of the following:

- [mc-*-grpcio](https://crates.io/search?q=mc-*-grpcio) for rust
  [grpcio](https://docs.rs/grpcio/latest/grpcio/) services.
- [mc-*-messages](https://crates.io/search?q=mc-*-messages) for
  [prost](https://docs.rs/prost/latest/prost/) messages.
- [mc-*-tonic](https://crates.io/search?q=mc-*-tonic) for
  [tonic](https://docs.rs/tonic/latest/tonic/) services.

Building the crates requires the client to have the Protocol Buffer Compiler,
[protoc](https://grpc.io/docs/protoc-installation/), installed.

For rust projects, the rust crates are the recommended way of using the MobileCoin Foundatin Protobuf defintions.

## The Protobuf Git Repo

Clients can directly access the Protobuf definitions via the git repo,
<https://github.com/mobilecoinfoundation/protobufs>. 

The other two methods should be prefered over direct access to the git repo, but
there are times when it is necessary to use the repo directly.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The MobileCoin Foundation Protobuf defintions are located at
<https://github.com/mobilecoinfoundation/protobufs>. When a series of git
changes are deemed ready to release, the new versions will be published to
<https://buf.build/mobilecoinfoundation> and crates published to <https://crates.io>. 

The Protobuf messages are placed in separate files from the Protobuf services.
This makes it easier for one to build just the message types without needing to
worry about the service implementation.

Each Protobuf API will be placed in it's own directory with the
following layout:

```console
mobilecoinfoundation
└── <name of api>
    ├── grpcio
    │   ├── build.rs
    │   ├── Cargo.toml
    │   ├── protos
    │   │   └── services.proto (Symlink to <version>/services.proto)
    │   ├── README.md
    │   └── src
    │       └── lib.rs
    ├── messages
    │   ├── build.rs
    │   ├── Cargo.toml
    │   ├── protos
    │   │   └── messages.proto (Symlink to <version>/messages.proto)
    │   ├── README.md
    │   └── src
    │       └── lib.rs
    ├── tonic
    │   ├── build.rs
    │   ├── Cargo.toml
    │   ├── protos
    │   │   └── services.proto (Symlink to <version>/services.proto)
    │   ├── README.md
    │   └── src
    │       └── lib.rs
    └── <version> (of the form `v1`, `v2`, etc.)
        ├── messages.proto
        └── services.proto
```

The `<name of api>` is used to separate different Protobuf APIs. For example
this might be `attestatoin` to define only the attestation Protobuf API.

The `grpcio` directory holds the files to build the rust crate
`mc-<name-of-api>-grpcio`. This rust crate is for building the services that
work with [rust grpcio](https://docs.rs/grpcio/latest/grpcio/).

The `messages` directory holds the files to build the rust crate
`mc-<name-of-api>-messages`. This rust crate is for building the
[prost](https://docs.rs/prost/latest/prost/) messages. 

The `tonic` directory holds the files to build the rust crate
`mc-<name-of-api>-tonic`. This rust crate is for building the services that work
with [tonic](https://docs.rs/tonic/latest/tonic/). 

The `<version>` directory is used to isolate breaking changes to the Protobuf
API. All updates to `v1` should continue to work with the first published
release of `v1`. If a breaking change is necessary it should be made in a
subsequent `v2` directory.

Symlinks are used from the rust crates back to the `<version>` directory to
ensure that the rust crates build on <https://crates.io>.

# Drawbacks
[drawbacks]: #drawbacks

- Using the Buf Schema Registry puts reliance on a third party. It is currently
  listed as "in beta", meaning it may have changes coming that we may need to
  account for as time progresses. 

- Housing the rust crates in the Protobuf repo may encourage housing other
  language versions in the repo.

- Separaing the Protobuf definitions from the main repo can make it less
  ergonomic to develop newer APIs. Developers may need to temporarily redirect
  crate dependencies to current inflight API changes.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

Publishing the Protobuf API definitions makes it easier for other code bases to
communicate with Consensus and Fog. Regardless of which language is used,
clients can generate code to communicate using the API without having to find
the files in the MobileCoin Foundation repo.

The Protobuf APIs can be versioned independent of the Consensus and Fog
releases. This makes it easier for a client to see when pertenent changes
occur.

Having a dedicated repo makes it easier to have CI checks to prevent breaking
changes to the Protobuf APIs.  Currently it is left to the developers to ensure
changes are non breaking. The main MobileCoin Foundation repo already has quite
a lot of CI steps adding another step for Protobuf breaking changes is likely to
fall to the wayside.

# Prior art
[prior-art]: #prior-art

None

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- How do we version the published crates?

# Future possibilities
[future-possibilities]: #future-possibilities

Unknown