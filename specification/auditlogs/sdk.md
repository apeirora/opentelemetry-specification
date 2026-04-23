<!--- Hugo front matter used to generate the website version of this page:
linkTitle: SDK
weight: 3
--->

# Audit Logs SDK

**Status**: [Development](../document-status.md)

<details>
<summary>Table of Contents</summary>

<!-- toc -->

- [AuditLoggerProvider](#auditloggerprovider)
  * [AuditLoggerProvider Creation](#auditloggerprovider-creation)
  * [AuditLogger Creation](#auditlogger-creation)
  * [Configuration](#configuration)
  * [Shutdown](#shutdown)
  * [ForceFlush](#forceflush)
- [AuditLogger](#auditlogger)
  * [Emit an AuditRecord](#emit-an-auditrecord)
  * [Enabled](#enabled)
- [Additional AuditRecord interfaces](#additional-auditrecord-interfaces)
  * [ReadableAuditRecord](#readableauditrecord)
  * [ReadWriteAuditRecord](#readwriteauditrecord)
- [AuditRecord Limits](#auditrecord-limits)
- [AuditRecordProcessor](#auditrecordprocessor)
  * [AuditRecordProcessor operations](#auditrecordprocessor-operations)
    + [OnEmit](#onemit)
    + [Enabled](#enabled-1)
    + [Shutdown](#shutdown-1)
    + [ForceFlush](#forceflush-1)
  * [Built-in processors](#built-in-processors)
    + [Single processor (sync)](#single-processor-sync)
    + [Single processor (async)](#single-processor-async)
    + [Batching processor (async)](#batching-processor-async)
- [AuditRecordExporter](#auditrecordexporter)
  * [AuditRecordExporter operations](#auditrecordexporter-operations)
    + [Export](#export)
    + [ForceFlush](#forceflush-2)
    + [Shutdown](#shutdown-2)
- [Tier-1 to Tier-2 Response Semantics](#tier-1-to-tier-2-response-semantics)
- [Concurrency requirements](#concurrency-requirements)
- [References](#references)

<!-- tocstop -->

</details>

This document defines SDK behavior for Tier-1 audit logging where the
application emits audit records and the SDK processes and exports them to Tier-2
for verification and sink delivery.

The SDK defined here follows the OpenTelemetry Logs SDK model while adding
audit-specific processing and status semantics.

## AuditLoggerProvider

An `AuditLoggerProvider` MUST provide a way to set a
[Resource](../resource/sdk.md). If a `Resource` is specified, it SHOULD be
associated with all emitted audit records.

### AuditLoggerProvider Creation

The SDK SHOULD allow creation of multiple independent `AuditLoggerProvider`
instances.

### AuditLogger Creation

It SHOULD only be possible to create `AuditLogger` instances through an
`AuditLoggerProvider`.

The user input MUST be used to create an
[`InstrumentationScope`](../common/instrumentation-scope.md) associated with the
created `AuditLogger`.

If an invalid logger name is supplied, the SDK MUST return a working fallback
logger and SHOULD log an internal diagnostic message.

### Configuration

`AuditRecordProcessor` and `AuditRecordExporter` configuration MUST be owned by
the `AuditLoggerProvider`.

If configuration is updated, the update MUST apply to all previously created
`AuditLogger` instances and new `AuditLogger` instances.

### Shutdown

`Shutdown` MUST be called only once per `AuditLoggerProvider`.

After `Shutdown`, attempts to get an `AuditLogger` SHOULD return a valid no-op
logger if possible.

`Shutdown` MUST invoke `Shutdown` on all registered
[`AuditRecordProcessor`](#auditrecordprocessor) instances.

### ForceFlush

`ForceFlush` notifies all registered processors to immediately process and
export records already accepted by the SDK.

`ForceFlush` MUST invoke `ForceFlush` on all registered processors.

If immediate send to Tier-2 is not possible because of temporary transport or
capacity failures, the SDK SHOULD persist records in configured storage for
later processing and retry.

## AuditLogger

### Emit an AuditRecord

When emitting an audit record, the SDK SHOULD ensure
[`ObservedTimestamp`](./data-model.md#field-observedtimestamp) is set to current
time if it is missing.

The SDK MUST preserve required Tier-1 fields from the
[Audit Logs Data Model](./data-model.md#audit-log-record-definition):
`Timestamp`, `ObservedTimestamp`, `EventName`, `Actor`, `ActorType`, `Outcome`,
`Resource`, `Action`, `SourceIp`, `Body`, `Attributes`, `RecordId`, `Hash`,
`Signature` or `Hmac`, and `SchemaVersion`.

The SDK SHOULD preserve implementation-dependent fields when provided:
`TenantId` or `ServiceName`, `SequenceNo`, `PrevHash`, `HashAlgorithm`, and
`KeyId`.

If a required field is missing, the record MUST be rejected by SDK processing
or marked with failure status according to processor policy.

### Enabled

`Enabled` SHOULD return `false` when:

- no processors are registered.
- provider is shut down.
- all processors that implement `Enabled` return `false`.

Otherwise it SHOULD return `true`.

## Additional AuditRecord interfaces

### ReadableAuditRecord

A function receiving a `ReadableAuditRecord` MUST be able to read:

- all fields from the [Audit Logs Data Model](./data-model.md).
- associated `Resource`.
- associated `InstrumentationScope`.

### ReadWriteAuditRecord

`ReadWriteAuditRecord` is a superset of `ReadableAuditRecord`.

A function receiving a `ReadWriteAuditRecord` MUST be able to modify at least:

- `Timestamp`
- `ObservedTimestamp`
- `EventName`
- `Body`
- `Attributes`
- `Outcome`
- `Hash`
- `Signature` or `Hmac`
- `HashAlgorithm`
- `KeyId`

SDKs MAY provide cloning for asynchronous processing to avoid race conditions.

## AuditRecord Limits

Audit record attributes MUST follow
[common attribute limit rules](../common/README.md#attribute-limits).

If limits are implemented, SDKs SHOULD provide configurable parameters for
attribute count and value length limits.

## AuditRecordProcessor

`AuditRecordProcessor` defines hooks for record processing before export.

Processors can perform validation, canonicalization, integrity preparation, and
routing decisions.

Processors registered on `AuditLoggerProvider` MUST be invoked in registration
order.

### AuditRecordProcessor operations

#### OnEmit

`OnEmit` is called synchronously when an audit record is emitted.

`OnEmit` SHOULD NOT block for long operations and SHOULD NOT throw exceptions.

**Parameters:**

- `auditRecord` - a [ReadWriteAuditRecord](#readwriteauditrecord).
- `context` - resolved [Context](../context/README.md).

For processors registered directly on the provider, mutations made by one
processor MUST be visible to next processors in order.

#### Enabled

`Enabled` is optional and supports pre-filter checks.

**Parameters:**

- resolved [Context](../context/README.md)
- [InstrumentationScope](../common/instrumentation-scope.md)
- event name

**Returns:** `Boolean`

If indeterminate, implementation SHOULD default to `true`.

#### Shutdown

`Shutdown` is called when SDK shuts down and SHOULD execute cleanup.

After `Shutdown`, calls to `OnEmit` SHOULD be ignored gracefully.

`Shutdown` SHOULD include effects of `ForceFlush`.

#### ForceFlush

`ForceFlush` is a hint to complete pending processing and export for records
accepted before the call.

If timeout is configured, processors MUST prioritize honoring timeout.

If processors cannot complete export because of temporary transport or capacity
failures, they SHOULD persist pending records to configured storage for later
processing and retry.

### Built-in processors

The standard audit SDK SHOULD implement the three processors below.

#### Single processor (sync)

Processes one record and exports immediately in the caller path.

This processor provides strongest immediate delivery signal but increases caller
latency.

**Configurable parameters:**

- `exporter`
- `exportTimeoutMillis`

#### Single processor (async)

Processes one record at a time asynchronously using queue or storage.

This processor reduces request-path latency and supports retries.

**Configurable parameters:**

- `exporter`
- `maxQueueSize`
- `exportTimeoutMillis`
- `maxRetryCount`
- `initialBackoffMillis`

#### Batching processor (async)

Builds batches for asynchronous export to improve throughput and efficiency.

**Configurable parameters:**

- `exporter`
- `maxQueueSize`
- `scheduledDelayMillis`
- `exportTimeoutMillis`
- `maxExportBatchSize`
- `maxRetryCount`
- `initialBackoffMillis`

## AuditRecordExporter

`AuditRecordExporter` defines exporter behavior from Tier-1 SDK to Tier-2
ingress endpoint.

Exporters SHOULD support protocol-specific details such as OTLP transport,
authn/authz headers, and TLS/mTLS configuration.

### AuditRecordExporter operations

An `AuditRecordExporter` MUST support the following operations.

#### Export

`Export` sends a batch of `ReadableAuditRecord` to Tier-2.

`Export` MUST NOT be called concurrently for the same exporter instance.

`Export` MUST NOT block indefinitely and MUST complete with bounded timeout.

**Returns:** `ExportResult`

`ExportResult` MUST represent success or failure for transport-level submission.
If protocol allows, exporters SHOULD propagate detailed response payloads from
Tier-2.

#### ForceFlush

`ForceFlush` is a hint to complete export of records already received by the
exporter.

#### Shutdown

`Shutdown` is called once per exporter instance. After shutdown, `Export` calls
SHOULD return failure.

## Tier-1 to Tier-2 Response Semantics

Tier-1 SDK processors and exporters SHOULD interpret response status using both:

- transport-level HTTP status code.
- domain-level status fields in response body when available.

Tier-1 behavior SHOULD treat success conservatively:

- `200` may be treated as immediate delivered success.
- `202` means accepted or queued, not fully delivered.
- `207` or `delivered_partial` MUST be treated as non-success final outcome.
- `429` and `503` SHOULD trigger retry behavior when policy allows.
- `400`, `401`, `403`, `409`, and `413` SHOULD be treated as non-retryable
  unless implementation policy explicitly says otherwise.

For required field expectations and status fields, see
[Audit Logs Data Model](./data-model.md#tier-1-response-model) and
[Tier-2 verification model](./data-model.md#tier-2-verification-and-delivery-model).

## Concurrency requirements

For languages with concurrent execution support:

- **AuditLoggerProvider**: logger creation, `ForceFlush`, and `Shutdown` MUST be
  safe for concurrent use.
- **AuditLogger**: all methods MUST be safe for concurrent use.
- **AuditRecordExporter**: `ForceFlush` and `Shutdown` MUST be safe for
  concurrent use.

## References

- [Audit Logs Data Model](./data-model.md)
- [OpenTelemetry Logs SDK](../logs/sdk.md)
