# Phase 1 Data Model: Shared-Contracts Schema Source of Truth

**Feature**: `001-shared-contracts-schemas` | **Date**: 2026-06-07

This feature defines **contract shapes**, not persisted runtime entities. "Data model" here
means the structure of the published contract artifact and the canonical message/DTO/event
shapes it exposes. No database tables are created (the module owns no schema).

## Artifact-level entities

### Contract Artifact

The single published, versioned unit of `shared-contracts`.

| Field | Type | Rule |
|-------|------|------|
| version | semantic version (MAJOR.MINOR.PATCH) | One version covers all categories together (FR-007). Bump severity matches detected change (FR-009/FR-009a). |
| changelog | ordered list of Changelog Entry | Each release MUST add an entry (FR-011). |
| categories | { HTTP DTOs, gRPC protos, Kafka event schemas, RFC 7807 model } | All present in one artifact (FR-005). |

### Changelog Entry

| Field | Type | Rule |
|-------|------|------|
| version | semantic version | Matches the release it documents. |
| date | ISO date | — |
| breaking | boolean | MUST be true iff a backward-incompatible change is included (FR-009/FR-011). |
| changes | list of text | Human-readable summary of what changed. |

### Version Identifier (semantics)

- MAJOR bump ⇔ at least one backward-incompatible contract change (any category).
- MINOR/PATCH ⇔ backward-compatible changes only (FR-010).
- Interoperability guaranteed across N and N-1 (FR-010a).

## Contract shapes (canonical definitions)

> These are the **representations** the artifact exposes. Concrete sketches live in
> [contracts/](contracts/). Field lists below are the canonical, technology-agnostic view.

### Common — Problem Detail (RFC 7807)

Used as the HTTP error surface (`ProblemDetail` Java DTO) and mirrored as a proto message
for gRPC/event error surfaces.

| Field | Type | Notes |
|-------|------|-------|
| type | URI string | RFC 7807 `type` |
| title | string | short, human-readable |
| status | int | HTTP status code |
| detail | string | instance-specific explanation |
| instance | URI string | identifies the occurrence |
| (extensions) | map | optional problem-specific members |

### Product domain

**Product (message / DTO)** — canonical product representation.

| Field | Type | Rule |
|-------|------|------|
| id | string (UUID) | identity; required |
| sku | string | unique business key; required |
| name | string | required, non-blank |
| description | string | optional |
| priceAmount | decimal (minor units or string) | ≥ 0 |
| currency | string (ISO 4217) | required |
| active | boolean | default true |

**Stock Movement (event payload)** — append-only (Principle V).

| Field | Type | Rule |
|-------|------|------|
| movementId | string (UUID) | identity; required; immutable |
| productId | string (UUID) | required |
| quantityDelta | int (signed) | non-zero; +in / -out |
| reason | enum { PURCHASE, SALE, ADJUSTMENT, RETURN } | required |
| occurredAt | timestamp | required |

> Stock movements are modeled as immutable event payloads — no update/delete shape exists,
> reinforcing Principle V at the contract level.

### Order domain

**Order (message / DTO)**

| Field | Type | Rule |
|-------|------|------|
| id | string (UUID) | identity; required |
| customerId | string (UUID) | required |
| status | enum OrderStatus | see state machine below |
| lines | list of OrderLine | ≥ 1 when CONFIRMED+ |
| totalAmount | decimal | ≥ 0 |
| currency | string (ISO 4217) | required |

**OrderLine**

| Field | Type | Rule |
|-------|------|------|
| productId | string (UUID) | required |
| sku | string | required |
| quantity | int | ≥ 1 |
| unitPriceAmount | decimal | ≥ 0 |

**OrderStatus (enum) + state machine** — mirrors Principle V exactly:

```text
DRAFT → CONFIRMED → SHIPPED → DELIVERED
DRAFT → CANCELLED
DELIVERED → RETURN_REQUESTED → RETURNED
```

Enum values: `DRAFT, CONFIRMED, SHIPPED, DELIVERED, CANCELLED, RETURN_REQUESTED, RETURNED`.

**Order State Changed (event payload)**

| Field | Type | Rule |
|-------|------|------|
| eventId | string (UUID) | identity; required |
| orderId | string (UUID) | required |
| fromStatus | OrderStatus | required |
| toStatus | OrderStatus | required; transition MUST be one allowed by the state machine |
| occurredAt | timestamp | required |

## Relationships

- `Order` 1—N `OrderLine`; each `OrderLine` references a `Product` by `productId`/`sku`.
- `Stock Movement` references a `Product` by `productId`.
- `Order State Changed` references an `Order` by `orderId`.
- No cross-service foreign keys or joins are implied — these are message shapes exchanged
  over the sanctioned channels, consistent with Principle I (bounded data ownership).

## Validation rules (summary)

- All identity fields are UUID strings and required.
- Monetary amounts are non-negative and always paired with an ISO 4217 currency.
- Order status transitions are constrained to the Principle V state machine; an event whose
  `fromStatus → toStatus` is not in the allowed set is an invalid contract instance.
- Stock movement `quantityDelta` is signed and non-zero; movement payloads have no mutation
  shape (append-only).
