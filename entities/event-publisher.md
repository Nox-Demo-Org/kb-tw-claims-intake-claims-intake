---
type: Component
title: Event Publisher
description: The Event Publisher module located at src/events/publisher.ts handles publishing asynchronous domain events from claims-intake to Google Cloud Pub/Sub.
resource: https://github.com/Nox-Demo-Org/kb-tw-claims-intake-claims-intake/blob/main/entities/event-publisher.md
tags:
- claims-intake
- entities
sources:
- resource: https://github.com/Nox-Demo-Org/claims-intake/blob/HEAD/src/events/publisher.ts
generated:
  by: nox/gemini-3.7-flash
  at: '2026-10-05T12:43:59Z'
---

<!-- anchor: src/events/publisher.ts:L1-L5 -->

# Event Publisher

The `Event Publisher` module located at `src/events/publisher.ts` handles publishing asynchronous domain events from `claims-intake` to Google Cloud Pub/Sub. It facilitates downstream communication to services such as `claims-management` and `fraud-scoring` as part of the [[decisions/adr-event-driven-intake|event-driven intake architecture]].

## Responsibilities

- Initialize and maintain the Google Cloud Pub/Sub client instance (`PubSub`).
- Provide an asynchronous `publish` function to publish JSON messages to designated Pub/Sub topics.
- Enforce compile-time topic restrictions, currently typing the topic literal to `"claims.claim.reported"`.

## Implementation Details

The module exports a single helper function:

```typescript
export async function publish(topic: "claims.claim.reported", payload: unknown): Promise<void>
```

When invoked by [[entities/claims-service]] as part of the [[concepts/claim-intake-flow|claim intake workflow]], the publisher targets the requested topic and emits the payload using:

```typescript
await pubsub.topic(topic).publishMessage({ json: payload });
```

The payload structure for the `"claims.claim.reported"` event is defined in [[entities/claim]] and documented in [[summaries/api-spec]].

## Dependencies

- **`@google-cloud/pubsub`**: Provides the `PubSub` client used to interface with Google Cloud Pub/Sub topics.
- **Used by**:
  - [[entities/claims-service]]: Invokes `publish` after validating claims and verifying policy coverage to broadcast the `claims.claim.reported` event.
