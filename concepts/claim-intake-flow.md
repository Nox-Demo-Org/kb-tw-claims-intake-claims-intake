---
type: Concept
title: End-to-End Claim Intake Flow
description: The claim intake workflow represents the First Notice of Loss (FNOL) process in claims-intake.
resource: https://github.com/Nox-Demo-Org/kb-tw-claims-intake-claims-intake/blob/main/concepts/claim-intake-flow.md
tags:
- claims-intake
- concepts
generated:
  by: nox/gemini-3.7-flash
  at: '2026-10-05T12:43:59Z'
---

# End-to-End Claim Intake Flow

The claim intake workflow represents the First Notice of Loss (FNOL) process in `claims-intake`. It handles incoming claim submissions from the `customer-portal` and contact centre, verifies policy coverage and status against `policy-admin`, determines the applicable excess amount, publishes an event for downstream processing, and returns a submission confirmation.

The orchestrator for this entire process is [[entities/claims-service|ClaimsService.report]], exposed via [[entities/claims-controller|ClaimsController]].

---

## Workflow Diagram

```
Customer / Agent
       |
       |  POST /v1/claims (NewClaim payload)
       v
ClaimsController
       |
       |  report(input)
       v
ClaimsService
       |-- 1. validate(input) -------------------------> ClaimValidator
       |                                                 (throws if invalid)
       |
       |-- 2. GET /v1/policies/{policy_id} ------------> PolicyClient -> policy-admin
       |                                                 (checks status === "active")
       |                                                 (checks cover for peril)
       |
       |-- 3. publish("claims.claim.reported", event) -> Publisher -> Google Cloud Pub/Sub
       |                                                 (consumed by claims-management, fraud-scoring)
       |
       v
Return Response: { claim_id, status: "submitted", excess_amount }
```

---

## Detailed Step-by-Step Flow

### 1. Request Ingestion
An HTTP `POST /v1/claims` request is received by [[entities/claims-controller|ClaimsController.create]] with a [[entities/claim|NewClaim]] payload. The controller delegates execution directly to `ClaimsService.report(input)`.

### 2. Input Validation
`ClaimsService` invokes the `validate(input)` function from [[concepts/validation-rules|ClaimValidator]]:
- Verifies that `policy_id` is present.
- If `channel` is `"phone"`, peril validation is bypassed.
- If `channel` is not `"phone"`, verifies that `peril` is present.
- If any check fails, an error is thrown synchronously.

### 3. Policy Verification and Excess Lookup
`ClaimsService` queries the `policy-admin` service using [[entities/policy-client|PolicyClient.get(input.policy_id)]]:
1. **Policy Status Check**: Evaluates `policy.status`. If `policy.status !== "active"`, the service throws `new Error("policy is not active")`.
2. **Peril Coverage Check**: Searches `policy.cover` for a matching `peril` (`policy.cover.find(c => c.peril === input.peril)`). If no matching cover is found, the service throws `new Error("peril not covered")`.
3. **Excess Retrieval**: Extracts `cover.excess_pence` from the matched cover entry to populate `excess_amount`.

For architectural context on this synchronous verification step, see [[decisions/adr-policy-cover-excess-lookup|ADR: Policy Cover and Excess Lookup]].

### 4. Event Assembly and Publication
Upon successful policy validation, `ClaimsService` constructs a [[entities/claim|ClaimReported]] event object:
- `claim_id`: Generated dynamically using `crypto.randomUUID()`.
- `...input`: Copies all fields from the incoming `NewClaim` request.
- `excess_amount`: Set to `cover.excess_pence`.
- `reported_at`: Set to current timestamp via `new Date().toISOString()`.

The event is published to the `claims.claim.reported` Pub/Sub topic using [[entities/event-publisher|publish]]:
```typescript
await publish("claims.claim.reported", event);
```

Downstream consumers (such as `claims-management` and `fraud-scoring`) subscribe to this topic to initiate claim adjudication, fraud scoring, and record management. See [[decisions/adr-event-driven-intake|ADR: Event-Driven Intake]] for details on this pattern.

### 5. Client Response
`ClaimsService` returns the submission result to `ClaimsController`, which responds with HTTP status 200/201 and the payload:

```json
{
  "claim_id": "a5c6d37b-91d1-4b13-918c-3db5a88fbc8d",
  "status": "submitted",
  "excess_amount": 25000
}
```

---

## Error Handling Matrix

| Failure Point | Condition | Resulting Error / Behavior |
| :--- | :--- | :--- |
| **Validation** | `!c.policy_id` | `Error("policy_id is required")` |
| **Validation** | `!c.peril` (and `channel !== "phone"`) | `Error("peril is required")` |
| **Policy Retrieval** | Network failure or timeout (> 2000ms) | `AbortSignal.timeout` triggers fetch failure |
| **Policy Status** | `policy.status !== "active"` | `Error("policy is not active")` |
| **Cover Check** | Peril not found in `policy.cover` array | `Error("peril not covered")` |
| **Pub/Sub** | Failure publishing to `claims.claim.reported` | Unhandled rejection / HTTP 500 error |

---

## Related Documentation
- [[summaries/api-spec|API & Event Contracts]]: Detailed schemas for `POST /v1/claims`, `GET /v1/policies/{id}`, and `claims.claim.reported`.
- [[entities/claims-service|ClaimsService]]: Core implementation file.
- [[concepts/validation-rules|Validation Rules]]: Comprehensive breakdown of validation behavior and edge cases.
