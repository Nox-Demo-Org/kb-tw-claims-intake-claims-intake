# Architecture decisions

One ADR per architecture decision the code or documents make evident.

## Pages

- [ADR: Event-Driven Claim Submission via Pub/Sub](/decisions/adr-event-driven-intake.md) — Once a claim passes validation (see validation-rules) and policy coverage verification via policy-client, the claim must be propagated to downstream systems including claims-management (to manage claim lifecycle and persist excess) and f…
- [ADR: Policy Cover and Excess Lookup at Intake](/decisions/adr-policy-cover-excess-lookup.md) — Accepted
