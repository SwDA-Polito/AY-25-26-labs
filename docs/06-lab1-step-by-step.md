# Lab 1 — Step by Step: DB-Coupled External Email Worker (State 1)

**Goal:** Extract email sending from MZinga into a Python worker that reads directly from MongoDB. Disable the in-process email sending in MZinga via a feature flag. At the end of this lab, MZinga saves the document and marks it `pending`; the Python worker picks it up, sends the email, and writes the final status back.

This is **State 1** of the architecture journey: Strangler Fig into a DB-Coupled External Worker.

---

## Prerequisites

Before starting, ensure you have installed:

- [Node.js](https://nodejs.org) v24 (check `.nvmrc` — the repo targets `24.10`)
- [npm](https://www.npmjs.com) (bundled with Node.js)
- [Docker Desktop](https://docs.docker.com/desktop/)
- [Git](https://git-scm.com/downloads)
- [Python 3.11+](https://www.python.org/downloads/)
- [mongosh](https://www.mongodb.com/docs/mongodb-shell/install/) (MongoDB shell, for database inspection)

---

## Step 1 — Clone and set up MZinga locally

### 1.1 Clone the repository

Inside the `mzinga/` folder of this lab repo (already in `.gitignore`):

```sh
cd mzinga
git clone https://github.com/mzinga-io/mzinga-apps.git
cd mzinga-apps
```

### 1.2 Install dependencies

```sh
npm install
```

### 1.3 Configure the environment

Copy the template and fill in the required values:

```sh
cp .env.template .env
```

Edit `.env` with the following minimum configuration for local development:

```sh
DISABLE_TRACING=1
MONGO_PORT=27017
MONGODB_URI="mongodb://admin:admin@localhost:27017/mzinga?authSource=admin&directConnection=true"
PAYLOAD_SECRET=r3pl4c3m3w1thv4l1ds3cr3t
TENANT=local-tenant
ENV=prod
DRIVER_OPTS_DEVICE=/tmp
DRIVER_OPTS_TYPE="none"
DRIVER_OPTS_OPTIONS="bind"
MONGO_HOST=<your_local_ip_usually_192.something_or_172.something>        # run: ifconfig | grep 192 or ifconfig | grep 172
CORS_CONFIGS=*
PAYLOAD_PUBLIC_SERVER_URL=http://localhost:3000
DEBUG_EMAIL_SEND=1                    # logs email content to console instead of sending
```

> `DEBUG_EMAIL_SEND=1` activates an existing flag in `MailUtils.ts` that logs the email payload to the console without actually calling the SMTP transport. This lets you verify the email flow without a real SendGrid key.

> `MONGO_HOST` must be your machine's current LAN IP (not `localhost`). It is used by the MongoDB replica set healthcheck inside Docker. Run `ifconfig | grep 192` to find it — it changes when you switch networks.

### 1.4 Start the infrastructure (MongoDB + RabbitMQ + Redis)

Prepare the volume directories and start only the infrastructure services:

```sh
rm -rf /tmp/database /tmp/mzinga /tmp/messagebus
mkdir -p /tmp/database /tmp/mzinga /tmp/messagebus
docker compose up database messagebus cache
```

Wait until you see the MongoDB replica set initialised in the logs before proceeding.

NOTE-1: if the database does not start with "unable to load security key" or similar error, please refer to the docker-compose-simplified.yaml file for a simplified version of the docker compose file that remove the need to run a replicaset

NOTE-2: if you already have a mongodb instance running locally, you can avoid entirely the docker compose command

### 1.5 Start MZinga

In a separate terminal:

```sh
npm run dev
```

Open [http://localhost:3000/admin](http://localhost:3000/admin) and create the first admin user when prompted.

### 1.6 Verify the setup

- Admin UI loads at `http://localhost:3000/admin`
- The **Communications** collection is visible under the **Notifications** group in the sidebar
- Create a test Communication document (you need at least one User to send to — create one first under the Users collection)
- With `DEBUG_EMAIL_SEND=1` and no `SENDGRID_API_KEY` set, the email content is logged to the terminal and no real email is sent

---

## Step 2 — Understand the current email flow

Before changing anything, read the code that you are about to replace.

### 2.1 The `afterChange` hook

Open `src/collections/Communications.ts`. The hook at **line 36** fires synchronously every time a `Communications` document is created or updated. Read through it and identify the five things it does in sequence:

1. Resolves upload attachments in the rich-text body (lines 37–47)
2. Serialises the Slate AST body to HTML via `TextUtils.Serialize` (line 48)
3. Resolves `tos` relationship references to actual email addresses (lines 50–58)
4. Resolves `ccs` and `bccs` the same way (lines 62–79)
5. Builds one SMTP message per recipient and dispatches them all with `Promise.all` (lines 80–95)

The HTTP request that saved the document does not return until every SMTP call settles. This is the blocking behaviour you are extracting.

### 2.2 The `MailUtils` flag

Open `src/utils/MailUtils.ts`. Locate the `DEBUG_EMAIL_SEND` check (lines 6–9). Understand how it works: it reads an environment variable and, when set to `1`, logs the message instead of sending it. This is the pattern you will extend with a new feature flag to disable sending entirely.

### 2.3 Inspect the MongoDB document shape

With MZinga running and a Communication document created, connect to MongoDB and inspect the raw document:

```sh
mongosh "mongodb://admin:admin@localhost:27017/mzinga?authSource=admin&directConnection=true"
```

```js
use mzinga
db.communications.findOne()
```

Observe the document structure:

```json
{
  "_id": ObjectId("..."),
  "subject": "Test subject",
  "body": [ /* Slate AST nodes */ ],
  "tos": [
    { "relationTo": "users", "value": ObjectId("...") }
  ],
  "ccs": null,
  "bccs": null,
  "sendToAll": false,
  "createdAt": ISODate("..."),
  "updatedAt": ISODate("...")
}
```

Key observations:
- `tos`, `ccs`, `bccs` store **relationship references** — `{ relationTo: "users", value: <ObjectId> }` — not email addresses directly. The Python worker must resolve them by querying the `users` collection.
- `body` is a **Slate AST** — an array of node objects. The Python worker must convert this to HTML to build the email body.
- There is **no `status` field** yet. You will add one in Step 3.

Also inspect the users collection to understand the email field:

```js
db.users.findOne({}, { email: 1 })
```

---

## Step 3 — Add a `status` field to the Communications collection

The current `Communications` collection has no `status` field. The Python worker needs one to know which documents to process and to write back the result.

### 3.1 Required statuses

| Value | Set by | Meaning |
|---|---|---|
| `pending` | MZinga `afterChange` hook | Document saved, waiting for the worker to process it |
| `processing` | Python worker | Worker has picked up the document and is sending |
| `sent` | Python worker | All emails dispatched successfully |
| `failed` | Python worker | Sending failed; see logs for details |

### 3.2 Add the field to `Communications.ts`

In `src/collections/Communications.ts`, add a new field to the `fields` array with the following characteristics:
- name: `status`
- type: `select` with the four options above
- marked as `readOnly` in the admin UI
- positioned in the sidebar

Also add `status` to the `defaultColumns` list in the `admin` block so it is visible in the collection list view.

Restart MZinga and verify the field appears in the admin UI on both the list view and the document sidebar.

---

## Step 4 — Disable in-process email sending via a feature flag

You will replace the entire body of the `afterChange` hook with a status write, gated by an environment variable so you can switch between the old and new behaviour without a code change.

### 4.1 Add the flag to `.env`

```sh
COMMUNICATIONS_EXTERNAL_WORKER=true
```

### 4.2 Modify the `afterChange` hook

In `src/collections/Communications.ts`, modify the `afterChange` hook body so that:

- When `COMMUNICATIONS_EXTERNAL_WORKER` is not `"true"`, the original email sending logic runs unchanged (keep all existing lines 37–103 intact inside this branch)
- When `COMMUNICATIONS_EXTERNAL_WORKER` is `"true"`, the hook instead calls `payload.update` to write `status: "pending"` on the document and returns immediately

> This is the **Branch by Abstraction** pattern: the abstraction boundary is the environment variable. The old path is preserved and reachable by setting `COMMUNICATIONS_EXTERNAL_WORKER=false`. You can roll back instantly.

### 4.3 Verify the flag works

With `COMMUNICATIONS_EXTERNAL_WORKER=true`:
- Create a new Communication document in the admin UI
- The HTTP request should return immediately (no SMTP delay)
- The document should show `status: pending` in the admin UI
- No email log should appear in the terminal

With `COMMUNICATIONS_EXTERNAL_WORKER=false` (or unset):
- The original behaviour is restored: email is logged to the console (because `DEBUG_EMAIL_SEND=1`) and the request blocks until done

---

## Step 5 — Build the Python worker

Create a new folder `lab1-worker/` at the root of this lab repo (outside `mzinga/`).

### 5.1 Project structure

```
lab1-worker/
├── worker.py
├── requirements.txt
└── .env
```

### 5.2 Dependencies

The worker needs two libraries. Add them to `requirements.txt`:

- `pymongo` — MongoDB driver for Python (version 4.10.1)
- `python-dotenv` — loads `.env` files into environment variables (version 1.0.1)

### 5.3 `.env`

```sh
MONGODB_URI=mongodb://admin:admin@localhost:27017/mzinga?authSource=admin&directConnection=true
POLL_INTERVAL_SECONDS=5
SMTP_HOST=localhost
SMTP_PORT=1025
EMAIL_FROM=worker@mzinga.io
```

> For local testing without a real SMTP server, use [MailHog](https://github.com/mailhog/MailHog):
> ```sh
> docker run -d -p 1025:1025 -p 8025:8025 mailhog/mailhog
> ```
> Sent emails appear at `http://localhost:8025`.

### 5.4 `worker.py` — what to implement

Write a Python script that does the following in a loop:

1. **Connect to MongoDB** using `pymongo` and the `MONGODB_URI` from the environment. The database name is `mzinga` (already in the URI).

2. **Poll for pending documents.** Query the `communications` collection for one document where `status` equals `"pending"`. If none is found, sleep for `POLL_INTERVAL_SECONDS` and try again.

3. **Claim the document** by immediately updating its `status` to `"processing"` before doing any work. This prevents two worker instances from processing the same document.

4. **Resolve recipient email addresses.** The `tos`, `ccs`, and `bccs` fields contain Payload relationship references in the form `{ "relationTo": "users", "value": <ObjectId> }`. Query the `users` collection to resolve the ObjectIds to actual email addresses.

5. **Serialise the body to HTML.** The `body` field is a Slate AST — a list of node objects with a `type` and `children`. Write a recursive function that converts the node tree to an HTML string. Handle at minimum: `paragraph`, `h1`, `h2`, `ul`, `li`, `link`, and leaf text nodes with `bold` and `italic` marks.

6. **Send the email** using Python's built-in `smtplib`. Build a `MIMEMultipart` message with the resolved `to`, `cc`, `bcc`, `subject`, and HTML body, then send it via the configured SMTP host and port.

7. **Write back the result.** On success, update the document's `status` to `"sent"`. On any exception, update it to `"failed"` and log the error.

### 5.5 Install dependencies and run

```sh
cd lab1-worker
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
python worker.py
```

---

## Step 6 — End-to-end verification

With both MZinga and the Python worker running:

1. Open the admin UI at `http://localhost:3000/admin`
2. Create a new Communication document with a valid recipient user
3. Save the document
4. Observe in the MZinga terminal: the request returns immediately, no email log
5. Observe in the worker terminal: the document is picked up, processed, and marked `sent`
6. In the admin UI, refresh the document — `status` should show `Sent`
7. If using MailHog, open `http://localhost:8025` to see the delivered email

To test failure handling, temporarily stop the worker, create a Communication, then restart the worker — it should pick up the `pending` document and process it.

---

## What you have built

| Concern | Implementation |
|---|---|
| Transition strategy | Strangler Fig — old hook preserved behind a feature flag |
| Feature flag | `COMMUNICATIONS_EXTERNAL_WORKER=true` in `.env` |
| New status field | `pending` → `processing` → `sent` / `failed` |
| Worker integration | Shared Database (direct MongoDB access) |
| Worker consumption model | Polling Consumer (interval-based query) |
| Rollback | Set `COMMUNICATIONS_EXTERNAL_WORKER=false`, restart MZinga |

## Known limitations (addressed in Lab 2)

- The worker is **tightly coupled to the MongoDB schema**. Any field rename in `Communications.ts` breaks the worker directly.
- The `tos` relationship resolution duplicates logic already in MZinga — the worker must know the internal Payload relationship format `{ relationTo, value }`.
- The Slate AST serialiser in Python is a manual reimplementation of `TextUtils.Serialize` from TypeScript — it must be kept in sync manually.
- There is **no retry logic** beyond the `failed` status — a failed document stays failed until manually reset to `pending`.
- The worker **does not handle concurrent instances** safely — two workers could pick up the same `pending` document simultaneously.

---

**Previous:** [05 — Supporting Patterns Catalogue](05-supporting-patterns-catalogue.md) · **Next:** [07 — Lab 2 Step by Step](07-lab2-step-by-step.md)
