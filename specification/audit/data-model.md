<!--- Hugo front matter used to generate the website version of this page:
linkTitle: Data Model
weight: 2
--->

# Audit Record Data Model

**Status**: [Development](../document-status.md)

<details>
<summary>Table of Contents</summary>

<!-- toc -->

- [Audit Record Data Model](#audit-record-data-model)
  - [Design Notes](#design-notes)
    - [Requirements](#requirements)
    - [Relationship to LogRecord](#relationship-to-logrecord)
    - [OTLP Envelope Layers](#otlp-envelope-layers)
  - [AuditRecord Definition](#auditrecord-definition)
    - [Field: `Timestamp`](#field-timestamp)
    - [Field: `ObservedTimestamp`](#field-observedtimestamp)
    - [Field: `EventName`](#field-eventname)
    - [Actor Fields](#actor-fields)
      - [Field: `Actor`](#field-actor)
      - [Field: `ActorType`](#field-actortype)
    - [Field: `Action`](#field-action)
    - [Field: `Outcome`](#field-outcome)
    - [Field: `TargetResource`](#field-targetresource)
    - [Field: `SourceIP`](#field-sourceip)
    - [Field: `Body`](#field-body)
    - [Field: `Attributes`](#field-attributes)
    - [Integrity Fields](#integrity-fields)
      - [Field: `Signature`](#field-signature)
      - [Field: `Algorithm`](#field-algorithm)
      - [Field: `Certificate`](#field-certificate)
  - [AuditReceipt Definition](#auditreceipt-definition)
    - [Field: `RecordId`](#field-recordid)
    - [Field: `IntegrityHash`](#field-integrityhash)
    - [Field: `SinkTimestamp`](#field-sinktimestamp)
  - [Example AuditRecords](#example-auditrecords)
    - [Successful user login](#successful-user-login)
    - [Failed privileged configuration change](#failed-privileged-configuration-change)
  - [References](#references)

<!-- tocstop -->

</details>

## Design Notes

### Requirements

The Audit Record Data Model was designed to satisfy the following
requirements:

- Every audit record MUST carry sufficient information to identify the
  actor, the action performed, the target of the action, and the
  outcome. This is the minimum required by ISO 27001 Annex A (event
  logging) and equivalent compliance frameworks.

- The data model MUST include timestamps that allow clock-skew detection
  across distributed services (ISO 27001 – clock synchronisation).

- The data model MUST support optional digital signatures so that
  records can be verified for integrity after delivery.

- The data model MUST be efficiently representable in OTLP by reusing
  the existing `LogRecord` proto message as a transport container.

- The data model MUST be extensible via arbitrary key-value attributes
  to accommodate application-specific compliance requirements.

### Relationship to LogRecord

`AuditRecord` is transported as an OTLP `LogRecord`. The mandatory
audit-specific fields are stored in dedicated `LogRecord` fields where
a natural mapping exists, and in the `Attributes` map otherwise. The
`Body` field carries the free-form `AuditRecord.Body` payload.

The `SeverityNumber` and `SeverityText` fields of the underlying
`LogRecord` MUST NOT be used for audit records. Severity is not a
meaningful concept for audit events; `Outcome` serves a similar purpose.

### OTLP Envelope Layers

The following OTLP envelope layers apply to audit records:

| OTLP layer             | Role in audit logging                                  |
|------------------------|--------------------------------------------------------|
| `Resource`             | Identifies the emitting service / host – reused as-is. |
| `LogRecord`            | Carries the `AuditRecord` payload (see mapping below). |
| `Attributes`           | Key-value context – reused as-is.                      |
| `InstrumentationScope` | **Not applicable.** MUST be left empty.                |

## AuditRecord Definition

An `AuditRecord` is a single security-relevant event emitted by
application or system code on behalf of an actor.

The following table provides a summary of all fields. Detailed
descriptions follow.

| Field               | Type                    | Req.   | Description                                    |
|---------------------|-------------------------|--------|------------------------------------------------|
| `Timestamp`         | `fixed64`               | MUST   | Event time, ns since UNIX epoch (UTC).         |
| `ObservedTimestamp` | `fixed64`               | MUST   | SDK observation time, ns since UNIX epoch.     |
| `EventName`         | `string`                | MUST   | Semantic name of the audit event.              |
| `Actor`             | `AnyValue`              | MUST   | Identity that performed the action.            |
| `ActorType`         | `enum`                  | MUST   | `USER`, `SERVICE`, or `SYSTEM`.                |
| `Action`            | `string`                | MUST   | Verb describing what was done.                 |
| `Outcome`           | `enum`                  | MUST   | `SUCCESS`, `FAILURE`, or `UNKNOWN`.            |
| `TargetResource`    | `AnyValue`              | SHOULD | The object acted upon.                         |
| `SourceIP`          | `string`                | MAY    | Source network address.                        |
| `Body`              | `AnyValue`              | MAY    | Free-form additional event details.            |
| `Attributes`        | `map<string, AnyValue>` | MAY    | Arbitrary key-value context.                   |
| `Signature`         | `bytes`                 | MAY    | Digital signature of the record.               |
| `Algorithm`         | `string`                | MAY    | Signature algorithm (e.g. `ES256`).            |
| `Certificate`       | `bytes`                 | MAY    | Public certificate for signature verification. |

### Field: `Timestamp`

| Property   | Value                            |
|------------|----------------------------------|
| Type       | `fixed64`                        |
| Required   | MUST be set                      |
| Constraint | `Timestamp <= ObservedTimestamp` |

The time at which the audit event occurred, expressed as nanoseconds
since the UNIX epoch (UTC). The application MUST set this field to the
time the auditable action actually took place.

If the application cannot determine the precise event time, it SHOULD
use the current time at the moment of the `emit` call and document this
in the `Body` or `Attributes`.

The clock used for `Timestamp` MUST be a wall-clock source
synchronised via NTP or an equivalent protocol. The SDK SHOULD warn at
startup if the system clock offset exceeds one second.

### Field: `ObservedTimestamp`

| Property   | Value                                              |
|------------|----------------------------------------------------|
| Type       | `fixed64`                                          |
| Required   | MUST be set                                        |
| Constraint | `ObservedTimestamp >= Timestamp` and `<= now(UTC)` |

The time at which the OpenTelemetry SDK observed the event, expressed
as nanoseconds since the UNIX epoch (UTC). The SDK MUST set this field
to the current time when `emit` is called if the application does not
provide it.

`ObservedTimestamp` enables the audit sink to detect clock skew between
the emitting service and the SDK host, and between the SDK host and the
sink. A significant difference between `Timestamp` and
`ObservedTimestamp` SHOULD be treated as a clock synchronisation
warning.

### Field: `EventName`

| Property | Value                          |
|----------|--------------------------------|
| Type     | `string`                       |
| Required | MUST be set; MUST NOT be empty |

A short, dot-separated semantic name that uniquely identifies the type
of audit event. `EventName` MUST be stable across releases and SHOULD
follow a hierarchical naming convention, for example:

- `user.login.success`
- `user.login.failure`
- `resource.file.read`
- `config.change`
- `privilege.escalation`

Semantic conventions for common `EventName` values SHOULD be defined
in the [OpenTelemetry Semantic Conventions](https://opentelemetry.io/docs/specs/semconv/).

### Actor Fields

Actor fields identify the entity that performed the auditable action.
Together, `Actor` and `ActorType` provide the minimum identity context
required by ISO 27001 Annex A item 1 (user IDs).

#### Field: `Actor`

| Property | Value       |
|----------|-------------|
| Type     | `AnyValue`  |
| Required | MUST be set |

The identity of the entity that performed the action. This MAY be:

- A user ID (string or integer)
- A service account name
- A fully qualified system identifier
- A structured object (e.g. `{ "id": "u123", "email": "a@example.com" }`)

If the actor identity cannot be determined (for example, an
unauthenticated request), `Actor` MUST still be set to a value
indicating the unknown state (e.g. `"anonymous"` or `"unknown"`).

#### Field: `ActorType`

| Property | Value                       |
|----------|-----------------------------|
| Type     | `enum`                      |
| Required | MUST be set                 |
| Values   | `USER`, `SERVICE`, `SYSTEM` |

Classifies the kind of entity that performed the action:

| Value     | Meaning                                                |
|-----------|--------------------------------------------------------|
| `USER`    | A human user, identified by a user account.            |
| `SERVICE` | An automated service, daemon, or service account.      |
| `SYSTEM`  | The operating system or a privileged system component. |

In times of AI agents and autonomous systems, the distinction between `USER`
and `SERVICE` may become blurred. The `ActorType` values are intended to be
broad categories rather than rigid classifications, and the SDK MUST NOT
enforce any particular semantics beyond what is described in the table above.
Applications SHOULD choose the most appropriate `ActorType` based on the
context of the action and the identity of the actor. Use `SERVICE` for
non-human actors that operate autonomously, and `USER` for human-initiated
actions, even if they are performed by an AI agent on behalf of a user.

### Field: `Action`

| Property | Value                          |
|----------|--------------------------------|
| Type     | `string`                       |
| Required | MUST be set; MUST NOT be empty |

A short verb or verb phrase that describes what the actor did.
Applications SHOULD use uppercase verbs to distinguish actions from
event names. Common values include:

`LOGIN`, `LOGOUT`, `READ`, `WRITE`, `CREATE`, `UPDATE`, `DELETE`,
`EXECUTE`, `APPROVE`, `REJECT`, `EXPORT`, `IMPORT`.

Applications MAY define domain-specific action verbs. Action names
SHOULD be stable across releases.

### Field: `Outcome`

| Property | Value                           |
|----------|---------------------------------|
| Type     | `enum`                          |
| Required | MUST be set                     |
| Values   | `SUCCESS`, `FAILURE`, `UNKNOWN` |

The result of the auditable action:

| Value     | Meaning                                                      |
|-----------|--------------------------------------------------------------|
| `SUCCESS` | The action completed successfully.                           |
| `FAILURE` | The action was attempted but did not complete successfully.  |
| `UNKNOWN` | The outcome could not be determined at the time of emission. |

ISO 27001 Annex A items 5 and 6 require logging both successful and
unsuccessful access attempts. Using `UNKNOWN` SHOULD be avoided except
for fire-and-forget actions where acknowledgement is not possible.

### Field: `TargetResource`

| Property | Value         |
|----------|---------------|
| Type     | `AnyValue`    |
| Required | SHOULD be set |

The object upon which the action was performed. This MAY be:

- A file path (string)
- A database table or row identifier
- A REST endpoint path
- A structured object with type and identifier fields

If the action was not directed at a specific resource (for example, a
system startup event), this field MAY be omitted.

Note: this field identifies the *target* of the auditable action and is
distinct from the OTLP `Resource` envelope, which identifies the
*emitting service or host*.

### Field: `SourceIP`

| Property | Value      |
|----------|------------|
| Type     | `string`   |
| Required | MAY be set |

The network address of the source of the auditable action. This field
SHOULD be set for network-initiated actions (for example, a login
attempt over HTTPS) and MAY be omitted for local or intra-process
actions.

The value SHOULD conform to the IPv4 dotted-decimal notation
(`203.0.113.42`) or the IPv6 full notation (`2001:db8::1`). Port
numbers MAY be appended using the `host:port` convention.

### Field: `Body`

| Property | Value      |
|----------|------------|
| Type     | `AnyValue` |
| Required | MAY be set |

Free-form additional information about the audit event. The value MAY
be a string, a byte buffer, or a structured object. Applications SHOULD
store information that does not fit naturally into the named fields here.

The `Body` MUST NOT be used to store fields that duplicate the named
mandatory fields (`Actor`, `Action`, `Outcome`, etc.).

### Field: `Attributes`

| Property | Value                   |
|----------|-------------------------|
| Type     | `map<string, AnyValue>` |
| Required | MAY be set              |

Arbitrary key-value pairs that provide additional context about the
audit event. Applications SHOULD use the OpenTelemetry
[attribute naming conventions](../common/attribute-naming.md).

Examples of useful attributes:

- `session.id` – the authenticated session identifier
- `request.id` – a correlation identifier for the originating request
- `tenant.id` – the tenant in a multi-tenant system
- `geolocation.country` – country of origin for network requests
- `tls.version` – the TLS protocol version used

Attribute keys MUST be unique within an `AuditRecord`. If the same key
is set multiple times, the last value MUST be used.

### Integrity Fields

The integrity fields enable a relying party to verify that an
`AuditRecord` was not altered after emission. They are OPTIONAL but
SHOULD be set in environments where tamper evidence is required by the
applicable compliance framework.

#### Field: `Signature`

| Property | Value      |
|----------|------------|
| Type     | `bytes`    |
| Required | MAY be set |

A digital signature over the canonical serialization of the
`AuditRecord`. The signature MUST cover all mandatory fields plus any
`Attributes` and `Body` that are present at emission time.

If `Signature` is set, `Algorithm` MUST also be set.

#### Field: `Algorithm`

| Property | Value                             |
|----------|-----------------------------------|
| Type     | `string`                          |
| Required | MUST be set if `Signature` is set |

The algorithm used to compute `Signature`. The value SHOULD be a
registered JWA algorithm identifier, for example `RS256`, `ES256`, or
`EdDSA`.

#### Field: `Certificate`

| Property | Value      |
|----------|------------|
| Type     | `bytes`    |
| Required | MAY be set |

The public-key certificate (DER-encoded X.509) corresponding to the
signing key, included so that relying parties can verify `Signature`
without an out-of-band key lookup.

Alternatively, Key ID, Subject Key Identifier (SKI), Issuer + SerialNumber or
other key reference MAY be included as an attribute instead of the full
certificate, but this is less self-contained and requires additional
configuration on the relying party side to resolve the key reference.

If `Certificate` is omitted, the relying party MUST obtain the public
key through a separately configured trust anchor.

## AuditReceipt Definition

An `AuditReceipt` is returned by `AuditLogger.emit` once the audit
sink has successfully persisted the record. It serves as proof-of-delivery
and enables integrity verification by the emitting application.

| Field           | Type      | Required    | Description                                                  |
|-----------------|-----------|-------------|--------------------------------------------------------------|
| `RecordId`      | `string`  | MUST be set | Unique identifier assigned by the sink.                      |
| `IntegrityHash` | `string`  | MUST be set | SHA-256 of the record as persisted by the sink.              |
| `SinkTimestamp` | `fixed64` | MUST be set | Nanoseconds since UNIX epoch when the sink wrote the record. |

### Field: `RecordId`

A unique, stable identifier for the persisted record, assigned by the
audit sink. The format is sink-specific (for example, a UUID, a
sequential integer, or a content-addressed hash).

The emitting application MAY store `RecordId` for later retrieval or
cross-referencing.

### Field: `IntegrityHash`

The SHA-256 hash of the canonical serialization of the `AuditRecord`
as it was written to persistent storage, computed by the sink. The hash
is returned to the emitting application so that it can verify that the
record was not altered between emission and persistence.

The `IntegrityHash` is computed by the **sink**, not by the SDK, so it
reflects the record as actually stored – not merely as transmitted.

The emitting application SHOULD compute the same hash locally
immediately after calling `emit` and compare it to the received
`IntegrityHash`. A mismatch indicates that the record was altered in
transit or by the sink and SHOULD trigger an alert (for example, log an
error or raise an incident).

### Field: `SinkTimestamp`

Nanoseconds since the UNIX epoch (UTC) at which the audit sink
persisted the record. This value MAY differ from `ObservedTimestamp`
due to buffering or network latency, and MAY be used to detect
abnormally long delivery times.

## Example AuditRecords

### Successful user login

```json
{
  "Timestamp": 1714041600000000000,
  "ObservedTimestamp": 1714041600001000000,
  "EventName": "user.login.success",
  "Actor": "u8472",
  "ActorType": "USER",
  "Action": "LOGIN",
  "Outcome": "SUCCESS",
  "TargetResource": "/api/auth/session",
  "SourceIP": "203.0.113.42",
  "Attributes": {
    "session.id": "sess-abc123",
    "tls.version": "TLSv1.3"
  }
}
```

### Failed privileged configuration change

```json
{
  "Timestamp": 1714041700000000000,
  "ObservedTimestamp": 1714041700002000000,
  "EventName": "config.change",
  "Actor": "svc-deployer",
  "ActorType": "SERVICE",
  "Action": "UPDATE",
  "Outcome": "FAILURE",
  "TargetResource": {
    "type": "kubernetes/configmap",
    "namespace": "production",
    "name": "app-secrets"
  },
  "Body": "Authorization denied: missing role 'config-writer'",
  "Attributes": {
    "request.id": "req-xyz789",
    "k8s.namespace": "production"
  }
}
```

## References

- [OTEP 0267 – Audit Logging Signal](../../oteps/0267-audit-logging.md)
- [OTEP 0097 – Log Data Model](../../oteps/logs/0097-log-data-model.md)
- [ISO 27001:2022 Annex A – Audit Logging controls](https://www.iso.org/standard/27001)
- [Common Event Format (CEF) specification](https://www.microfocus.com/documentation/arcsight/arcsight-smartconnectors-8.4/cef-implementation-standard/)
