---
type: Component
title: ClaimsController
description: ClaimsController is a NestJS REST controller located in src/claims/claims.controller.ts.
resource: https://github.com/Nox-Demo-Org/kb-tw-claims-intake-claims-intake/blob/main/entities/claims-controller.md
tags:
- claims-intake
- entities
sources:
- resource: https://github.com/Nox-Demo-Org/claims-intake/blob/HEAD/src/claims/claims.controller.ts
- resource: https://github.com/Nox-Demo-Org/claims-intake/blob/HEAD/src/claims/claim.dto.ts
- resource: https://github.com/Nox-Demo-Org/claims-intake/blob/HEAD/src/claims/claims.service.ts
generated:
  by: nox/gemini-3.7-flash
  at: '2026-10-05T12:43:59Z'
---

<!-- anchor: src/claims/claims.controller.ts:L1-L14 -->
<!-- anchor: src/claims/claim.dto.ts:L1-L20 -->
<!-- anchor: src/claims/claims.service.ts:L1-L25 -->

# ClaimsController

`ClaimsController` is a NestJS REST controller located in `src/claims/claims.controller.ts`. It provides the HTTP entry point for reporting new claims into the FNOL (First Notice of Loss) service.

## Responsibilities

- **Expose Claim Reporting Endpoint**: Defines the `POST /v1/claims` route using NestJS `@Controller("v1/claims")` and `@Post()` decorators.
- **Request Ingestion**: Receives and binds the incoming HTTP request body to the [[entities/claim|NewClaim]] DTO.
- **Service Delegation**: Forwards the intake payload directly to [[entities/claims-service|ClaimsService.report]] for validation, policy cover checks, and event publication.
- **Response Delivery**: Returns the result of the submission back to the caller in the format `{ claim_id: string, status: "submitted", excess_amount: number }`.

## Endpoints

### `POST /v1/claims`

Receives a new claim submission payload and initiates the intake process.

- **Request Body**: [[entities/claim|NewClaim]]
  - `policy_id`: `string`
  - `customer_id`: `string`
  - `peril`: `"escape_of_water" | "storm" | "theft" | "fire" | "accidental_damage" | "collision"`
  - `incident_date`: `string`
  - `description`: `string`
  - `channel`: `"web" | "phone"`
- **Response**:
  ```json
  {
    "claim_id": "string (UUID)",
    "status": "submitted",
    "excess_amount": 0
  }
  ```
- **Handler Method**: `create(@Body() body: NewClaim)`

For API contract details, see [[summaries/api-spec]].

## Dependencies

- **[[entities/claims-service|ClaimsService]]**: Injected via constructor (`private readonly claims: ClaimsService`) to execute the intake flow (`report(body)`).
- **[[entities/claim|NewClaim]]**: DTO imported from `src/claims/claim.dto.ts` representing the claim request body.
- **`@nestjs/common`**: Provides decorators `@Controller`, `@Post`, and `@Body`.

## Related Links
- [[concepts/claim-intake-flow]] - End-to-end flow triggered by this controller
- [[entities/claims-service]] - Business logic and orchestration service
- [[entities/claim]] - Data transfer object definitions
