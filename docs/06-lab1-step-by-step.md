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

### 1.1 Fork this repository and configure `.gitignore`

This lab repo has `mzinga/` listed in `.gitignore` to prevent accidentally pushing the MZinga source code to the shared lab repository. Before cloning MZinga, fork this repo to your own GitHub/GitLab account so you have a personal copy where you can commit everything — including the MZinga code and your worker — without affecting the shared repo.

After forking and cloning your fork locally, open `.gitignore` at the root of the repo and remove the `mzinga/` line:

```sh
# remove this line from .gitignore:
mzinga/
```

Commit the change:

```sh
git add .gitignore
git commit -m "chore: allow mzinga folder to be tracked in personal fork"
```

From this point your fork will track the `mzinga/` folder and all changes you make inside it.

### 1.2 Clone the MZinga repository

Inside the `mzinga/` folder of this lab repo:

```sh
cd mzinga
git clone https://github.com/mzinga-io/mzinga-apps.git
cd mzinga-apps
```

### 1.3 Install dependencies

```sh
npm install
```

### 1.4 Choose which docker-compose file to use

Two compose files are provided in the `docs/` folder of this lab repo. Copy the one that matches your situation into `mzinga-apps/` before running any `docker compose` command.

**`docker-compose-original.yml`** — uses a MongoDB replica set secured with a key file. Required for full Payload CMS functionality (change streams, transactions). More complex to set up; can fail with permission errors on the key file depending on the OS and Docker configuration.

**`docker-compose-simplified.yml`** — uses a plain MongoDB instance with no replica set and no key file. Simpler and more portable across all platforms. Sufficient for this lab.

**Start with the simplified file.** Switch to the original only if you have a specific reason to need replica set features.

```sh
cp ../../docs/docker-compose-simplified.yml docker-compose.yml
```

> This overwrites the existing `docker-compose.yml` in `mzinga-apps/`. The original is preserved in `docs/docker-compose-original.yml`.

---

### 1.5 Configure the environment

Copy the template:

```sh
cp .env.template .env
```

Two values require OS-specific attention: `MONGO_HOST` and `DRIVER_OPTS_DEVICE`.

**`MONGO_HOST`** is the IP address of your machine as seen from inside Docker containers. It is used by the MongoDB healthcheck. It is never `localhost` or `127.0.0.1` — those resolve to the container itself, not your host.

**`DRIVER_OPTS_DEVICE`** is the absolute path on your host where Docker will bind-mount persistent volume data (MongoDB, RabbitMQ, MZinga uploads). The directory must exist before you run `docker compose`.

| Setting | macOS | Linux | Windows — containers in WSL | Windows — containers outside WSL |
|---|---|---|---|---|
| How to find `MONGO_HOST` | `ifconfig \| grep 192` or `ifconfig \| grep 172` — use the `inet` value | `ip addr \| grep 192` or `ip addr \| grep 172` | Run `ip addr` inside WSL — use the `eth0` inet value (typically `172.x.x.x`) | Run `ipconfig` in PowerShell — use the **vEthernet (WSL)** adapter IP (typically `172.x.x.x`) |
| `DRIVER_OPTS_DEVICE` example | `/tmp` or `/Users/<user>/mzinga-data` | `/tmp` or `/home/<user>/mzinga-data` | `/tmp` or `/home/<user>/mzinga-data` (WSL path) | `C:/Users/<user>/mzinga-data` (forward slashes) |

Full `.env` for local development:

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
MONGO_HOST=<your_ip>
CORS_CONFIGS=*
PAYLOAD_PUBLIC_SERVER_URL=http://localhost:3000
DEBUG_EMAIL_SEND=1
```

> `DEBUG_EMAIL_SEND=1` activates an existing flag in `MailUtils.ts` that logs the email payload to the console without actually calling the SMTP transport. This lets you verify the email flow without a real SendGrid key.

---

### 1.6 Prepare volume directories

The data directories must exist before Docker can bind-mount them.

**macOS, Linux, and Windows running containers inside WSL:**

```sh
rm -rf /tmp/database /tmp/mzinga /tmp/messagebus
mkdir -p /tmp/database /tmp/mzinga /tmp/messagebus
```

If you prefer persistent data that survives reboots, use a directory in your home folder and update `DRIVER_OPTS_DEVICE` accordingly:

```sh
mkdir -p ~/mzinga-data/database ~/mzinga-data/mzinga ~/mzinga-data/messagebus
```

**Windows running containers outside WSL (Docker Desktop with Windows filesystem volumes):**

Run in PowerShell:

```powershell
New-Item -ItemType Directory -Force -Path C:\mzinga-data\database
New-Item -ItemType Directory -Force -Path C:\mzinga-data\mzinga
New-Item -ItemType Directory -Force -Path C:\mzinga-data\messagebus
```

Set `DRIVER_OPTS_DEVICE=C:/mzinga-data` in `.env` (use forward slashes).

> On Windows with Docker Desktop, if you see volume mount errors, ensure the drive is shared: Docker Desktop → Settings → Resources → File Sharing → add the drive or folder.

---

### 1.7 Start the infrastructure

Start only the infrastructure services (MongoDB, RabbitMQ, Redis). MZinga itself will be started separately via `npm run dev`.

**macOS, Linux, WSL:**

```sh
docker compose up database messagebus cache
```

**Windows PowerShell:**

```powershell
docker compose up database messagebus cache
```

Watch the logs. MongoDB is ready when the `database` container reports it is accepting connections. With the simplified file there is no replica set init — it starts faster.

**If you are using `docker-compose-original.yml` and the database container exits with an error containing `Unable to acquire security key` or `Unable to read security file`:**

Switch to the simplified file and wipe the data directories before retrying — a volume initialised with a replica set configuration cannot be reused by a standalone instance:

```sh
# macOS / Linux / WSL
rm -rf /tmp/database /tmp/mzinga /tmp/messagebus
mkdir -p /tmp/database /tmp/mzinga /tmp/messagebus
cp ../../docs/docker-compose-simplified.yml docker-compose.yml
docker compose up database messagebus cache
```

```powershell
# Windows PowerShell
Remove-Item -Recurse -Force C:\mzinga-data
New-Item -ItemType Directory -Force -Path C:\mzinga-data\database
New-Item -ItemType Directory -Force -Path C:\mzinga-data\mzinga
New-Item -ItemType Directory -Force -Path C:\mzinga-data\messagebus
docker compose up database messagebus cache
```

---

### 1.8 Start MZinga

Open a new terminal and run:

```sh
npm run dev
```

Open [http://localhost:3000/admin](http://localhost:3000/admin) and create the first admin user when prompted.

**Windows note:** run this in the same environment where Node.js is installed. If Node.js is installed on Windows (not inside WSL), use PowerShell or Command Prompt. If Node.js is installed inside WSL, use the WSL terminal.

---

### 1.9 Verify the setup

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

- When `COMMUNICATIONS_EXTERNAL_WORKER` is not `"true"`, the original email sending logic runs unchanged (keep all existing lines 37–103 intact inside this branch), then write `status: "sent"` on the document
- When `COMMUNICATIONS_EXTERNAL_WORKER` is `"true"`, the hook instead calls `payload.update` to write `status: "pending"` on the document and returns immediately

Add a guard at the top of the hook: if `doc.status` is already `"pending"` or `"sent"`, return immediately without doing anything. This prevents the `payload.update` call from triggering the hook again in an infinite loop.

> This is the **Branch by Abstraction** pattern: the abstraction boundary is the environment variable. The old path is preserved and reachable by setting `COMMUNICATIONS_EXTERNAL_WORKER=false`. You can roll back instantly.

### 4.3 Verify the flag works

With `COMMUNICATIONS_EXTERNAL_WORKER=true`:
- Create a new Communication document in the admin UI
- The HTTP request should return immediately (no SMTP delay)
- The document should show `status: pending` in the admin UI
- No email log should appear in the terminal

With `COMMUNICATIONS_EXTERNAL_WORKER=false` (or unset):
- The original behaviour is restored: email is logged to the console (because `DEBUG_EMAIL_SEND=1`) and the request blocks until done
- The document should show `status: sent` after saving

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

**macOS, Linux, WSL:**

```sh
cd lab1-worker
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
python worker.py
```

**Windows PowerShell:**

```powershell
cd lab1-worker
python -m venv .venv
.venv\Scripts\Activate.ps1
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

**Previous:** [05b — Infrastructure Reference](05b-infrastructure-reference.md) · **Code snippets:** [06b — Lab 1 Code Snippets](06-lab1-code-snippets.md) · **Next:** [07 — Lab 2 Step by Step](07-lab2-step-by-step.md)
