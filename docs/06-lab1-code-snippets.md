# Lab 1 — Code Snippets

This file is the code companion to [06-lab1-step-by-step.md](06-lab1-step-by-step.md). It contains every snippet referenced in the step-by-step guide, with OS-specific variants where the commands differ between macOS, Linux, and Windows.

---

## Step 1 — Clone and set up MZinga locally

### 1.1 — Remove `mzinga/` from `.gitignore`

Open `.gitignore` at the root of your forked repo and delete this line:

```
mzinga/
```

Commit the change:

```sh
git add .gitignore
git commit -m "chore: allow mzinga folder to be tracked in personal fork"
```

### 1.2 — Clone MZinga

```sh
cd mzinga
git clone https://github.com/mzinga-io/mzinga-apps.git
cd mzinga-apps
```

### 1.3 — Install dependencies

```sh
npm install
```

### 1.4 — Copy the simplified docker-compose file

**macOS / Linux / WSL:**

```sh
cp ../../docs/docker-compose-simplified.yml docker-compose.yml
```

**Windows PowerShell:**

```powershell
Copy-Item ..\..\docs\docker-compose-simplified.yml docker-compose.yml
```

### 1.5 — Configure `.env`

**macOS / Linux / WSL:**

```sh
cp .env.template .env
```

**Windows PowerShell:**

```powershell
Copy-Item .env.template .env
```

Find your `MONGO_HOST` value:

**macOS:**

```sh
ifconfig | grep "inet " | grep -v 127.0.0.1
```

**Linux:**

```sh
ip addr | grep "inet " | grep -v 127.0.0.1
```

**Windows PowerShell (look for the vEthernet WSL adapter):**

```powershell
ipconfig
```

Full `.env` content:

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

For Windows containers outside WSL, use a Windows path:

```sh
DRIVER_OPTS_DEVICE=C:/Users/<user>/mzinga-data
```

### 1.6 — Prepare volume directories

**macOS / Linux / WSL:**

```sh
rm -rf /tmp/database /tmp/mzinga /tmp/messagebus
mkdir -p /tmp/database /tmp/mzinga /tmp/messagebus
```

For persistent storage across reboots:

```sh
mkdir -p ~/mzinga-data/database ~/mzinga-data/mzinga ~/mzinga-data/messagebus
```

**Windows PowerShell:**

```powershell
Remove-Item -Recurse -Force C:\mzinga-data -ErrorAction SilentlyContinue
New-Item -ItemType Directory -Force -Path C:\mzinga-data\database
New-Item -ItemType Directory -Force -Path C:\mzinga-data\mzinga
New-Item -ItemType Directory -Force -Path C:\mzinga-data\messagebus
```

### 1.7 — Start the infrastructure

**macOS / Linux / WSL / Windows PowerShell:**

```sh
docker compose up database messagebus cache
```

If the database fails with `Unable to acquire security key` or `Unable to read security file`, wipe the volumes and retry with the simplified file:

**macOS / Linux / WSL:**

```sh
rm -rf /tmp/database /tmp/mzinga /tmp/messagebus
mkdir -p /tmp/database /tmp/mzinga /tmp/messagebus
cp ../../docs/docker-compose-simplified.yml docker-compose.yml
docker compose up database messagebus cache
```

**Windows PowerShell:**

```powershell
Remove-Item -Recurse -Force C:\mzinga-data
New-Item -ItemType Directory -Force -Path C:\mzinga-data\database
New-Item -ItemType Directory -Force -Path C:\mzinga-data\mzinga
New-Item -ItemType Directory -Force -Path C:\mzinga-data\messagebus
Copy-Item ..\..\docs\docker-compose-simplified.yml docker-compose.yml
docker compose up database messagebus cache
```

### 1.8 — Start MZinga

**macOS / Linux / WSL:**

```sh
npm run dev
```

**Windows PowerShell (if Node.js is installed on Windows, not in WSL):**

```powershell
npm run dev
```

### 1.9 — Start MailHog (local SMTP for testing)

**macOS / Linux / WSL / Windows PowerShell:**

```sh
docker run -d -p 1025:1025 -p 8025:8025 mailhog/mailhog
```

Sent emails appear at `http://localhost:8025`.

---

## Step 2 — Understand the current email flow

### 2.3 — Inspect MongoDB documents

Connect to MongoDB:

```sh
mongosh "mongodb://admin:admin@localhost:27017/mzinga?authSource=admin&directConnection=true"
```

Inspect a Communications document:

```js
use mzinga
db.communications.findOne()
```

Inspect a User document:

```js
db.users.findOne({}, { email: 1 })
```

---

## Step 3 — Add a `status` field to Communications

### 3.2 — `status` field definition in `Communications.ts`

Add to the `fields` array in `src/collections/Communications.ts`:

```ts
{
  name: "status",
  type: "select",
  options: [
    { label: "Pending", value: "pending" },
    { label: "Processing", value: "processing" },
    { label: "Sent", value: "sent" },
    { label: "Failed", value: "failed" },
  ],
  admin: {
    readOnly: true,
    position: "sidebar",
  },
},
```

Update `defaultColumns` in the `admin` block:

```ts
defaultColumns: ["subject", "tos", "status"],
```

Restart MZinga after saving:

```sh
npm run dev
```

---

## Step 4 — Disable in-process email sending via a feature flag

### 4.1 — Add the flag to `.env`

```sh
COMMUNICATIONS_EXTERNAL_WORKER=true
```

### 4.2 — Modified `afterChange` hook in `Communications.ts`

The full updated hook body. The guard at the top prevents infinite loops caused by `payload.update` re-triggering `afterChange`:

```ts
afterChange: [
  async ({ doc }) => {
    const { tos, ccs, bccs, subject, body } = doc;

    // Guard: skip if status was already written by this hook or the worker
    if (doc.status === "pending" || doc.status === "sent") {
      return doc;
    }

    if (process.env.COMMUNICATIONS_EXTERNAL_WORKER === "true") {
      await payload.update({
        collection: Slugs.Communications,
        id: doc.id,
        data: { status: "pending" },
      });
      return doc;
    }

    // Original in-process email sending path
    for (const part of body) {
      if (part.type !== "upload") { continue; }
      const relationToSlug = part.relationTo;
      const uploadDoc = await payload.findByID({
        collection: relationToSlug,
        id: part.value.id,
      });
      part.value = { ...part.value, ...uploadDoc };
    }
    const html = TextUtils.Serialize(body || "");
    try {
      const users = await payload.find({
        collection: tos[0].relationTo,
        where: { id: { in: tos.map((to) => to.value.id || to.value).join(",") } },
      });
      const usersEmails = users.docs.map((u) => u.email);
      if (!usersEmails.length) {
        throw new Error("No valid email addresses found for 'tos' users.");
      }
      let cc;
      if (ccs) {
        const copiedUsers = await payload.find({
          collection: ccs[0].relationTo,
          where: { id: { in: ccs.map((cc) => cc.value.id).join(",") } },
        });
        cc = copiedUsers.docs.map((u) => u.email).join(",");
      }
      let bcc;
      if (bccs) {
        const blindCopiedUsers = await payload.find({
          collection: bccs[0].relationTo,
          where: { id: { in: bccs.map((bcc) => bcc.value.id).join(",") } },
        });
        bcc = blindCopiedUsers.docs.map((u) => u.email).join(",");
      }
      const promises = [];
      for (const to of usersEmails) {
        const message = {
          from: payload.emailOptions.fromAddress,
          subject, to, cc, bcc, html,
        };
        promises.push(
          MailUtils.sendMail(payload, message).catch((e) => {
            MZingaLogger.Instance?.error(`[Communications:err] ${e}`);
            return null;
          }),
        );
      }
      await Promise.all(promises.filter((p) => Boolean(p)));
      await payload.update({
        collection: Slugs.Communications,
        id: doc.id,
        data: { status: "sent" },
      });
      return doc;
    } catch (err) {
      if (err.response?.body?.errors) {
        err.response.body.errors.forEach((error) =>
          MZingaLogger.Instance?.error(
            `[Communications:err] ${error.field} ${error.message}`,
          ),
        );
      } else {
        MZingaLogger.Instance?.error(`[Communications:err] ${err}`);
      }
      throw err;
    }
  },
],
```

---

## Step 5 — Build the Python worker

### 5.1 — Create the project folder

**macOS / Linux / WSL:**

```sh
mkdir -p lab1-worker
cd lab1-worker
```

**Windows PowerShell:**

```powershell
New-Item -ItemType Directory -Force -Path lab1-worker
cd lab1-worker
```

### 5.2 — `requirements.txt`

```
pymongo==4.10.1
python-dotenv==1.0.1
```

### 5.3 — `.env`

```sh
MONGODB_URI=mongodb://admin:admin@localhost:27017/mzinga?authSource=admin&directConnection=true
POLL_INTERVAL_SECONDS=5
SMTP_HOST=localhost
SMTP_PORT=1025
EMAIL_FROM=worker@mzinga.io
```

### 5.4 — `worker.py`

```python
import os
import time
import smtplib
import logging
from email.mime.multipart import MIMEMultipart
from email.mime.text import MIMEText
from dotenv import load_dotenv
from pymongo import MongoClient
from bson import ObjectId

load_dotenv()
logging.basicConfig(level=logging.INFO, format="%(asctime)s %(levelname)s %(message)s")
log = logging.getLogger(__name__)

client = MongoClient(os.environ["MONGODB_URI"])
db = client.get_default_database()

POLL_INTERVAL = int(os.getenv("POLL_INTERVAL_SECONDS", 5))
SMTP_HOST = os.getenv("SMTP_HOST", "localhost")
SMTP_PORT = int(os.getenv("SMTP_PORT", 1025))
EMAIL_FROM = os.getenv("EMAIL_FROM", "worker@mzinga.io")


def slate_to_html(nodes: list) -> str:
    """Minimal Slate AST → HTML serialiser."""
    html = ""
    for node in nodes or []:
        if node.get("type") == "paragraph":
            html += f"<p>{slate_to_html(node.get('children', []))}</p>"
        elif node.get("type") == "h1":
            html += f"<h1>{slate_to_html(node.get('children', []))}</h1>"
        elif node.get("type") == "h2":
            html += f"<h2>{slate_to_html(node.get('children', []))}</h2>"
        elif node.get("type") == "ul":
            html += f"<ul>{slate_to_html(node.get('children', []))}</ul>"
        elif node.get("type") == "li":
            html += f"<li>{slate_to_html(node.get('children', []))}</li>"
        elif node.get("type") == "link":
            url = node.get("url", "#")
            html += f'<a href="{url}">{slate_to_html(node.get("children", []))}</a>'
        elif "text" in node:
            text = node["text"]
            if node.get("bold"):
                text = f"<strong>{text}</strong>"
            if node.get("italic"):
                text = f"<em>{text}</em>"
            html += text
        else:
            html += slate_to_html(node.get("children", []))
    return html


def resolve_emails(relationship_list: list) -> list[str]:
    """Resolve Payload relationship references to email addresses."""
    if not relationship_list:
        return []
    ids = [ObjectId(r["value"]) for r in relationship_list if r.get("value")]
    users = db.users.find({"_id": {"$in": ids}}, {"email": 1})
    return [u["email"] for u in users if u.get("email")]


def send_email(to_addresses: list[str], subject: str, html: str,
               cc_addresses: list[str] = None, bcc_addresses: list[str] = None):
    msg = MIMEMultipart("alternative")
    msg["Subject"] = subject
    msg["From"] = EMAIL_FROM
    msg["To"] = ", ".join(to_addresses)
    if cc_addresses:
        msg["Cc"] = ", ".join(cc_addresses)
    msg.attach(MIMEText(html, "html"))
    all_recipients = to_addresses + (cc_addresses or []) + (bcc_addresses or [])
    with smtplib.SMTP(SMTP_HOST, SMTP_PORT) as server:
        server.sendmail(EMAIL_FROM, all_recipients, msg.as_string())


def process(doc: dict):
    doc_id = doc["_id"]
    log.info(f"Processing communication {doc_id}")
    db.communications.update_one({"_id": doc_id}, {"$set": {"status": "processing"}})
    try:
        to_emails = resolve_emails(doc.get("tos") or [])
        if not to_emails:
            raise ValueError("No valid 'to' email addresses found")
        cc_emails = resolve_emails(doc.get("ccs") or [])
        bcc_emails = resolve_emails(doc.get("bccs") or [])
        html = slate_to_html(doc.get("body") or [])
        send_email(to_emails, doc["subject"], html, cc_emails, bcc_emails)
        db.communications.update_one({"_id": doc_id}, {"$set": {"status": "sent"}})
        log.info(f"Communication {doc_id} sent successfully")
    except Exception as e:
        log.error(f"Failed to process communication {doc_id}: {e}")
        db.communications.update_one({"_id": doc_id}, {"$set": {"status": "failed"}})


def poll():
    log.info(f"Worker started. Polling every {POLL_INTERVAL}s")
    while True:
        doc = db.communications.find_one({"status": "pending"})
        if doc:
            process(doc)
        else:
            time.sleep(POLL_INTERVAL)


if __name__ == "__main__":
    poll()
```

### 5.5 — Create virtualenv, install dependencies, and run

**macOS / Linux / WSL:**

```sh
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
python worker.py
```

**Windows PowerShell:**

```powershell
python -m venv .venv
.venv\Scripts\Activate.ps1
pip install -r requirements.txt
python worker.py
```

> If `Activate.ps1` is blocked by the execution policy, run first:
> ```powershell
> Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
> ```

---

## Step 6 — End-to-end verification

### Reset a document to `pending` for re-testing

If you need to reprocess a document that is already `sent` or `failed`, reset it directly in MongoDB:

```js
db.communications.updateOne(
  { subject: "Test subject" },
  { $set: { status: "pending" } }
)
```

Or reset all failed documents at once:

```js
db.communications.updateMany(
  { status: "failed" },
  { $set: { status: "pending" } }
)
```

---

**Previous:** [06 — Lab 1 Step by Step](06-lab1-step-by-step.md) · **Next:** [07 — Lab 2 Step by Step](07-lab2-step-by-step.md)
