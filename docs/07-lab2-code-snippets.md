# Lab 2 — Code Snippets

This file is the code companion to [07-lab2-step-by-step.md](07-lab2-step-by-step.md). It contains every snippet referenced in the step-by-step guide, with OS-specific variants where commands differ.

---

## Part A — REST API Worker

### Step A1 — Fix the `update` access rule in `Communications.ts`

In `src/collections/Communications.ts`, change the `update` rule inside the `access` block:

```ts
update: access.GetIsAdmin,
```

### Step A2 — Explore the REST API with curl

**Login and obtain a JWT:**

**macOS / Linux / WSL:**

```sh
curl -s -X POST http://localhost:3000/api/users/login \
  -H "Content-Type: application/json" \
  -d '{"email": "<admin_email>", "password": "<admin_password>"}' \
  | python3 -m json.tool
```

**Windows PowerShell:**

```powershell
curl.exe -s -X POST http://localhost:3000/api/users/login `
  -H "Content-Type: application/json" `
  -d '{"email": "<admin_email>", "password": "<admin_password>"}' `
  | python -m json.tool
```

**Query pending communications with resolved relationships:**

**macOS / Linux / WSL:**

```sh
curl -s "http://localhost:3000/api/communications?where[status][equals]=pending&depth=1" \
  -H "Authorization: Bearer <token>" \
  | python3 -m json.tool
```

**Windows PowerShell:**

```powershell
curl.exe -s "http://localhost:3000/api/communications?where[status][equals]=pending&depth=1" `
  -H "Authorization: Bearer <token>" `
  | python -m json.tool
```

Expected response shape with `depth=1`:

```json
{
  "docs": [
    {
      "id": "...",
      "subject": "Test subject",
      "body": [ ],
      "tos": [
        { "relationTo": "users", "value": { "id": "...", "email": "user@example.com" } }
      ],
      "status": "pending"
    }
  ],
  "totalDocs": 1
}
```

**Update status via PATCH:**

**macOS / Linux / WSL:**

```sh
curl -s -X PATCH http://localhost:3000/api/communications/<id> \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"status": "sent"}'
```

**Windows PowerShell:**

```powershell
curl.exe -s -X PATCH http://localhost:3000/api/communications/<id> `
  -H "Authorization: Bearer <token>" `
  -H "Content-Type: application/json" `
  -d '{"status": "sent"}'
```

---

### Step A3 — REST API worker

**Create the project folder:**

**macOS / Linux / WSL:**

```sh
mkdir -p lab2-worker-rest
cd lab2-worker-rest
```

**Windows PowerShell:**

```powershell
New-Item -ItemType Directory -Force -Path lab2-worker-rest
cd lab2-worker-rest
```

**`requirements.txt`:**

```
requests==2.32.3
python-dotenv==1.0.1
```

**`.env`:**

```sh
MZINGA_URL=http://localhost:3000
MZINGA_EMAIL=<admin_email>
MZINGA_PASSWORD=<admin_password>
POLL_INTERVAL_SECONDS=5
SMTP_HOST=localhost
SMTP_PORT=1025
EMAIL_FROM=worker@mzinga.io
```

**`worker.py`:**

```python
import os
import time
import smtplib
import logging
from email.mime.multipart import MIMEMultipart
from email.mime.text import MIMEText
from dotenv import load_dotenv
import requests

load_dotenv()
logging.basicConfig(level=logging.INFO, format="%(asctime)s %(levelname)s %(message)s")
log = logging.getLogger(__name__)

MZINGA_URL = os.environ["MZINGA_URL"]
MZINGA_EMAIL = os.environ["MZINGA_EMAIL"]
MZINGA_PASSWORD = os.environ["MZINGA_PASSWORD"]
POLL_INTERVAL = int(os.getenv("POLL_INTERVAL_SECONDS", 5))
SMTP_HOST = os.getenv("SMTP_HOST", "localhost")
SMTP_PORT = int(os.getenv("SMTP_PORT", 1025))
EMAIL_FROM = os.getenv("EMAIL_FROM", "worker@mzinga.io")


def login() -> str:
    resp = requests.post(
        f"{MZINGA_URL}/api/users/login",
        json={"email": MZINGA_EMAIL, "password": MZINGA_PASSWORD},
    )
    resp.raise_for_status()
    log.info("Authenticated with MZinga API")
    return resp.json()["token"]


def auth_headers(token: str) -> dict:
    return {"Authorization": f"Bearer {token}"}


def fetch_pending(token: str) -> list:
    resp = requests.get(
        f"{MZINGA_URL}/api/communications",
        params={"where[status][equals]": "pending", "depth": 1},
        headers=auth_headers(token),
    )
    resp.raise_for_status()
    return resp.json().get("docs", [])


def update_status(token: str, doc_id: str, status: str):
    resp = requests.patch(
        f"{MZINGA_URL}/api/communications/{doc_id}",
        json={"status": status},
        headers=auth_headers(token),
    )
    resp.raise_for_status()


def slate_to_html(nodes: list) -> str:
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


def extract_emails(relationship_list: list) -> list[str]:
    emails = []
    for r in relationship_list or []:
        value = r.get("value") or {}
        if isinstance(value, dict) and value.get("email"):
            emails.append(value["email"])
    return emails


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


def process(token: str, doc: dict):
    doc_id = doc["id"]
    log.info(f"Processing communication {doc_id}")
    update_status(token, doc_id, "processing")
    try:
        to_emails = extract_emails(doc.get("tos"))
        if not to_emails:
            raise ValueError("No valid 'to' email addresses found")
        cc_emails = extract_emails(doc.get("ccs"))
        bcc_emails = extract_emails(doc.get("bccs"))
        html = slate_to_html(doc.get("body") or [])
        send_email(to_emails, doc["subject"], html, cc_emails, bcc_emails)
        update_status(token, doc_id, "sent")
        log.info(f"Communication {doc_id} sent successfully")
    except Exception as e:
        log.error(f"Failed to process communication {doc_id}: {e}")
        update_status(token, doc_id, "failed")


def poll():
    token = login()
    log.info(f"Worker started. Polling every {POLL_INTERVAL}s")
    while True:
        try:
            docs = fetch_pending(token)
            for doc in docs:
                process(token, doc)
            if not docs:
                time.sleep(POLL_INTERVAL)
        except requests.HTTPError as e:
            if e.response.status_code == 401:
                log.warning("Token expired, re-authenticating")
                token = login()
            else:
                log.error(f"HTTP error: {e}")
                time.sleep(POLL_INTERVAL)


if __name__ == "__main__":
    poll()
```

**Install and run:**

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

---

## Part B — Event-Driven Worker

### Step B2 — Add env vars to `mzinga-apps/.env`

```sh
RABBITMQ_URL=amqp://guest:guest@localhost:5672/
HOOKSURL_COMMUNICATIONS_AFTERCHANGE=rabbitmq
```

Restart MZinga after saving.

### Step B3 — Run the servicebus-subscriber example to inspect events

**macOS / Linux / WSL:**

```sh
cd mzinga/mzinga-apps/examples/servicebus-subscriber
npm install
RABBITMQ_URL=amqp://guest:guest@localhost:5672/ \
ROUTING_KEY=HOOKSURL_COMMUNICATIONS_AFTERCHANGE \
node index.js
```

**Windows PowerShell:**

```powershell
cd mzinga\mzinga-apps\examples\servicebus-subscriber
npm install
$env:RABBITMQ_URL="amqp://guest:guest@localhost:5672/"
$env:ROUTING_KEY="HOOKSURL_COMMUNICATIONS_AFTERCHANGE"
node index.js
```

Expected event structure published by MZinga:

```json
{
  "type": "HOOKSURL_COMMUNICATIONS_AFTERCHANGE",
  "data": {
    "hook": {
      "envKey": "HOOKSURL_COMMUNICATIONS_AFTERCHANGE",
      "key": "COMMUNICATIONS",
      "type": "afterChange"
    },
    "doc": {
      "id": "...",
      "subject": "...",
      "body": [],
      "tos": [],
      "status": "pending"
    },
    "operation": "create",
    "previousDoc": {}
  }
}
```

---

### Step B4 — Event-driven worker

**Create the project folder:**

**macOS / Linux / WSL:**

```sh
mkdir -p lab2-worker-events
cd lab2-worker-events
```

**Windows PowerShell:**

```powershell
New-Item -ItemType Directory -Force -Path lab2-worker-events
cd lab2-worker-events
```

**`requirements.txt`:**

```
aio-pika==9.5.5
requests==2.32.3
python-dotenv==1.0.1
```

**`.env`:**

```sh
RABBITMQ_URL=amqp://guest:guest@localhost:5672/
ROUTING_KEY=HOOKSURL_COMMUNICATIONS_AFTERCHANGE
EXCHANGE_NAME=mzinga_events_durable
QUEUE_NAME=communications-email-worker
MZINGA_URL=http://localhost:3000
MZINGA_EMAIL=<admin_email>
MZINGA_PASSWORD=<admin_password>
SMTP_HOST=localhost
SMTP_PORT=1025
EMAIL_FROM=worker@mzinga.io
```

**`worker.py`:**

```python
import asyncio
import os
import json
import smtplib
import logging
from email.mime.multipart import MIMEMultipart
from email.mime.text import MIMEText
from dotenv import load_dotenv
import aio_pika
import requests

load_dotenv()
logging.basicConfig(level=logging.INFO, format="%(asctime)s %(levelname)s %(message)s")
log = logging.getLogger(__name__)

RABBITMQ_URL = os.environ["RABBITMQ_URL"]
ROUTING_KEY = os.environ["ROUTING_KEY"]
EXCHANGE_NAME = os.environ["EXCHANGE_NAME"]
QUEUE_NAME = os.environ["QUEUE_NAME"]
MZINGA_URL = os.environ["MZINGA_URL"]
MZINGA_EMAIL = os.environ["MZINGA_EMAIL"]
MZINGA_PASSWORD = os.environ["MZINGA_PASSWORD"]
SMTP_HOST = os.getenv("SMTP_HOST", "localhost")
SMTP_PORT = int(os.getenv("SMTP_PORT", 1025))
EMAIL_FROM = os.getenv("EMAIL_FROM", "worker@mzinga.io")


def login() -> str:
    resp = requests.post(
        f"{MZINGA_URL}/api/users/login",
        json={"email": MZINGA_EMAIL, "password": MZINGA_PASSWORD},
    )
    resp.raise_for_status()
    log.info("Authenticated with MZinga API")
    return resp.json()["token"]


def auth_headers(token: str) -> dict:
    return {"Authorization": f"Bearer {token}"}


def fetch_doc(token: str, doc_id: str) -> dict:
    resp = requests.get(
        f"{MZINGA_URL}/api/communications/{doc_id}",
        params={"depth": 1},
        headers=auth_headers(token),
    )
    resp.raise_for_status()
    return resp.json()


def update_status(token: str, doc_id: str, status: str):
    resp = requests.patch(
        f"{MZINGA_URL}/api/communications/{doc_id}",
        json={"status": status},
        headers=auth_headers(token),
    )
    resp.raise_for_status()


def slate_to_html(nodes: list) -> str:
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


def extract_emails(relationship_list: list) -> list[str]:
    emails = []
    for r in relationship_list or []:
        value = r.get("value") or {}
        if isinstance(value, dict) and value.get("email"):
            emails.append(value["email"])
    return emails


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


def process(token: str, doc: dict) -> str:
    doc_id = doc["id"]

    # Idempotency guard: skip if already processed
    if doc.get("status") in ("sent", "processing"):
        log.info(f"Skipping {doc_id} — already {doc['status']}")
        return token

    log.info(f"Processing communication {doc_id}")
    update_status(token, doc_id, "processing")

    try:
        to_emails = extract_emails(doc.get("tos"))
        if not to_emails:
            raise ValueError("No valid 'to' email addresses found")
        cc_emails = extract_emails(doc.get("ccs"))
        bcc_emails = extract_emails(doc.get("bccs"))
        html = slate_to_html(doc.get("body") or [])
        send_email(to_emails, doc["subject"], html, cc_emails, bcc_emails)
        update_status(token, doc_id, "sent")
        log.info(f"Communication {doc_id} sent successfully")
    except Exception as e:
        log.error(f"Failed to process communication {doc_id}: {e}")
        update_status(token, doc_id, "failed")

    return token


async def main():
    token = login()

    connection = await aio_pika.connect_robust(RABBITMQ_URL)
    async with connection:
        channel = await connection.channel()
        await channel.set_qos(prefetch_count=1)

        exchange = await channel.declare_exchange(
            EXCHANGE_NAME, aio_pika.ExchangeType.TOPIC,
            durable=True, internal=True, auto_delete=False,
        )

        queue = await channel.declare_queue(QUEUE_NAME, durable=True)
        await queue.bind(exchange, routing_key=ROUTING_KEY)

        log.info(f"Subscribed to {EXCHANGE_NAME} with key {ROUTING_KEY}. Waiting for messages.")

        async with queue.iterator() as messages:
            async for message in messages:
                async with message.process(requeue_on_timeout=True):
                    try:
                        body = json.loads(message.body.decode())
                        event_data = body.get("data", {})
                        operation = event_data.get("operation")
                        doc_id = (event_data.get("doc") or {}).get("id")

                        if not doc_id:
                            log.warning("Message missing doc.id, skipping")
                            continue

                        # Filter out update operations to avoid infinite loop:
                        # the worker's own PATCH status write-back triggers another
                        # afterChange event with operation="update"
                        if operation != "create":
                            log.debug(f"Ignoring operation={operation} for {doc_id}")
                            continue

                        doc = fetch_doc(token, doc_id)
                        token = process(token, doc)

                    except requests.HTTPError as e:
                        if e.response.status_code == 401:
                            log.warning("Token expired, re-authenticating")
                            token = login()
                        else:
                            log.error(f"HTTP error processing message: {e}")
                            raise


if __name__ == "__main__":
    asyncio.run(main())
```

**Install and run:**

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

---

**Previous:** [06 — Lab 1 Code Snippets](06-lab1-code-snippets.md) · **Step-by-step guide:** [07 — Lab 2 Step by Step](07-lab2-step-by-step.md)
