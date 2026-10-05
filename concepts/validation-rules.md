---
type: Concept
title: Validation Rules
description: The claim validation logic is implemented in src/claims/claim.validator.ts and evaluated during the claim-intake-flow before policy cover is checked in claims-service.
resource: https://github.com/Nox-Demo-Org/kb-tw-claims-intake-claims-intake/blob/main/concepts/validation-rules.md
tags:
- claims-intake
- concepts
generated:
  by: nox/gemini-3.7-flash
  at: '2026-10-05T12:43:59Z'
---

# Validation Rules

The claim validation logic is implemented in `src/claims/claim.validator.ts` and evaluated during the [[concepts/claim-intake-flow]] before policy cover is checked in [[entities/claims-service]].

## Input Model

Validation operates on incoming [[entities/claim|NewClaim]] payload objects:

```typescript
export interface NewClaim {
  policy_id: string;
  customer_id: string;
  peril: "escape_of_water" | "storm" | "theft" | "fire" | "accidental_damage" | "collision";
  incident_date: string;
  description: string;
  channel: "web" | "phone";
}
```

## Validation Logic

The `validate(c: NewClaim)` function enforces basic presence checks on the claim payload:

1. **Policy Identifier Check**: 
   - `policy_id` must be present.
   - If missing or falsy, throws `Error("policy_id is required")`.

2. **Channel-Specific Evaluation**:
   - `channel === "phone"`: The validator executes an early return and skips subsequent peril validation.
   - `channel === "web"` (or any non-phone channel): The validator requires `peril`. If missing or falsy, throws `Error("peril is required")`.

## Channel Differences

| Channel | `policy_id` Required | `peril` Required |
| :--- | :--- | :--- |
| `web` | Yes (`"policy_id is required"`) | Yes (`"peril is required"`) |
| `phone` | Yes (`"policy_id is required"`) | No (validator returns early) |

## Known Gaps and Limitations

The `claim.validator.ts` implementation contains documented limitations and technical debt marked in the codebase:

- **Peril validation bypass**: The `phone` intake channel skips peril validation entirely (`FIXME: phone channel skips peril validation entirely`).
- **Future incident dates**: No check exists to prevent `incident_date` values in the future (`TODO: incident_date in the future is accepted`).
- **Policy period alignment**: The validator does not verify whether `incident_date` falls inside the policy validity window (`TODO: no check that incident_date is inside the policy period`).
- **Description payload size**: `description` does not enforce maximum string length limits (`TODO: description has no length limit (we have seen 40k-character pastes)`).
- **Duplicate detection**: There is no duplicate check to identify multiple claim submissions for the same incident (`TODO: duplicate claims for the same incident are not detected`).
