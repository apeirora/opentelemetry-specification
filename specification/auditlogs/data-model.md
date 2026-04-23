<!--- Hugo front matter used to generate the website version of this page:
linkTitle: Data Model
weight: 2
--->

# Audit Logs Data Model

**Status**: [Development](../document-status.md)

<details>
<summary>Table of Contents</summary>

<!-- toc -->

- [Design Notes](#design-notes)
  * [Requirements](#requirements)
  * [Tiered Processing Model](#tiered-processing-model)
  * [Integrity and Verification Model](#integrity-and-verification-model)
- [Audit Log Record Definition](#audit-log-record-definition)
  * [Field: `Timestamp`](#field-timestamp)
  * [Field: `ObservedTimestamp`](#field-observedtimestamp)
  * [Field: `RecordId`](#field-recordid)
  * [Field: `EventName`](#field-eventname)
  * [Field: `Actor`](#field-actor)
  * [Field: `ActorType`](#field-actortype)
  * [Field: `Action`](#field-action)
  * [Field: `Resource`](#field-resource)
  * [Field: `Outcome`](#field-outcome)
  * [Field: `SourceIp`](#field-sourceip)
  * [Field: `Body`](#field-body)
  * [Field: `Attributes`](#field-attributes)
  * [Field: `SchemaVersion`](#field-schemaversion)
  * [Integrity Fields](#integrity-fields)
    + [Field: `Hash`](#field-hash)
    + [Field: `Signature` / `Hmac`](#field-signature--hmac)
    + [Field: `HashAlgorithm`](#field-hashalgorithm)
    + [Field: `KeyId`](#field-keyid)
  * [Optional Ordering Fields](#optional-ordering-fields)
    + [Field: `SequenceNo`](#field-sequenceno)
    + [Field: `PrevHash`](#field-prevhash)
- [Tier-1 Response Model](#tier-1-response-model)
  * [Single-Record Response Fields](#single-record-response-fields)
  * [Batch Response Fields](#batch-response-fields)
  * [Recommended Tier-1 HTTP Status Codes](#recommended-tier-1-http-status-codes)
- [Tier-2 Verification and Delivery Model](#tier-2-verification-and-delivery-model)
  * [Tier-2 Domain Statuses](#tier-2-domain-statuses)
  * [Verify Processor Output Fields](#verify-processor-output-fields)
  * [Per-Sink Delivery Status Fields](#per-sink-delivery-status-fields)
  * [Recommended Tier-2 HTTP Status Codes](#recommended-tier-2-http-status-codes)
- [Tier-2 Record Lifecycle](#tier-2-record-lifecycle)
- [References](#references)

<!-- tocstop -->

</details>

This document defines a data model for audit logs that can be used across
application producers (Tier-1) and collector-side verification and distribution
pipelines (Tier-2).

The model extends the OpenTelemetry logs signal with audit-specific identity,
integrity, and delivery semantics so that both security and operational behavior
are represented in a consistent way.

## Design Notes

### Requirements

The audit logs data model is designed to satisfy the following requirements:

- It should capture core audit semantics consistently: who did what, to which
  resource, when, and with what result.
- It should preserve end-to-end integrity metadata (hash and signature/HMAC)
  and allow deterministic verification.
- It should support both single-record and batch processing.
- It should support synchronous and asynchronous delivery models without losing
  auditability.
- It should support clear failure and retry outcomes suitable for policy and
  forensics.

### Tiered Processing Model

This model assumes two logical tiers:

- **Tier-1** receives audit events from producers, applies initial processing,
  and forwards to Tier-2.
- **Tier-2** performs verification, routing, and exporter delivery tracking to
  required sinks.

Both tiers may expose response metadata. Tier-1 responses focus on immediate
acceptance or queueing behavior. Tier-2 responses focus on verification and
delivery state.

### Integrity and Verification Model

Audit events should include integrity fields that allow content authenticity
checks:

- `Hash` of canonical payload.
- `Signature` or `Hmac`.
- `HashAlgorithm`, `SchemaVersion`, and optional `KeyId`.

For stronger sequence tamper evidence, implementations may include
`SequenceNo` and `PrevHash` to enable hash chain validation.

## Audit Log Record Definition

Here is the list of fields in an audit log record:

| Field Name | Description |
| ---------- | ----------- |
| Timestamp | Time when the audited action occurred. |
| ObservedTimestamp | Time when Tier-1 observed the record. |
| RecordId | Unique identifier for the record. |
| EventName | Name identifying audit event type. |
| Actor | Principal that performed the action. |
| ActorType | Classification of actor (for example user, service, system). |
| Action | Action performed by the actor. |
| Resource | Target resource of the action. |
| Outcome | Result of the action (for example success, failure, denied). |
| SourceIp | Source network address for the action if available. |
| Body | Event payload. |
| Attributes | Additional structured metadata. |
| SchemaVersion | Version of the audit payload schema. |
| Hash | Integrity hash of canonical payload. |
| Signature / Hmac | Cryptographic authenticity proof. |
| HashAlgorithm | Algorithm used to compute hash. |
| KeyId | Identifier of signing key, if applicable. |
| SequenceNo | Optional sequence number for chain continuity. |
| PrevHash | Optional previous record hash for hash chain continuity. |

Below is the detailed description of each field.

### Field: `Timestamp`

Type: Timestamp, uint64 nanoseconds since UNIX epoch.

Description: Time when the audited event occurred at the source.

### Field: `ObservedTimestamp`

Type: Timestamp, uint64 nanoseconds since UNIX epoch.

Description: Time when the record was first observed by Tier-1.

### Field: `RecordId`

Type: string.

Description: Unique immutable record identifier. It is required for retries,
status reporting, deduplication, and Tier-1/Tier-2 correlation.

### Field: `EventName`

Type: string.

Description: Name identifying the event type. This should remain stable for a
given event schema.

### Field: `Actor`

Type: string or structured object encoded in [AnyValue](../common/README.md#anyvalue).

Description: Identity of the principal that executed the action.

### Field: `ActorType`

Type: string.

Description: Actor class such as `user`, `service`, or `system`.

### Field: `Action`

Type: string.

Description: Canonical verb describing the operation performed.

### Field: `Resource`

Type: [AnyValue](../common/README.md#anyvalue) or [Attribute Collection](../common/README.md#attribute-collections).

Description: Target object of the audit event, such as account, dataset,
endpoint, or configuration object.

### Field: `Outcome`

Type: string.

Description: Event result, for example `success`, `failure`, or `denied`.

### Field: `SourceIp`

Type: string.

Description: Source IP address or equivalent network origin identifier when
available.

### Field: `Body`

Type: [AnyValue](../common/README.md#anyvalue).

Description: Main payload for the audit record, potentially including structured
details specific to the event type.

### Field: `Attributes`

Type: [Attribute Collection](../common/README.md#attribute-collections).

Description: Additional context fields not represented as top-level fields.

### Field: `SchemaVersion`

Type: string.

Description: Schema version of canonical payload used for validation and
verification.

### Integrity Fields

#### Field: `Hash`

Type: string.

Description: Digest of canonical event payload for integrity validation.

#### Field: `Signature` / `Hmac`

Type: string or byte sequence.

Description: Source-authenticated cryptographic proof. Implementations may use
signature or HMAC depending on trust model and key management strategy.

#### Field: `HashAlgorithm`

Type: string.

Description: Hash algorithm identifier used for `Hash` (for example `sha256`).

#### Field: `KeyId`

Type: string.

Description: Optional key identifier used to verify `Signature` or `Hmac`.

### Optional Ordering Fields

#### Field: `SequenceNo`

Type: integer.

Description: Optional sequence number used for ordered processing and chain
continuity.

#### Field: `PrevHash`

Type: string.

Description: Optional hash pointer to previous record in a chain.

## Tier-1 Response Model

Tier-1 responses communicate immediate intake outcome and initial delivery or
queueing status.

### Single-Record Response Fields

- `record_id`
- `status_code`
- `status` (for example `delivered`, `queued`, `rejected`)
- `hash`
- `sink_timestamp` when synchronously delivered
- `reason` when non-success
- `retry_after` when queueing or backpressure applies

### Batch Response Fields

- `batch_id`
- `status_code`
- `hash` (batch hash or root when used)
- `sink_timestamp`
- `log_status_map`

### Recommended Tier-1 HTTP Status Codes

- `200` sent now.
- `202` stored for later processing.
- `400` invalid payload or schema.
- `401` or `403` authentication or authorization failure.
- `413` payload too large.
- `429` too many requests when quota policies are enabled.
- `500` unexpected internal failure.
- `503` cannot accept due to capacity constraints.

## Tier-2 Verification and Delivery Model

Tier-2 response semantics should represent verification and per-sink delivery
status, not only transport-level outcomes.

### Tier-2 Domain Statuses

- `accepted_pending_verify`
- `verified_queued`
- `verified_exporting`
- `delivered_all_sinks`
- `delivered_partial`
- `rejected_verify_failed`
- `rejected_schema_invalid`
- `rejected_authz`
- `failed_internal`

### Verify Processor Output Fields

For each record:

- `verify_status` (`passed`, `failed`, `deferred`)
- `verify_reason`
- `verify_details`
- `verified_at`
- `verification_profile`

### Per-Sink Delivery Status Fields

For each required sink:

- `exporter_name`
- `sink_type`
- `delivery_status` (`success`, `failed`, `timeout`, `retrying`, `unknown`)
- `attempt_count`
- `last_attempt_at`
- `sink_ack_id`
- `error_code`
- `error_message`

Record outcome is successful only when every required sink reports `success`.

### Recommended Tier-2 HTTP Status Codes

- `200` fully processed and all required sinks confirmed.
- `202` accepted for async verify or export.
- `207` optional mixed sink results; treat as non-success.
- `400` malformed payload or canonicalization failure.
- `401` or `403` authentication or authorization failure.
- `409` conflicting duplicate `record_id`.
- `413` payload too large.
- `429` intake throttled by policy.
- `500` unexpected Tier-2 failure.
- `503` temporary unavailability or no reliable intake capacity.

## Tier-2 Record Lifecycle

Recommended state transitions:

`received -> accepted_pending_verify -> verified_queued -> verified_exporting -> delivered_all_sinks`

Alternative terminal states:

- `rejected_schema_invalid`
- `rejected_verify_failed`
- `rejected_authz`
- `delivered_partial`
- `failed_internal`

## References

- Tier-1 Audit Log Processing and Security Proposals.
- Tier-2 AuditLog Endpoint Processing and Verification Proposals.
