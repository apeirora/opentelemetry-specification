<!--- Hugo front matter used to generate the website version of this page:
linkTitle: Tier-2 API
weight: 4
--->

# Audit Logs Tier-2 API

**Status**: [Development](../document-status.md)

<details>
<summary>Table of Contents</summary>

<!-- toc -->

- [Overview](#overview)
- [Endpoint Scope](#endpoint-scope)
- [Endpoint Definition](#endpoint-definition)
  * [Single-Record Ingest](#single-record-ingest)
  * [Batch Ingest](#batch-ingest)
- [Request Model](#request-model)
  * [Required Record Fields](#required-record-fields)
  * [Optional Record Fields](#optional-record-fields)
- [Response Model](#response-model)
  * [Single-Record Response Fields](#single-record-response-fields)
  * [Batch Response Fields](#batch-response-fields)
  * [Tier-2 Domain Status Values](#tier-2-domain-status-values)
- [HTTP Status Codes](#http-status-codes)
- [Idempotency and Duplicate Handling](#idempotency-and-duplicate-handling)
- [Verification and Sink Delivery Semantics](#verification-and-sink-delivery-semantics)
- [Retry Guidance](#retry-guidance)
- [Security Requirements](#security-requirements)
- [References](#references)

<!-- tocstop -->

</details>

## Overview

This document specifies the Tier-2 collector ingress endpoint for audit logs.
It defines API behavior for receiving `auditLog` records from Tier-1 and
returning transport and domain-level processing outcomes.

## Endpoint Scope

This API is Tier-2 only. It covers collector ingress behavior and response
semantics for verification and sink delivery tracking.

Tier-1 emit behavior is defined in:

- [Audit Logs Tier-1 API](./tier1-api.md)
- [Audit Logs SDK](./sdk.md)

## Endpoint Definition

The Tier-2 endpoint MUST support ingesting audit records in single-record and
batch modes.

### Single-Record Ingest

- **Path**: implementation defined (`/v1/auditlog` is recommended).
- **Method**: `POST`.
- **Body**: one audit record payload.

### Batch Ingest

- **Path**: implementation defined (`/v1/auditlogs` is recommended).
- **Method**: `POST`.
- **Body**: batch payload containing multiple audit records.

## Request Model

Request payloads MUST follow the [Audit Logs Data Model](./data-model.md).

### Required Record Fields

Each record MUST include:

- `timestamp`
- `record_id`
- `event_name`
- `actor`
- `actor_type`
- `action`
- `resource`
- `outcome`
- `body`
- `attributes`
- `schema_version`
- `hash`
- `signature` or `hmac`

### Optional Record Fields

Each record MAY include:

- `observed_timestamp`
- `source_ip`
- `hash_algorithm`
- `key_id`
- `sequence_no`
- `prev_hash`

## Response Model

Tier-2 responses MUST include:

- transport-level HTTP status code.
- domain-level Tier-2 processing status.

### Single-Record Response Fields

- `record_id`
- `http_status_code`
- `tier2_status`
- `verify_status`
- `hash`
- `received_at`
- `processed_at`
- `required_sinks`
- `sink_status_map`
- `reason`
- `retry_after` (when relevant)

### Batch Response Fields

- `batch_id`
- `http_status_code`
- `tier2_status`
- `batch_summary`
- `record_status_map`
- `sink_status_aggregate`

### Tier-2 Domain Status Values

Tier-2 SHOULD use the following domain status values:

- `accepted_pending_verify`
- `verified_queued`
- `verified_exporting`
- `delivered_all_sinks`
- `delivered_partial`
- `rejected_verify_failed`
- `rejected_schema_invalid`
- `rejected_authz`
- `failed_internal`

## HTTP Status Codes

Tier-2 SHOULD use the following HTTP status codes:

- `200` fully processed synchronously with all required sinks confirmed.
- `202` accepted for asynchronous verify/export, final state pending.
- `207` optional for mixed sink outcomes (`delivered_partial`), non-success.
- `400` malformed payload, schema mismatch, or canonicalization error.
- `401` or `403` authentication or authorization failure.
- `409` duplicate `record_id` with incompatible payload hash.
- `413` payload too large.
- `429` intake throttled by policy.
- `500` unexpected internal error.
- `503` temporarily unavailable or no reliable intake capacity.

## Idempotency and Duplicate Handling

Tier-2 MUST treat `record_id` as the primary idempotency key.

When a duplicate `record_id` is received:

- if canonical payload hash is identical, endpoint SHOULD return a deterministic
  idempotent response for existing record state.
- if canonical payload hash differs, endpoint MUST return conflict (`409`).

## Verification and Sink Delivery Semantics

Tier-2 MUST separate acceptance from final delivery state.

- `202` means accepted for processing, not fully delivered.
- `delivered_all_sinks` means every required sink has reported `success`.
- `delivered_partial` means at least one required sink is non-success and MUST
  be treated as non-success final outcome.

For per-sink status, Tier-2 SHOULD provide:

- `exporter_name`
- `sink_type`
- `delivery_status`
- `attempt_count`
- `last_attempt_at`
- `sink_ack_id`
- `error_code`
- `error_message`

## Retry Guidance

Clients SHOULD use `retry_after` when present.

Tier-1 SDK/exporter implementations SHOULD treat retryability as follows:

- retryable by default: `202`, `429`, `503`.
- generally non-retryable: `400`, `401`, `403`, `409`, `413`.

`500` retry behavior is implementation and policy dependent.

## Security Requirements

Tier-2 ingress MUST require secure transport and SHOULD support:

- TLS with server authentication.
- mTLS for stronger client identity where required.
- TLS with token-based authentication where mTLS is not used.

Tier-2 SHOULD validate integrity fields (`hash`, `signature` or `hmac`) per
configured verification policy.

## References

- [Audit Logs Data Model](./data-model.md)
- [Audit Logs Tier-1 API](./tier1-api.md)
- [Audit Logs SDK](./sdk.md)
