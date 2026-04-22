# Patterns Relevant to the MZinga Communications Evolution

---

## Patterns for the Monolith Itself

### Anti-Corruption Layer (ACL)

When the external worker starts consuming data from MZinga — whether via the database or the REST API — it will encounter MZinga's internal data model: Payload relationship references (`{ relationTo, value }`), Slate AST rich text, and Payload-specific pagination shapes. The ACL is a translation layer inside the worker that converts MZinga's model into the worker's own domain model. Without it, MZinga's internal conventions leak into the worker and create invisible coupling. If MZinga's schema changes, only the ACL needs updating, not the worker's core logic.

### Facade

Already partially present — `MailUtils.ts` is a thin facade over Nodemailer's `transport.sendMail`. The same pattern should be applied to the delivery channels in the worker: a `CommunicationChannel` interface with concrete implementations for SMTP, Slack, and any future channel. The worker's core logic calls the interface and never knows which transport is underneath.

---

## Patterns for the Migration Phase

### Branch by Abstraction

A companion to the Strangler Fig. Where the Strangler Fig works at the service boundary (routing traffic between old and new), Branch by Abstraction works inside the monolith's codebase. You introduce an abstraction over the capability you want to replace — in this case, an `EmailDispatcher` interface — make the existing SMTP code implement it, then build the new queue-based dispatcher behind the same interface. A feature flag or environment variable switches between them. This lets you test the new path in production with a subset of documents before removing the old implementation entirely. It is safer than directly replacing the hook body.

### Parallel Run

During the migration phase, both the old in-process SMTP sender and the new external worker process the same `Communications` document simultaneously. Their outputs are compared — delivery success, timing, recipient lists — but only one result is used (the old one, until the new one is trusted). This is how you build confidence in the new service without exposing users to its failures. It requires the `status` field and a comparison log, but no user-facing change.

### Feature Toggle (Feature Flag)

Controls which path is active — the in-process hook or the external worker — at runtime without a deployment. In MZinga's context this could be as simple as an environment variable:

```bash
COMMUNICATIONS_USE_EXTERNAL_WORKER=true
```

Combined with Branch by Abstraction, it gives you the ability to roll back instantly if the worker misbehaves in production.

---

## Patterns for the External Worker

### Outbox Pattern

Directly relevant to Step 1 and Step 2. When the `afterChange` hook writes `status: "pending"` to MongoDB, that write and the original document save happen in the same MongoDB operation — but if the worker crashes before reading the pending document, or if the status write fails after the document is saved, you have a consistency gap. The Outbox pattern formalises this: the monolith writes an explicit `outbox` collection entry in the same transaction as the document save. The worker reads from the outbox, processes, and deletes the entry. This guarantees at-least-once delivery without distributed transactions.

### Idempotent Consumer

A direct consequence of at-least-once delivery. If the worker crashes after sending the email but before writing `status: "sent"`, RabbitMQ will redeliver the message and the email will be sent twice. The worker must be idempotent — it checks whether `status` is already `sent` before processing, and uses a deduplication key (the document `id`) to detect redeliveries.

> This is not optional in any reliable messaging system.

### Dead Letter Queue (DLQ)

When a message fails processing repeatedly — for example because a user's email address is malformed or the Slack API is down — it should not block the queue. After a configurable number of retries, RabbitMQ moves the message to a dead letter exchange. The worker team monitors the DLQ, investigates failures, and replays messages manually or automatically after fixing the root cause.

> This is the missing retry and error handling that the current `Communications.ts` hook lacks entirely.

### Competing Consumers

Multiple instances of the worker subscribe to the same RabbitMQ queue. RabbitMQ distributes messages across them. This gives horizontal scalability with no coordination logic in the worker itself — relevant when `sendToAll` generates hundreds of pending communications simultaneously.

---

## Patterns for the Event-Driven Step

### Event Carried State Transfer

In Step 3, the worker receives a RabbitMQ message containing only `doc.id` and then calls `GET /api/communications/:id` to fetch the full document. This is the **Event Notification** sub-pattern — lightweight event, data fetched on demand. The alternative is **Event Carried State Transfer**: the full document payload is embedded in the RabbitMQ message itself (which `WebHooks.ts` lines 82–95 already does — `doc`, `data`, `operation`, `previousDoc` are all included). This eliminates the REST API call entirely, reducing latency and the number of moving parts, at the cost of larger message payloads and the risk of acting on stale data if the document is updated between publish and consume.

### Saga

If sending a communication involves multiple steps that can each fail independently — resolve recipients, render HTML, send SMTP, post to Slack, write status back — a Saga coordinates them as a sequence of compensatable steps. If the Slack post succeeds but the SMTP send fails, the Saga knows which steps to retry or roll back.

> In MZinga's current scope this is likely over-engineering, but it becomes relevant as soon as a single `Communications` document needs to dispatch to multiple channels and partial failure needs to be handled gracefully.

### Event Sourcing *(future consideration)*

Rather than storing only the current `status` of a `Communications` document, store every state transition as an immutable event: `created`, `pending`, `dispatching`, `sent`, `failed`, `retried`. The current state is derived by replaying the events. This gives a complete audit trail of every delivery attempt — who was notified, when, via which channel, and whether it succeeded — which is directly relevant for compliance in enterprise and SaaS contexts.

---

## Summary Map

| Phase | Pattern | Concern it solves |
|---|---|---|
| Monolith | Anti-Corruption Layer | Prevents MZinga's internal model leaking into the worker |
| Monolith | Facade | Abstracts delivery channel behind a common interface |
| Migration | Branch by Abstraction | Safe in-process switch between old and new dispatcher |
| Migration | Parallel Run | Validates new worker against old path before cutover |
| Migration | Feature Toggle | Runtime switch without redeployment, instant rollback |
| Worker | Outbox Pattern | Guarantees at-least-once delivery without distributed transactions |
| Worker | Idempotent Consumer | Prevents duplicate sends on message redelivery |
| Worker | Dead Letter Queue | Isolates poison messages, enables retry without blocking the queue |
| Worker | Competing Consumers | Horizontal scaling with no coordination logic |
| Event-driven | Event Carried State Transfer | Eliminates REST API call, reduces latency |
| Event-driven | Saga | Coordinates multi-channel dispatch with compensatable steps |
| Event-driven | Event Sourcing | Full audit trail of every delivery attempt |

---

**Previous:** [04 — The Strangler Fig Pattern](04-strangler-fig-pattern.md) · **Next:** [05b — Infrastructure Reference](05b-infrastructure-reference.md)
