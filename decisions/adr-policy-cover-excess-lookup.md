---
type: Architecture Decision
title: 'ADR: Policy Cover and Excess Lookup at Intake'
description: Accepted
resource: https://github.com/Nox-Demo-Org/kb-tw-claims-intake-claims-intake/blob/main/decisions/adr-policy-cover-excess-lookup.md
tags:
- claims-intake
- decisions
generated:
  by: nox/gemini-3.7-flash
  at: '2026-10-05T12:43:59Z'
---

# ADR: Policy Cover and Excess Lookup at Intake

## Status

Accepted

## Context

When a customer or call center agent reports a new claim via `POST /v1/claims` in [[entities/claims-controller]], the First Notice of Loss (FNOL) system must verify whether the policy is active and whether the specified peril is covered under the policy terms before accepting the submission. 

Additionally, the excess amount associated with the specific peril needs to be calculated and shown to the customer immediately during intake, as well as captured and forwarded downstream to `claims-management` for storage and tracking.

## Decision

We perform a synchronous REST lookup against `policy-admin` during the intake flow within [[entities/claims-service]] before accepting the claim or publishing events.

1. **Policy Administration Integration**:
   - [[entities/policy-client]] calls `GET /v1/policies/{id}` using the configured `POLICY_ADMIN_URL` (defaulting to `http://policy-admin/v1`) with a 2-second timeout (`AbortSignal.timeout(2000)`).
   - The response is typed as `PolicyView`:
     ```typescript
     export interface PolicyView {
       id: string;
       status: string;
       cover: { peril: string; limit_pence: number; excess_pence: number }[];
     }
     ```

2. **Policy Status and Peril Verification**:
   - The service verifies that `policy.status === "active"`. If not, it throws an error (`"policy is not active"`).
   - The service searches `policy.cover` for a matching `peril` (one of `"escape_of_water"`, `"storm"`, `"theft"`, `"fire"`, `"accidental_damage"`, or `"collision"`). If not found, it throws an error (`"peril not covered"`).

3. **Excess Amount Resolution**:
   - The peril's `excess_pence` is extracted and assigned to `excess_amount` in the [[entities/claim#claimreported]] event payload.
   - The `claims.claim.reported` event is published via [[entities/event-publisher]].
   - The HTTP response returned to the caller includes `{ claim_id, status: "submitted", excess_amount }`.

## Consequences

### Positive
- **Early Rejection**: Inactive policies and uncovered perils are rejected at the edge before any downstream claim records or workflows are initiated in `claims-management` or `fraud-scoring`.
- **Immediate Transparency**: The customer receives the exact excess amount (`excess_amount`) in pence synchronously in the submission response.
- **Consistent Event Payload**: The published `claims.claim.reported` event carries the resolved `excess_amount` calculated directly from the active policy contract at the time of reporting.

### Negative / Trade-offs
- **Synchronous Dependency**: The intake flow has a hard dependency on the availability and latency of `policy-admin`.
- **Latency & Timeout Constraints**: Requests to `policy-admin` are capped at a 2000ms timeout (`AbortSignal.timeout(2000)`), causing intake failures if `policy-admin` responds slowly or becomes unavailable.

## References

- Implementation: [[entities/policy-client]], [[entities/claims-service]]
- DTOs and Payloads: [[entities/claim]]
- Workflow Context: [[concepts/claim-intake-flow]]
- Related Decisions: [[decisions/adr-event-driven-intake]]
- Contract Specs: [[summaries/api-spec]]
