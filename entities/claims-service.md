---
type: Component
title: ClaimsService
description: The ClaimsService class (src/claims/claims.service.ts) orchestrates the First Notice of Loss (FNOL) business logic in claims-intake.
resource: https://github.com/Nox-Demo-Org/kb-tw-claims-intake-claims-intake/blob/main/entities/claims-service.md
tags:
- claims-intake
- entities
sources:
- resource: https://github.com/Nox-Demo-Org/claims-intake/blob/HEAD/src/claims/claims.service.ts
- resource: https://github.com/Nox-Demo-Org/claims-intake/blob/HEAD/src/claims/claim.validator.ts
- resource: https://github.com/Nox-Demo-Org/claims-intake/blob/HEAD/src/clients/policy.client.ts
generated:
  by: nox/gemini-3.7-flash
  at: '2026-10-05T12:43:59Z'
---

<!-- anchor: src/claims/claims.service.ts:L1-L25 -->
<!-- anchor: src/claims/claim.validator.ts:L1-L12 -->
<!-- anchor: src/clients/policy.client.ts:L1-L14 -->

# ClaimsService

The `ClaimsService` class (`src/claims/claims.service.ts`) orchestrates the First Notice of Loss (FNOL) business logic in `claims-intake`. Called by the [[entities/claims-controller]], it manages the intake process from initial payload validation and policy cover verification to event publishing and response generation.

## Responsibilities

- **Input Validation**: Executes channel-aware validation on incoming `NewClaim` payloads by calling `validate()` from [[concepts/validation-rules]].
- **Policy Verification**: Queries the `policy-admin` service via [[entities/policy-client]] to confirm that the policy exists and has a status of `"active"`.
- **Peril Cover & Excess Resolution**: Inspects the policy's `cover` array to verify coverage for the specified `peril` and extracts the applicable `excess_pence` (see [[decisions/adr-policy-cover-excess-lookup]]).
- **Event Construction & Publishing**: Generates a unique `claim_id` (UUID v4) and timestamp (`reported_at`), constructs a `ClaimReported` event payload, and publishes it to the `claims.claim.reported` Google Cloud Pub/Sub topic via [[entities/event-publisher]] (see [[decisions/adr-event-driven-intake]]).
- **Response Construction**: Returns the submission acknowledgment containing the generated `claim_id`, `status: "submitted"`, and the calculated `excess_amount`.

## Dependencies

- **[[entities/policy-client]] (`PolicyClient`)**: Injected via constructor (`private readonly policies: PolicyClient`). Used to fetch policy details from `policy-admin` via `GET /v1/policies/{id}`.
- **`validate` (`src/claims/claim.validator.ts`)**: Validation helper function used to enforce input rules before external service calls (see [[concepts/validation-rules]]).
- **[[entities/event-publisher]] (`publish`)**: Function used to emit the `claims.claim.reported` Pub/Sub message.
- **[[entities/claim]] (`NewClaim`, `ClaimReported`)**: Data transfer objects defining incoming requests and outgoing event structures.

## Intake Process (`report`)

The core intake workflow is encapsulated in `ClaimsService.prototype.report(input: NewClaim)`:

```typescript
async report(input: NewClaim) {
  validate(input);
  const policy = await this.policies.get(input.policy_id); // GET /v1/policies/{id}
  if (policy.status !== "active") throw new Error("policy is not active");
  const cover = policy.cover.find((c) => c.peril === input.peril);
  if (!cover) throw new Error("peril not covered");

  const event: ClaimReported = {
    claim_id: crypto.randomUUID(),
    ...input,
    excess_amount: cover.excess_pence,
    reported_at: new Date().toISOString(),
  };
  await publish("claims.claim.reported", event);
  return { claim_id: event.claim_id, status: "submitted", excess_amount: event.excess_amount };
}
```

### Execution Steps & Error Handling

1. **Validation**: Calls `validate(input)`. Throws an error if required fields like `policy_id` (or `peril` for non-phone channels) are missing.
2. **Policy Retrieval**: Calls `this.policies.get(input.policy_id)` to retrieve `PolicyView` details.
3. **Status Check**: Verifies `policy.status === "active"`. If not active, throws `Error("policy is not active")`.
4. **Peril Coverage Check**: Searches `policy.cover` for an entry matching `input.peril`. If no match is found, throws `Error("peril not covered")`.
5. **Event Publication**: Assembles the `ClaimReported` payload including `crypto.randomUUID()` and `cover.excess_pence`, then awaits `publish("claims.claim.reported", event)`.
6. **Return Value**: Resolves with:
   - `claim_id`: Generated UUID string.
   - `status`: `"submitted"`.
   - `excess_amount`: Integer excess amount in pence (`cover.excess_pence`).

For more details on the end-to-end flow, see [[concepts/claim-intake-flow]] and [[summaries/api-spec]].
