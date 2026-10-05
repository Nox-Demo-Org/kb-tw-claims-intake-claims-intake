---
type: Component
title: Claim Data Models and DTOs
description: The src/claims/claim.dto.ts file defines the core data transfer objects (DTOs) and event schemas for claim ingestion and event publishing within claims-intake.
resource: https://github.com/Nox-Demo-Org/kb-tw-claims-intake-claims-intake/blob/main/entities/claim.md
tags:
- claims-intake
- entities
sources:
- resource: https://github.com/Nox-Demo-Org/claims-intake/blob/HEAD/src/claims/claim.dto.ts
generated:
  by: nox/gemini-3.7-flash
  at: '2026-10-05T12:43:59Z'
---

<!-- anchor: src/claims/claim.dto.ts:L1-L20 -->

# Claim Data Models and DTOs

The `src/claims/claim.dto.ts` file defines the core data transfer objects (DTOs) and event schemas for claim ingestion and event publishing within `claims-intake`.

## Responsibilities

- Define the shape of incoming claim submission payloads (`NewClaim`) received via REST endpoints.
- Define the schema of the Google Cloud Pub/Sub event payload (`ClaimReported`) emitted when a claim is successfully processed.

## Data Structures

### `NewClaim`

Represents the request payload submitted by clients (e.g., `customer-portal` or call center agents) to initiate a claim.

```typescript
export interface NewClaim {
  policy_id: string;
  customer_id: string;
  peril: "escape_of_water" | "storm" | "theft" | "fire" | "accidental_damage" | "collision";
  incident_date: string;
  description: string;
  channel: "web" | "phone";
}
```

#### Fields
| Field | Type | Description |
|---|---|---|
| `policy_id` | `string` | Unique identifier of the policy associated with the claim. |
| `customer_id` | `string` | Unique identifier of the customer submitting the claim. |
| `peril` | `"escape_of_water"` \| `"storm"` \| `"theft"` \| `"fire"` \| `"accidental_damage"` \| `"collision"` | The peril/cause category under which the loss is being reported. |
| `incident_date` | `string` | Date/timestamp string representing when the incident occurred. |
| `description` | `string` | Free-text narrative describing the incident. |
| `channel` | `"web"` \| `"phone"` | Channel through which the FNOL intake was submitted. |

### `ClaimReported`

Represents the payload of the `claims.claim.reported` event published to Google Cloud Pub/Sub after intake verification and excess calculation.

```typescript
export interface ClaimReported {
  claim_id: string;
  policy_id: string;
  customer_id: string;
  peril: string;
  incident_date: string;
  description: string;
  excess_amount: number; // pence, shown to the customer at intake
  reported_at: string;
}
```

#### Fields
| Field | Type | Description |
|---|---|---|
| `claim_id` | `string` | Newly generated unique identifier for the registered claim. |
| `policy_id` | `string` | Policy identifier attached to the claim. |
| `customer_id` | `string` | Customer identifier who filed the claim. |
| `peril` | `string` | The peril name associated with the loss. |
| `incident_date` | `string` | Date string when the incident occurred. |
| `description` | `string` | Narrative description of the incident. |
| `excess_amount` | `number` | Policy excess amount in pence calculated and shown to the customer at intake. |
| `reported_at` | `string` | Timestamp string marking when the claim was accepted and reported. |

## Usage in System Flows

- **Input Ingestion**: `NewClaim` is received by [[entities/claims-controller]] during `POST /v1/claims` calls and validated against channel and format rules via [[concepts/validation-rules]].
- **Orchestration**: [[entities/claims-service]] consumes `NewClaim` data, performs policy checks using [[entities/policy-client]], resolves the excess amount, and constructs the `ClaimReported` payload.
- **Event Publication**: [[entities/event-publisher]] transmits `ClaimReported` on the `claims.claim.reported` topic for downstream systems as described in [[concepts/claim-intake-flow]] and [[decisions/adr-event-driven-intake]].

## Dependencies

- Defined in: `src/claims/claim.dto.ts`
- Consumed by:
  - [[entities/claims-controller]]
  - [[entities/claims-service]]
  - [[entities/event-publisher]]
  - [[summaries/api-spec]]
