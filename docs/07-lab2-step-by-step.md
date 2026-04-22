# Lab 2 — Step by Step: REST API Worker and Event-Driven Worker (States 2 and 3)

**Goal:** Evolve the Python worker built in Lab 1 through two further states.

- **Part A (core):** Remove the direct MongoDB dependency and replace it with calls to the MZinga REST API (State 2). At the end of Part A the worker has no knowledge of MongoDB and communicates exclusively through MZinga's published HTTP contract.
- **Part B (optional extension):** Remove polling entirely and replace it with a RabbitMQ subscription driven by MZinga's existing webhook hook infrastructure (State 3). Part B can be completed in a follow-up session.

---

## Prerequisites

### All platforms

- Lab 1 fully completed and verified:
  - MZinga running locally with `COMMUNICATIONS_EXTERNAL_WORKER=true`
  - The `Communications` collection has a `status` field with values `pending`, `processing`, `sent`, `failed`
  - The `lab1-worker/` folder exists with a working polling worker
- Python 3.11+ installed
- The `requests` library available (`pip install requests`)

### macOS

- No additional prerequisites beyond Lab 1

### Linux

- No additional prerequisites beyond Lab 1

### Windows — containers running inside WSL

- Node.js and Python must both be installed inside WSL
- All terminal commands in this lab must be run from the WSL terminal
- Docker Desktop must be configured to use the WSL 2 backend

### Windows — containers running outside WSL

- Node.js installed on Windows; `npm run dev` runs in PowerShell or Command Prompt
- Python installed on Windows; worker commands run in PowerShell
- Docker Desktop running with Windows containers or WSL 2 backend
- The venv activation command differs: use `.venv\Scripts\Activate.ps1` instead of `source .venv/bin/activate`
- If `Activate.ps1` is blocked, run `Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser` once in PowerShell

### Part B only — additional prerequisites

- RabbitMQ running (started as part of the Lab 1 infrastructure: `docker compose up database messagebus cache`)
- The `aio-pika` library available (`pip install aio-pika`)
- Access to the RabbitMQ management UI at `http://localhost:15672` (credentials: `guest` / `guest`)

---

## Starting point

Confirm before proceeding:

- MZinga is running (`npm run dev` in `mzinga-apps/`)
- The infrastructure is running (`docker compose up database messagebus cache`)
- Creating a Communication document in the admin UI results in `status: pending` and no email log in the MZinga terminal
- The Lab 1 worker, when running, picks up the document and marks it `sent`

---

## Part A — State 2: REST API Worker

### What changes and why

The Lab 1 worker connects directly to MongoDB. It must know the internal Payload document schema — the `{ relationTo, value }` relationship format, the raw ObjectId types, the collection names. Any schema change in MZinga breaks the worker silently.

In State 2 the worker talks exclusively to MZinga's auto-generated REST API. MZinga owns the data; the worker is a consumer of a published HTTP contract. The worker becomes schema-agnostic: it receives resolved data (email addresses, not ObjectIds) and writes back through the same API.

---

### Step A1 — Fix the `update` access rule on Communications

Open `src/collections/Communications.ts` and locate the `access` block. The `update` rule currently returns `false` unconditionally, which blocks all PATCH requests — including the ones the worker needs to write `status` back.

Change the `update` rule to use `access.GetIsAdmin`, the same access helper used for `read` and `create`. This allows any authenticated admin user — including the worker's service account — to update documents via the REST API, while still blocking unauthenticated requests.

Restart MZinga and verify: try a PATCH request to `/api/communications/:id` without a token — it should be rejected. With a valid admin JWT it should succeed.

---

### Step A2 — Understand the REST API shape

Before writing the worker, explore the auto-generated endpoints manually to understand the data shape the worker will receive.

All endpoints require a Bearer JWT. Obtain one by posting credentials to `/api/users/login`. The response contains a `token` field — use it as `Authorization: Bearer <token>` on all subsequent requests.

Explore the following endpoints and observe their responses:

- `GET /api/communications?where[status][equals]=pending&depth=1` — lists pending documents. The `depth=1` parameter tells Payload to resolve relationship references one level deep. `tos`, `ccs`, and `bccs` come back as full User objects with an `email` field, not raw ObjectIds. Compare this to the raw MongoDB document you inspected in Lab 1.
- `GET /api/communications/:id?depth=1` — fetches a single document with resolved relationships.
- `PATCH /api/communications/:id` — updates a document. Send `{ "status": "sent" }` in the body and confirm the change is reflected in the admin UI.

The key insight: with `depth=1`, the worker receives `value.email` directly from the API response. It no longer needs to query the `users` collection separately.

See the code snippets file for the exact curl commands to run these requests.

---

### Step A3 — Build the REST API worker

Create a new folder `lab2-worker-rest/` at the root of this lab repo (outside `mzinga/`).

**Project structure:**

```
lab2-worker-rest/
├── worker.py
├── requirements.txt
└── .env
```

**Dependencies** — add to `requirements.txt`:

- `requests` — HTTP client for the MZinga REST API (version 2.32.3)
- `python-dotenv` — loads `.env` files (version 1.0.1)

**`.env`** — configure with your MZinga admin credentials and SMTP settings. See the code snippets file for the full content.

**`worker.py`** — implement the following logic:

1. **Authenticate** against `POST /api/users/login` with the admin credentials from the environment. Store the returned JWT token.

2. **Poll for pending documents** by calling `GET /api/communications?where[status][equals]=pending&depth=1`. If the response contains no documents, sleep for `POLL_INTERVAL_SECONDS` and retry.

3. **For each pending document**, call `PATCH /api/communications/:id` to set `status: "processing"` before doing any work.

4. **Extract email addresses** from the `tos`, `ccs`, and `bccs` fields. With `depth=1`, each relationship entry has a `value` object containing an `email` field — no MongoDB query needed.

5. **Serialise the Slate AST body to HTML** using the same recursive function from Lab 1.

6. **Send the email** via `smtplib` using the same approach as Lab 1.

7. **Write back the result** via `PATCH /api/communications/:id` — `status: "sent"` on success, `status: "failed"` on exception.

8. **Handle token expiry**: if any API call returns HTTP 401, re-authenticate and retry.

**Install and run:**

**macOS / Linux / WSL:**

```sh
cd lab2-worker-rest
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
python worker.py
```

**Windows PowerShell:**

```powershell
cd lab2-worker-rest
python -m venv .venv
.venv\Scripts\Activate.ps1
pip install -r requirements.txt
python worker.py
```

---

### Step A4 — Verify State 2

1. Stop the Lab 1 worker if still running
2. Start the REST API worker
3. Create a Communication document in the admin UI
4. The MZinga hook writes `status: pending`
5. The worker polls, fetches the document via REST with `depth=1`, sends the email, writes `status: sent` back via PATCH
6. Confirm in the admin UI that `status` shows `Sent`
7. Confirm there is no MongoDB connection string anywhere in `lab2-worker-rest/`
8. If using MailHog, confirm the email appears at `http://localhost:8025`

---

### What changed from Lab 1

| Concern | Lab 1 (State 1) | Lab 2 Part A (State 2) |
|---|---|---|
| Data access | Direct MongoDB (`pymongo`) | MZinga REST API (`requests`) |
| Relationship resolution | Manual ObjectId lookup in `users` collection | Automatic via `depth=1` query parameter |
| Status write-back | `db.communications.update_one(...)` | `PATCH /api/communications/:id` |
| Schema coupling | MongoDB field names and BSON types | HTTP response shape (JSON) |
| Auth | None (direct DB access) | JWT Bearer token |

### Known limitations (addressed in Part B)

- The worker still **polls** on an interval — delivery is delayed by up to `POLL_INTERVAL_SECONDS` after a document is saved.
- Polling generates constant HTTP traffic against the MZinga API even when there is nothing to process.
- Multiple worker instances would process the same `pending` document concurrently — the REST API has no atomic claim operation.

---

## Part B — State 3: Event-Driven Worker via RabbitMQ *(optional extension)*

> **This part is optional.** Part A is the required deliverable for the lab session. Complete Part B if you finish Part A early or in a follow-up session.

### What changes and why

The polling loop is the last remaining inefficiency. The worker wakes up on a timer, asks "is there anything to do?", and usually the answer is no. This wastes resources and introduces latency.

MZinga already has the infrastructure to publish events to RabbitMQ whenever a collection document changes — the `WebHooks` system in `src/hooks/WebHooks.ts`. Setting a single environment variable is enough to make MZinga publish a message to RabbitMQ every time a `Communications` document is saved. The worker subscribes to that exchange and reacts immediately, with no polling.

The monolith requires **zero code changes** for this step.

---

### Step B1 — Understand the MZinga WebHooks event system

Read `src/hooks/WebHooks.ts` carefully. Locate the `AddHooksFromList` method and understand how it reads environment variables of the form `HOOKSURL_<COLLECTION_SLUG>_<HOOK_TYPE>`. When the value is `rabbitmq` and `RABBITMQ_URL` is set, it attaches a hook that publishes to RabbitMQ after every matching lifecycle event.

Read `src/messageBusService.ts` and identify:
- The two exchanges declared: `mzinga_events` (transient) and `mzinga_events_durable` (durable, persistent)
- The binding between them with routing key `#` — every event published to `mzinga_events` is automatically forwarded to `mzinga_events_durable`
- The routing key used when publishing: it is the environment variable name itself (e.g. `HOOKSURL_COMMUNICATIONS_AFTERCHANGE`)

Understand the published message structure — the event body wraps the document under `data.doc`, and includes `data.operation` which is either `"create"` or `"update"`. See the code snippets file for the full JSON shape.

The worker must subscribe to `mzinga_events_durable` (not `mzinga_events`) so that messages queued while the worker is down are not lost.

---

### Step B2 — Configure MZinga to publish Communications events

In `mzinga-apps/.env`, add two variables:

- `RABBITMQ_URL` — the connection string for the local RabbitMQ instance
- `HOOKSURL_COMMUNICATIONS_AFTERCHANGE=rabbitmq` — instructs the WebHooks system to attach a RabbitMQ publisher hook to the `Communications` collection's `afterChange` event

Restart MZinga. Confirm in the terminal that the RabbitMQ connection is established successfully.

> The `afterChange` hook from Lab 1 (`COMMUNICATIONS_EXTERNAL_WORKER=true`) still runs and writes `status: pending`. The WebHooks system adds its own hook on top — both run. The `pending` write is now redundant but harmless — it provides a visible status in the admin UI.

---

### Step B3 — Inspect the event on RabbitMQ

Before writing the worker, verify the event is being published correctly.

Open the RabbitMQ management UI at `http://localhost:15672` (credentials: `guest` / `guest`). Navigate to **Exchanges** and confirm both `mzinga_events` and `mzinga_events_durable` exist. Create a Communication document in the admin UI and watch the message rate counters on the exchanges.

Use the existing `servicebus-subscriber` example in `mzinga/mzinga-apps/examples/servicebus-subscriber/` to print the raw event to the terminal. Install its dependencies and run it with `ROUTING_KEY=HOOKSURL_COMMUNICATIONS_AFTERCHANGE`. Create a Communication document and observe the full event payload. Note:

- The routing key matches the environment variable name exactly
- The document is nested at `data.doc`
- The operation is `"create"` for a new document and `"update"` for any subsequent change

See the code snippets file for the exact commands to run the subscriber.

---

### Step B4 — Build the event-driven worker

Create a new folder `lab2-worker-events/` at the root of this lab repo.

**Project structure:**

```
lab2-worker-events/
├── worker.py
├── requirements.txt
└── .env
```

**Dependencies** — add to `requirements.txt`:

- `aio-pika` — async RabbitMQ client for Python (version 9.5.5)
- `requests` — HTTP client for the MZinga REST API (version 2.32.3)
- `python-dotenv` — loads `.env` files (version 1.0.1)

**`.env`** — configure with RabbitMQ connection details, exchange and queue names, MZinga admin credentials, and SMTP settings. See the code snippets file for the full content. Key values:

- `EXCHANGE_NAME` must be `mzinga_events_durable` — the durable exchange, not the transient one
- `QUEUE_NAME` should be a fixed name (e.g. `communications-email-worker`) so the queue persists across worker restarts and messages are not lost
- `ROUTING_KEY` must be `HOOKSURL_COMMUNICATIONS_AFTERCHANGE` — exactly matching the env variable name MZinga uses as the routing key

**`worker.py`** — implement the following logic:

1. **Authenticate** against the MZinga REST API using the same login approach as Part A.

2. **Connect to RabbitMQ** using `aio_pika.connect_robust` — this handles automatic reconnection on network interruptions.

3. **Declare the exchange** with the same parameters MZinga uses: topic type, durable, internal, no auto-delete. The declaration must match exactly or RabbitMQ will reject it.

4. **Declare a named durable queue** and bind it to the exchange with the routing key. A named durable queue survives worker restarts — messages sent while the worker is down are delivered when it reconnects.

5. **Set `prefetch_count=1`** on the channel so RabbitMQ delivers one message at a time per worker instance. This makes running multiple instances safe without any coordination logic.

6. **Consume messages** in an async loop. For each message:
   - Parse the JSON body
   - Extract `data.operation` and `data.doc.id`
   - **Filter out `"update"` operations** — the worker's own `PATCH` status write-back triggers another `afterChange` event with `operation: "update"`. Without this filter the worker would process the same document in an infinite loop.
   - Fetch the full document via `GET /api/communications/:id?depth=1`
   - Apply the idempotency guard: skip if `status` is already `"sent"` or `"processing"`
   - Process and send the email using the same logic as Part A
   - Acknowledge the message only after processing completes

7. **Handle token expiry**: if a REST API call returns HTTP 401, re-authenticate and retry.

**Install and run:**

**macOS / Linux / WSL:**

```sh
cd lab2-worker-events
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
python worker.py
```

**Windows PowerShell:**

```powershell
cd lab2-worker-events
python -m venv .venv
.venv\Scripts\Activate.ps1
pip install -r requirements.txt
python worker.py
```

---

### Step B5 — Verify State 3

1. Stop the REST API worker from Part A if still running
2. Start the event-driven worker
3. Create a Communication document in the admin UI
4. Observe in the worker terminal: the message arrives immediately without any polling delay
5. Observe `status` transitions in the admin UI: `pending` → `processing` → `sent`
6. Stop the worker, create another Communication document, then restart the worker — the message should be waiting in the durable queue and processed immediately on reconnect
7. Open the RabbitMQ management UI at `http://localhost:15672`, navigate to **Queues**, and confirm the `communications-email-worker` queue exists, is durable, and shows zero unacknowledged messages after processing

---

### What changed from Part A

| Concern | Part A (State 2) | Part B (State 3) |
|---|---|---|
| Trigger mechanism | Polling (`time.sleep` loop) | RabbitMQ subscription (push) |
| Latency | Up to `POLL_INTERVAL_SECONDS` | Near-zero (event-driven) |
| Idle load | Constant HTTP requests to MZinga API | Zero — worker sleeps until a message arrives |
| Message durability | None — pending docs accumulate silently if worker is down | Durable queue — messages survive worker restarts |
| MZinga code change | None | None — only `.env` variable added |
| Concurrent workers | Race condition on `pending` poll | Safe — RabbitMQ delivers each message to exactly one consumer |

---

## Evolution summary across all labs

| | Lab 1 (State 1) | Lab 2 Part A (State 2) | Lab 2 Part B (State 3) |
|---|---|---|---|
| Data access | Direct MongoDB | MZinga REST API | MZinga REST API |
| Trigger | Polling DB query | Polling HTTP GET | RabbitMQ push |
| Coupling | MongoDB schema | HTTP contract | Routing key only |
| Relationship resolution | Manual ObjectId lookup | `depth=1` auto-resolve | `depth=1` auto-resolve |
| Auth | None | JWT Bearer | JWT Bearer |
| Durability | None | None | Durable queue |

---

**Previous:** [06b — Lab 1 Code Snippets](06-lab1-code-snippets.md) · **Code snippets:** [07b — Lab 2 Code Snippets](07-lab2-code-snippets.md)
