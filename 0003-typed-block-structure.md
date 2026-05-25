---
layout: default
title: Typed Block Structure and Canonical Transaction Envelope Signing
nav_order: 3
---

- Feature Name: typed_block_structure
- Start Date: 2026-05-25
- RFC PR:
- Fabric-X Component: fabric-x-common, fabric-x-orderer, fabric-x-committer
- Fabric-X Issue: https://github.com/hyperledger/fabric-x-common/issues/76

# Summary
[summary]: #summary

This RFC replaces the current byte-serialized Fabric envelope structure with a typed block and transaction envelope structure owned by `fabric-x-common`. Transactions use explicit typed variants instead of generic byte payloads, and submitter signatures are computed over deterministic ASN.1 DER bytes derived from typed transaction-envelope data. The orderer verifies the same canonical bytes, and the committer consumes typed transaction fields directly without nested envelope, payload, or header unmarshaling.

# Motivation
[motivation]: #motivation

The current block structure follows the Fabric envelope model:

```proto
message Block {
  BlockHeader header = 1;
  BlockData data = 2;
  BlockMetadata metadata = 3;
}

message BlockData {
  repeated bytes data = 1; // serialized Envelope
}

message Envelope {
  bytes payload = 1;   // serialized Payload
  bytes signature = 2;
}

message Payload {
  Header header = 1;
  bytes data = 2;      // serialized transaction-specific message
}

message Header {
  bytes channel_header = 1;    // serialized ChannelHeader
  bytes signature_header = 2;  // serialized SignatureHeader
}
```

This model adds several nested byte boundaries: block data contains serialized envelopes, each envelope contains a serialized payload, each payload contains transaction-specific serialized data, and the header contains serialized subheaders.

This has three concrete drawbacks for Fabric-X:

1. The structure is overcomplicated. Developers must reason about block data, envelopes, payloads, headers, and transaction-specific bytes separately even when the data is already known structurally.
2. Signature handling is inconsistent. Endorsers and Fabric Smart Client submitter code currently sign protobuf payload bytes for orderer submission, while fabric-x-committer endorsement verification uses `TxNamespace.ASN1Marshal(txID)` and fabric-x-orderer BFT signatures use ASN.1-marshaled consensus messages.
3. The committer pays a performance cost from repeated nested unmarshaling, extra byte-slice allocations, and later garbage-collection overhead.

Fabric-X does not have a production network that requires preserving the current wire format. The proposal can therefore directly replace the current request/block representation instead of supporting both old and new representations in parallel.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Fabric-X blocks now carry typed transaction envelopes. Instead of treating each transaction in a block as opaque bytes, the block exposes a typed `TransactionEnvelope` value. Each envelope has a typed header, one submitter signature, and exactly one transaction body variant.

The shared concepts are:

- `TypedBlock`: block structure with ordered typed transaction envelopes.
- `TransactionEnvelope`: header, signature, and typed transaction body.
- `TransactionHeader`: transaction type, transaction ID, submitter identity, and nonce.
- `TransactionVariant`: one of the concrete transaction message types, such as application or config transaction.
- `TransactionEnvelopeToSign`: ASN.1 signing input derived from the envelope, excluding the signature field.

A Fabric-X developer should think of transaction data as typed from the point where it enters the system. Generic byte slots are no longer the mechanism for distinguishing transaction kinds. If Fabric-X adds a new transaction type, the shared schema gets a new explicit variant for that type.

For example, an application transaction envelope conceptually looks like this:

```proto
message TransactionEnvelope {
  TransactionHeader header = 1;
  bytes signature = 2;
  oneof transaction {
    ApplicationTransaction application = 3;
    ConfigTransaction config = 4;
  }
}
```

A submitter constructs the typed envelope, derives canonical ASN.1 DER signing bytes from the header and selected transaction variant, signs those bytes, and stores the signature in `TransactionEnvelope.signature`. The signature field itself is not part of the signed data.

The orderer no longer verifies `request.Payload` protobuf bytes. It reconstructs the same ASN.1 DER bytes from typed fields and verifies the signature over those bytes.

The committer receives typed transaction envelopes in blocks. It reads the typed header and transaction variant directly, so the sidecar and verification pipeline do not need to unwrap nested `Envelope`, `Payload`, `ChannelHeader`, or `SignatureHeader` byte fields. Endorsement verification over application namespace data can continue using the existing `TxNamespace.ASN1Marshal(txID)` path unless a later design unifies that path with transaction-envelope signing.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## Shared typed block structure

The new block/envelope contract belongs in `fabric-x-common` because it is shared by endorsers, Fabric Smart Client submitter code, orderer, and committer.

Conceptual protobuf contract:

```proto
message Block {
  BlockHeader header = 1;
  repeated TransactionEnvelope transactions = 2;
  BlockMetadata metadata = 3;
}

message TransactionEnvelope {
  TransactionHeader header = 1;
  bytes signature = 2;
  oneof transaction {
    ApplicationTransaction application = 3;
    ConfigTransaction config = 4;
    // Future transaction types get explicit one-off fields.
  }
}

message TransactionHeader {
  uint64 type = 1;
  string tx_id = 2;
  bytes creator = 3;
  bytes nonce = 4;
}
```

The exact file/package placement in `fabric-x-common` should be chosen during implementation, but the schema must be generated and consumed by `fabric-x-orderer`, `fabric-x-committer`, endorsers, and Fabric Smart Client submitter code.

Validation rules:

- `TransactionEnvelope.header` is required.
- `TransactionEnvelope.signature` is required for submitted transactions.
- Exactly one transaction variant is set.
- `TransactionHeader.type` matches the selected transaction variant.
- New transaction types use explicit variants, not generic `bytes data`.

## ASN.1 transaction-envelope signing

The signing input follows the same pattern as `TxNamespace.ASN1Marshal(txID)` in `fabric-x-common/api/applicationpb/asn1.go`: translate protobuf data into stable ASN.1 stub structs, encode with DER, and sign those deterministic bytes.

Conceptual Go structure:

```go
type asn1TransactionEnvelopeToSign struct {
    Type    int
    TxID    string `asn1:"utf8"`
    Creator []byte
    Nonce   []byte

    Application *asn1ApplicationTransaction `asn1:"optional"`
    Config      *asn1ConfigTransaction      `asn1:"optional"`
}
```

Included fields:

- transaction type
- transaction ID
- submitter creator identity bytes
- nonce
- exactly one typed transaction variant

Excluded fields:

- signature bytes
- serialized envelope bytes
- serialized payload bytes
- serialized header bytes

Submitter behavior:

1. Construct typed `TransactionEnvelope` without signature.
2. Validate header and exactly one transaction variant.
3. Translate the envelope to `asn1TransactionEnvelopeToSign`.
4. DER-encode the ASN.1 structure.
5. Sign DER bytes.
6. Store signature in `TransactionEnvelope.signature`.

Orderer behavior:

1. Receive typed request/envelope data.
2. Validate header, signature, and transaction variant.
3. Reconstruct ASN.1 DER bytes from typed fields, excluding signature.
4. Verify submitter signature over DER bytes.
5. Verify the submitter identity against the ASN.1 DER signed data.

The existing BFT orderer consensus signing path over `MessageToSign.ASN1MarshalOrPanic()` remains out of scope for this proposal.

## Committer processing

The committer sidecar consumes typed transaction envelopes. It does not unwrap nested Fabric envelopes in the target path. After normal protobuf decoding of the block message, committer processing reads typed fields directly.

Target behavior:

- No explicit downstream `proto.Unmarshal` of envelope bytes.
- No explicit downstream `proto.Unmarshal` of payload bytes.
- No explicit downstream `proto.Unmarshal` of channel/signature header bytes.
- No redundant storage of serialized envelope, payload, or header byte copies in committer transaction flow.

The verifier receives typed transaction data. Current endorsement verification over namespace data can continue using `TxNamespace.ASN1Marshal(txID)`.

## Error cases

The new path rejects:

- missing header
- missing signature
- missing creator
- missing nonce
- missing transaction ID
- unset transaction variant
- multiple transaction variants
- transaction type mismatch
- non-canonical ASN.1 signed data

# Drawbacks
[drawbacks]: #drawbacks

This is a cross-repository contract change. It requires coordinated updates in `fabric-x-common`, `fabric-x-orderer`, `fabric-x-committer`, endorsers, and Fabric Smart Client submitter code.

The proposal also changes generated protobuf APIs, which can create broad compile-time churn. All dependent repos must update generated code consistently.

ASN.1 DER signing requires schema discipline. Any change to typed transaction fields must update the ASN.1 translation structs and determinism tests. If the ASN.1 structure and protobuf structure drift apart, signatures can fail or omit important fields.

The direct replacement changes the submitter contract. This is acceptable because Fabric-X does not have a production network that needs the current request format, but it still requires synchronized updates to tests, endorsers, and Fabric Smart Client submitter code.

# Rationale and alternatives
[alternatives]: #alternatives

This design is preferred because it removes the source of repeated byte conversions rather than hiding it behind helper functions. Typed envelopes make transaction shape explicit, and ASN.1 DER signing aligns orderer request signatures with existing Fabric-X signing patterns.

Alternatives considered:

- Keep the Fabric envelope byte model. This preserves the current protobuf payload signing shape, but keeps the overcomplicated structure, inconsistent signing model, and committer overhead.
- Deserialize once at committer ingress. The current committer already partially follows this approach by parsing envelopes at sidecar ingress, but the current structure still allocates nested byte slices and parsed objects that increase garbage-collection pressure. The typed block structure removes the nested byte representation from the source format rather than only improving where it is parsed.
- Use generic `bytes data` with a transaction type enum. This recreates the current problem by keeping transaction bodies opaque.
- Use separate transaction arrays per type. This makes mixed transaction ordering harder to represent and validate.
- Sign the whole envelope including the signature. This is impossible because the signature cannot include itself in signed data.
- Sign only the transaction body. This fails to bind header fields such as type, transaction ID, creator, and nonce to the signature.

If Fabric-X does not make this change, the platform keeps nested byte structures, repeated committer parsing, and inconsistent signing semantics between endorsers, Fabric Smart Client submitter code, orderer request filtering, committer endorsement verification, and BFT consensus signatures.

# Prior art
[prior-art]: #prior-art

The current Fabric envelope model is the main prior design. It is flexible and supports many transaction types through opaque bytes, but that flexibility is exactly what makes Fabric-X block handling more complex than necessary.

Fabric-X already has ASN.1 signing prior art:

- `TxNamespace.ASN1Marshal(txID)` in `fabric-x-common/api/applicationpb/asn1.go` produces deterministic ASN.1 data for committer endorsement verification.
- `MessageToSign.ASN1MarshalOrPanic()` in `fabric-x-common/protoutil/blockutils.go` is used for orderer BFT consensus signatures.

This RFC extends that pattern to transaction-envelope submission signatures so Fabric-X uses one consistent approach for canonical signed data.

# Testing
[testing]: #testing

Required tests:

- All current tests in `fabric-x-common`, `fabric-x-committer`, and `fabric-x-orderer` must pass.
- ASN.1 DER determinism tests for `TransactionEnvelopeToSign`.
- Orderer `SigFilter` tests verifying ASN.1 DER signed requests.
- Committer tests proving sidecar/verifier path consumes typed fields without nested envelope/payload/header unmarshaling.

Expected commands:

```bash
cd ../fabric-x-common
go test ./...

cd ../fabric-x-orderer
go test ./...

cd ../fabric-x-committer
make proto
make test
make test-no-db
make test-all-db
make test-integration
make test-container
```

Performance validation should compare before/after allocation and CPU behavior for committer transaction processing. Benchmarks should show fewer byte-slice allocations in the path that replaces nested envelope bytes with typed fields.

# Dependencies
[dependencies]: #dependencies

- `fabric-x-common` issue: https://github.com/hyperledger/fabric-x-common/issues/76
- `fabric-x-common`: owns typed block/envelope protobufs and ASN.1 signing helpers.
- `fabric-x-orderer`: verifies submitter signatures over ASN.1 DER typed data.
- `fabric-x-committer`: consumes typed transaction envelopes and removes redundant byte-oriented internal paths.
- Endorsers and Fabric Smart Client submitter code: construct typed envelopes and sign ASN.1 DER `TransactionEnvelopeToSign` bytes.
- Protobuf generation in all affected repos.

# Unresolved questions
[unresolved]: #unresolved-questions

- Which exact `fabric-x-common` proto package and file should own the new typed block/envelope messages?
- What exact benchmark threshold should be used to measure allocation and GC improvement?
- Which endorser and Fabric Smart Client submitter components should adopt `TransactionEnvelopeToSign` helpers during initial implementation?
