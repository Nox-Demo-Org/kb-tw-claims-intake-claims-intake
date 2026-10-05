---
type: Architecture Decision
title: 'ADR: Event-Driven Claim Submission via Pub/Sub'
description: Once a claim passes validation (see validation-rules) and policy coverage verification via policy-client, the claim must be propagated to downstream systems including claims-management (to manage claim lifecycle and persist excess) and f…
resource: https://github.com/Nox-Demo-Org/kb-tw-claims-intake-claims-intake/blob/main/decisions/adr-event-driven-intake.md
tags:
- claims-intake
- decisions
generated:
  by: nox/gemini-3.7-flash
  at: '2026-10-05T12:43:59Z'
---

# ADR: Event-Driven Claim Submission via Pub/Sub

## Status
Accepted

## Context
`claims-intake` serves as the First Notice of Loss (FNOL) service for Tidewell Mutual, accepting claim submissions from the `customer-portal` and the contact centre via `POST /v1/claims` (see [[entities/claims-controller]]). 

Once a claim passes validation (see [[concepts/validation-rules]]) and policy coverage verification via [[entities/policy-client]], the claim must be propagated to downstream systems including `claims-management` (to manage claim lifecycle and persist excess) and `fraud-scoring`. Direct synchronous REST calls from `claims-intake` to all downstream consumers would create tight coupling and reduce availability during peak intake periods.

## Decision
We use Google Cloud Pub/Sub (`@google-cloud/pubsub`) for asynchronous event-driven claim submission:

1. **Event Topic**: A single topic `claims.claim.reported` is used for broadcasting newly reported claims.
2. **Publisher Module**: Implemented in `src/events/publisher.ts` via `publish("claims.claim.reported", payload)` using `pubsub.topic(topic).publishMessage({ json: payload })` (see [[entities/event-publisher]]).
3. **Orchestration**: In [[entities/claims-service]], after verifying policy status and peril cover via `GET /v1/policies/{id}` (see [[decisions/adr-policy-cover-excess-lookup]]), the service constructs a `ClaimReported` payload containing:
   - `claim_id` (generated UUID via `crypto.randomUUID()`)
   - The original `NewClaim` input fields (`policy_id`, `peril`, etc.)
   - `excess_amount` (derived from `cover.excess_pence`)
   - `reported_at` (ISO timestamp string)
4. **Intake Response**: Once the event is published to Pub/Sub, the intake API synchronously returns `{ claim_id, status: "submitted", excess_amount }` to the client. See [[concepts/claim-intake-flow]].

## Consequences

### Positive
- **Decoupling**: `claims-intake` does not need direct knowledge of or network access to downstream consumers (`claims-management`, `fraud-scoring`).
- **Extensibility**: Additional consumers can subscribe to `claims.claim.reported` without code changes to `claims-intake`.
- **Consistent Excess Handoff**: The calculated excess shown to the customer is embedded in the `claims.claim.reported` payload as `excess_amount`, allowing `claims-management` to reliably store it.

### Trade-offs
- **Asynchronous Lifecycle**: The synchronous response returns `status: "submitted"`; full claim processing downstream is asynchronous.
- **Infrastructure Dependency**: Introduces a dependency on Google Cloud Pub/Sub infrastructure and SDK (`@google-cloud/pubsub`). For API and contract schemas, see [[summaries/api-spec]] and [[entities/claim]].
