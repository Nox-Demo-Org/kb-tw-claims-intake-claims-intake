---
type: Component
title: PolicyClient
description: PolicyClient is the HTTP client responsible for integrating the claims intake workflow with the external policy-admin service.
resource: https://github.com/Nox-Demo-Org/kb-tw-claims-intake-claims-intake/blob/main/entities/policy-client.md
tags:
- claims-intake
- entities
sources:
- resource: https://github.com/Nox-Demo-Org/claims-intake/blob/HEAD/src/clients/policy.client.ts
generated:
  by: nox/gemini-3.7-flash
  at: '2026-10-05T12:43:59Z'
---

<!-- anchor: src/clients/policy.client.ts:L1-L14 -->

# PolicyClient

`PolicyClient` is the HTTP client responsible for integrating the claims intake workflow with the external `policy-admin` service. Defined in `src/clients/policy.client.ts`, it retrieves policy information to verify policy status, peril coverage, limits, and excess terms during claim processing in [[entities/claims-service]].

## Responsibilities

- **Policy Retrieval**: Executes HTTP requests to fetch policy data from the `policy-admin` endpoint `GET /v1/policies/{id}`.
- **Timeout Management**: Applies an HTTP timeout of 2,000 milliseconds (`2000` ms) using `AbortSignal.timeout(2000)` on every request.
- **Data Typing**: Deserializes and returns the response as a strongly-typed `PolicyView` object for downstream evaluation in [[concepts/claim-intake-flow]] and [[decisions/adr-policy-cover-excess-lookup]].
- **Endpoint Configuration**: Provides a default base URL fallback (`http://policy-admin/v1`) while allowing configuration overrides via environment variables.

## Interface and Models

### `PolicyView`

`PolicyView` represents the policy details received from `policy-admin`:

```typescript
export interface PolicyView {
  id: string;
  status: string;
  cover: { peril: string; limit_pence: number; excess_pence: number }[];
}
```

| Field | Type | Description |
|---|---|---|
| `id` | `string` | Unique identifier of the policy. |
| `status` | `string` | Current status of the policy (e.g., used to verify active coverage). |
| `cover` | `Array<{ peril: string; limit_pence: number; excess_pence: number }>` | List of covered perils, their respective coverage limits in pence (`limit_pence`), and excess amounts in pence (`excess_pence`). |

### Class Definition

```typescript
export class PolicyClient {
  constructor(private readonly base = process.env.POLICY_ADMIN_URL ?? "http://policy-admin/v1") {}
  async get(id: string): Promise<PolicyView>
}
```

- **`constructor(base?: string)`**: Initializes the base URL from the `POLICY_ADMIN_URL` environment variable, defaulting to `"http://policy-admin/v1"`.
- **`get(id: string): Promise<PolicyView>`**: Calls `${this.base}/policies/${id}` using standard `fetch` with an abort signal timeout of 2,000 ms, returning the JSON response.

## Dependencies

- **External Services**:
  - **`policy-admin`**: Consumes `GET /v1/policies/{id}` to fetch live policy contracts.
- **Internal Consumers**:
  - **[[entities/claims-service]]**: Calls `PolicyClient.get(policy_id)` to validate policy status and extract peril excess during claim creation.
- **Environment Configuration**:
  - `POLICY_ADMIN_URL`: Base HTTP URL pointing to the `policy-admin` API (defaults to `"http://policy-admin/v1"`).
