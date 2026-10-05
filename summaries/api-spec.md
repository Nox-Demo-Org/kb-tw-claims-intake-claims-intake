---
type: Interface Reference
title: API and Event Specification
description: claims-intake serves as the First Notice of Loss (FNOL) entry point for Tidewell Mutual.
resource: https://github.com/Nox-Demo-Org/kb-tw-claims-intake-claims-intake/blob/main/summaries/api-spec.md
tags:
- claims-intake
- summaries
sources:
- resource: https://github.com/Nox-Demo-Org/claims-intake/blob/HEAD/src/claims/claims.controller.ts
- resource: https://github.com/Nox-Demo-Org/claims-intake/blob/HEAD/src/claims/claim.dto.ts
- resource: https://github.com/Nox-Demo-Org/claims-intake/blob/HEAD/src/events/publisher.ts
generated:
  by: nox/gemini-3.7-flash
  at: '2026-10-05T12:43:59Z'
---

<!-- anchor: src/claims/claims.controller.ts:L1-L14 -->
<!-- anchor: src/claims/claim.dto.ts:L1-L20 -->
<!-- anchor: src/events/publisher.ts:L1-L5 -->

# API and Event Specification

`claims-intake` serves as the First Notice of Loss (FNOL) entry point for Tidewell Mutual. It exposes HTTP REST endpoints for intake channels, integrates synchronously with upstream policy and customer services, and publishes asynchronous events to Google Cloud Pub/Sub upon claim creation.

---

## Contract Summary

| Contract | Protocol / Transport | Direction | Counterparties | Description |
| --- | --- | --- | --- | --- |
| `POST /v1/claims` | HTTP REST | Inbound (Exposed) | `customer-portal`, Contact Centre | Intake endpoint to report a new claim |
| `claims.claim.reported` | Google Cloud Pub/Sub | Outbound (Published) | `claims-management`, `fraud-scoring` | Notification emitted when a claim is reported |
| `GET /v1/policies/{id}` | HTTP REST | Outbound (Consumed) | `policy-admin` | Policy status and cover lookup |
| `GET /v1/customers/{id}` | HTTP REST | Outbound (Consumed) | `customer-identity` | Customer verification lookup |

---

## Inbound REST Endpoints

### Report a Claim (`POST /v1/claims`)

Handled by [[entities/claims-controller|ClaimsController]] and orchestrated by [[entities/claims-service|ClaimsService]] through the [[concepts/claim-intake-flow|Claim Intake Flow]].

- **Method**: `POST`
- **Path**: `/v1/claims`
- **Controller Method**: `ClaimsController.create(body: NewClaim)`
- **Validation**: Enforced according to channel rules described in [[concepts/validation-rules|Validation Rules]].

#### Request Payload (`NewClaim`)

Defined in [[entities/claim|claim.dto.ts]]:

| Field | Type | Required | Description / Constraints |
| --- | --- | --- | --- |
| `policy_id` | `string` | Yes | Policy identifier to verify against `policy-admin` |
| `customer_id` | `string` | Yes | Unique identifier for the customer reporting the claim |
| `peril` | `"escape_of_water" \| "storm" \| "theft" \| "fire" \| "accidental_damage" \| "collision"` | Yes | Cause of loss / peril type |
| `incident_date` | `string` | Yes | Date/timestamp when the incident occurred |
| `description` | `string` | Yes | Text description of the incident |
| `channel` | `"web" \| "phone"` | Yes | Intake channel (`web` for customer portal, `phone` for contact centre) |

#### Request Example

```json
{
  "policy_id": "pol_987654",
  "customer_id": "cust_123456",
  "peril": "escape_of_water",
  "incident_date": "2025-01-15T14:30:00Z",
  "description": "Burst pipe under kitchen sink causing water damage",
  "channel": "web"
}
```

#### Response Structure

Returns a JSON object upon successful creation:

| Field | Type | Description |
| --- | --- | --- |
| `claim_id` | `string` | Generated claim identifier |
| `status` | `string` | Literal value `"submitted"` |
| `excess_amount` | `number` | The excess amount shown to the customer (in pence), resolved from the policy |

#### Response Example

```json
{
  "claim_id": "clm_102938",
  "status": "submitted",
  "excess_amount": 25000
}
```

---

## Published Pub/Sub Events

### Claim Reported (`claims.claim.reported`)

Published using [[entities/event-publisher|Publisher]] via `@google-cloud/pubsub` whenever a valid claim is reported and excess is determined. See [[decisions/adr-event-driven-intake|ADR: Event-Driven Intake]].

- **Topic**: `claims.claim.reported`
- **Target Consumers**: `claims-management` (stores the excess and manages lifecycle), `fraud-scoring` (initiates risk scoring)
- **Payload Interface**: `ClaimReported` (see [[entities/claim|claim.dto.ts]])

#### Event Payload Schema (`ClaimReported`)

| Field | Type | Description |
| --- | --- | --- |
| `claim_id` | `string` | Unique identifier assigned to the reported claim |
| `policy_id` | `string` | Associated policy identifier |
| `customer_id` | `string` | Customer identifier reporting the claim |
| `peril` | `string` | The peril under which the claim was made |
| `incident_date` | `string` | ISO timestamp / string of the incident |
| `description` | `string` | Claim description submitted by the reporter |
| `excess_amount` | `number` | Excess amount in pence presented to the customer during intake |
| `reported_at` | `string` | Timestamp when the claim intake was completed |

#### Event Example

```json
{
  "claim_id": "clm_102938",
  "policy_id": "pol_987654",
  "customer_id": "cust_123456",
  "peril": "escape_of_water",
  "incident_date": "2025-01-15T14:30:00Z",
  "description": "Burst pipe under kitchen sink causing water damage",
  "excess_amount": 25000,
  "reported_at": "2025-01-15T15:00:00Z"
}
```

---

## Outbound Client Contracts

### Policy Admin Service (`GET /v1/policies/{id}`)

Consumed by [[entities/policy-client|PolicyClient]] to verify policy status and cover limits/excess. See [[decisions/adr-policy-cover-excess-lookup|ADR: Policy Cover and Excess Lookup]].

- **Base URL**: `process.env.POLICY_ADMIN_URL ?? "http://policy-admin/v1"`
- **Path**: `/policies/{id}`
- **Timeout**: `AbortSignal.timeout(2000)` (2 seconds)
- **Response Type**: `PolicyView`

```typescript
export interface PolicyView {
  id: string;
  status: string;
  cover: {
    peril: string;
    limit_pence: number;
    excess_pence: number;
  }[];
}
```

### Customer Identity Service (`GET /v1/customers/{id}`)

Referenced in system contracts for verifying customer identity details during intake.
