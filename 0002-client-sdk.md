---
layout: default
title: Fabric-X SDK
nav_order: 3
---

- Feature Name: Fabric-X SDK
- Start Date: 2026-03-09
- RFC PR: -
- Fabric-X Component: fabric-x-sdk
- Fabric-X Issue: -

# Summary
[summary]: #summary

A modular set of building blocks that can be used to develop client applications, endorsers, and custom components for Fabric-X. Some components are designed to be compatible with classic fabric as well for easy reuse.

# Motivation
[motivation]: #motivation

Fabric-X does not have the concept of chaincode, so transactions have to be created and endorsed in a different way. The new programming model has endless possibilities to create read/write sets, but these possibilities have so far not been easily accessible to developers.

The most mature solution to create and submit Fabric-X transactions is the [Fabric Smart Client](github.com/hyperledger-labs/fabric-smart-client/). It excels in interactive peer-to-peer flows, for instance for privacy-friendly compliant token transactions in combination with the [Fabric Token SDK](github.com/hyperledger-labs/fabric-token-sdk/). The extensive feature set of the Fabric Smart Client is not necessary for some use cases, and there is a need to expand the number of lightweight options for Fabric-X client applications.

The community should have all the tools it needs to build lightweight, tailored solutions. Examples may include:

- Fabric-X block explorers
- event-based middleware or notification services
- an endorser that can execute EVM-based (or any other kind of) transactions
- a wrapper for fabric-style chaincodes
- Fabric Smart Client will have a simplified Fabric-X platform by using this sdk
- and whatever the community comes up with. 

Some of the examples above are already under development. A shared library will reduce the maintenance burden on these projects, while also making it easier to develop new tools and client applications.

The modular SDK form will make it feasible to write your own endorsers, with whatever execution engine you want - for both classic fabric networks and for Fabric-X. One of the main goals for the SDK is to become the foundation to introduce classic chaincodes to Fabric-X; allowing the community to build this feature. It's an important ingredient for a migration path towards Fabric-X, and can even be expected to remain a core way of interacting with the ledger. Some features like private data collections might be hard to replicate, but in exchange for that there is a huge flexibility to go beyond the limitations of the chaincode paradigm.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The Fabric-X SDK contains a set of tools that will help you build client applications and custom components for Fabric-X.

The main components are:

- *Identity*:
  - Load identities from an MSP directory
  - Signers for Fabric-X transactions and endorsement
- *Synchronizer*:
  - a client for the sidecar Delivery endpoint
  - synchronize from a specific block
- *Parser*:
  - marshalers and unmarshalers for blocks and transactions
- *Endorser*:
  - world state storage
  - simulation store
- *Query client*:
  - a client for the committer query service
- *Submitter*:
  - Endorsement client (fabric 3 style)
  - build and sign Fabric-X transactions
  - ordering service client
- *Examples*:
  - a generic endorser
  - ... 

**NOTE**: to be discussed: some of these might be better situated in [Fabric-X-common](https://github.com/hyperledger/Fabric-X-common).

For example, if you were building a block explorer, you would take the Synchronizer, load it with an Identity and point to the committer sidecar. With help from the Parser, you extract the information you want to store from the blocks that are coming in through the Synchronizer. Then you build your application around it.

If you were building a chaincode endorser, you would use the same elements, as well as the Endorser world state storage and simulation store. You'll expose the ProcessProposal API. To complete the picture, you could equip your client applications with the Endorsement client and Submitter.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The SDK is not meant to be used by Fabric-X core components. It will contain simple clients to the Fabric-X components:
- orderer
- committer query service
- committer sidecar.

Additionally it includes some building blocks to create generic endorsers, to bridge the gap with fabric 3 and enable community innovation.

Ideally, existing and in-development solutions will start to depend on these libraries. Developers of Fabric Smart Client, EVM compatibility, block explorers are all invited to contribute.

The first implementation might be basic, but the clients should evolve to include more production-grade requirements like retries, failover, and intelligent peer selection. We would like to leverage the work that has been done in the Fabric Smart Client, Fabric-X-committer, and in existing Fabric SDKs.

# Drawbacks
[drawbacks]: #drawbacks

When adding a new repository, there is always the risk of the maintenance burden becoming too high. In this case, that can be mitigated with the commitment from some teams to use and contribute to the library. Next to that we will keep the features limited to what we use.

# Rationale and alternatives
[alternatives]: #alternatives

- There is some potential overlap with Fabric-X-common. Some basic building blocks proposed in this RFC should probably be part of that. The Fabric-X-sdk should focus on a higher level of abstraction, with more opinionated and ready-to-use tooling. 
- The idea for this library was born out of the concept of custom endorsers. We have considered publishing a generic endorser, that can be forked to include custom execution engines. That might be easier to do, but more difficult to maintain. The modular SDK approach makes it more broadly usable, and with that more valuable.

If we don't start an SDK now, we will continue to develop similar tooling on our respective islands. And it would continue to be difficult for builders in the community to work on developer tooling, to migrate their networks, and to start using Fabric-X.

# Prior art
[prior-art]: #prior-art

- [Fabric Smart Client](github.com/hyperledger-labs/fabric-smart-client/), already mentioned, is great at what it does but we also need a lighter, more modular toolset.
- [Fabric Gateway](https://hyperledger.github.io/fabric-gateway/) is the SDK for Fabric 2 and 3. As the name implies it relies heavily on the Gateway in the peer, which is not part of Fabric-X. We should learn from the gateway and reuse code where relevant.

# Testing
[testing]: #testing

- Unit tests for all components.
- Integration tests using the Fabric-X-committer test container (or a real network with microservices) and fabric 3.
- A basic in-process test fabric "network" will be provided for quick local integration tests.

# Dependencies
[dependencies]: #dependencies

The code will depend on:

-  `fabric-protos-go-apiv2`
-  `fabric-lib-go`
-  `google.golang.org/grpc`
-  `google.golang.org/protobuf`

# Unresolved questions
[unresolved]: #unresolved-questions

- What should be part of Fabric-X-common vs this repo?
- Can we already reuse components directly from Fabric-X?
- Are there any essential features that are missing from this proposal?
- Which other tests do we need?
