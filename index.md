---
okf_version: '0.2'
title: claims-intake
description: claims-intake is the First Notice of Loss (FNOL) service for Tidewell Mutual.
generated:
  at: '2026-10-05T12:43:59Z'
---

# claims-intake

`claims-intake` is the First Notice of Loss (FNOL) service for Tidewell Mutual. It provides the entry point for customers (via `customer-portal`) and call center agents to report insurance claims, validates claim details, verifies policy cover and status against `policy-admin`, calculates excess amounts, and publishes claim events to downstream systems such as `claims-management` and `fraud-scoring`.

### Main Components
- **ClaimsController**: Exposes REST endpoints (`POST /v1/claims`) for reporting claims.
- **ClaimsService**: Orchestrates claim intake, validation, policy cover checks, and event publication.
- **ClaimValidator**: Validates incoming claim payloads based on channel rules.
- **PolicyClient**: Communicates via HTTP with the `policy-admin` service (`GET /v1/policies/{id}`) with configured timeouts.
- **Publisher**: Publishes claim creation events to Google Cloud Pub/Sub (`claims.claim.reported`).

### Key Data Flows
1. Intake request received via `POST /v1/claims`.
2. Service validates input payload (`validate`).
3. Service queries `policy-admin` via `PolicyClient.get` to verify active policy status and peril cover.
4. Upon successful validation and excess calculation, `claims.claim.reported` event is published to Pub/Sub.
5. Returns claim submission response containing `claim_id`, `status: "submitted"`, and `excess_amount`.

### Running the Application
The application is built with NestJS and Node.js. Dependencies are managed via npm (`@nestjs/core`, `@google-cloud/pubsub`). Target runtime configuration relies on environment variables such as `POLICY_ADMIN_URL`.

<!-- okf:contents -->

## Contents

- [Concepts and flows](/concepts/index.md) — 2 pages. Flows, lifecycles and cross-cutting mechanisms.
- [Architecture decisions](/decisions/index.md) — 2 pages. One ADR per architecture decision the code or documents make evident.
- [Components and data models](/entities/index.md) — 5 pages. One page per significant component and core data model.
- [Interfaces and references](/summaries/index.md) — 1 page. API, event and module references for the application.
- [Change log](/log.md) — every generation and sync, newest first.

<!-- /okf:contents -->
