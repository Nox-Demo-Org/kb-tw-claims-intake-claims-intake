# Components and data models

One page per significant component and core data model.

## Pages

- [Claim Data Models and DTOs](/entities/claim.md) — The src/claims/claim.dto.ts file defines the core data transfer objects (DTOs) and event schemas for claim ingestion and event publishing within claims-intake.
- [ClaimsController](/entities/claims-controller.md) — ClaimsController is a NestJS REST controller located in src/claims/claims.controller.ts.
- [ClaimsService](/entities/claims-service.md) — The ClaimsService class (src/claims/claims.service.ts) orchestrates the First Notice of Loss (FNOL) business logic in claims-intake.
- [Event Publisher](/entities/event-publisher.md) — The Event Publisher module located at src/events/publisher.ts handles publishing asynchronous domain events from claims-intake to Google Cloud Pub/Sub.
- [PolicyClient](/entities/policy-client.md) — PolicyClient is the HTTP client responsible for integrating the claims intake workflow with the external policy-admin service.
