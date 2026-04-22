# Infrastructure Reference: MongoDB and RabbitMQ

This document explains the infrastructure components used in these labs — what they are, how they are configured here, and why certain choices are appropriate for a lab environment but not for production.

---

## MongoDB

### What it is

MongoDB is a document-oriented database. Instead of storing data in rows and columns like a relational database, it stores data as JSON-like documents grouped into collections. Each document can have a different shape — there is no enforced schema at the database level unless you add one explicitly.

MZinga uses MongoDB as its primary data store. Every collection defined in `src/collections/` — `Communications`, `Users`, `Media`, and so on — maps directly to a MongoDB collection. Payload CMS generates the queries, indexes, and schema validation automatically from the TypeScript collection definitions.

---

### Standalone vs replica set

MongoDB can run in two modes relevant to this lab.

**Standalone** — a single MongoDB process with no replication. Simple to start, no key files or replica set configuration required. Reads and writes go to one node. This is what `docker-compose-simplified.yml` runs.

Limitations of standalone:
- No high availability — if the process crashes, the database is unavailable until it restarts
- No change streams — MongoDB change streams (which allow applications to subscribe to real-time document changes) require a replica set or sharded cluster
- No multi-document transactions across collections

**Replica set** — a group of MongoDB processes (called members) that maintain the same data. One member is the primary and accepts all writes; the others are secondaries that replicate from the primary. If the primary fails, the secondaries elect a new primary automatically.

In this lab the replica set has a single member (`rs0` with one node). This is not high availability — a single-member replica set has no failover — but it unlocks the features that require replica set mode, specifically change streams and transactions, which Payload CMS uses internally.

The `docker-compose-original.yml` runs MongoDB in replica set mode. It requires a key file (`custom.replica.key`) that all members use to authenticate with each other. On some OS and Docker configurations the file permissions required by MongoDB (`chmod 400`, owned by uid 999) are difficult to set correctly from a Docker init container, which is why the simplified file exists as a fallback.

**Which to use in this lab:** start with `docker-compose-simplified.yml`. The lab exercises do not require change streams or transactions. Use the original only if you have a specific reason.

---

### Connection string anatomy

The connection string used in this lab:

```
mongodb://admin:admin@localhost:27017/mzinga?authSource=admin&directConnection=true
```

| Part | Meaning |
|---|---|
| `mongodb://` | Protocol |
| `admin:admin` | Username and password |
| `localhost:27017` | Host and port |
| `/mzinga` | Database name to use by default |
| `authSource=admin` | The database where the user credentials are stored (the `admin` database) |
| `directConnection=true` | Connect directly to this node, do not attempt replica set discovery |

`directConnection=true` is required when connecting to a single-node replica set from outside Docker. Without it the driver tries to discover all replica set members using the hostnames they advertise internally, which are Docker container names not reachable from the host machine.

---

## RabbitMQ

### What it is

RabbitMQ is a message broker. It receives messages from producers, stores them temporarily, and delivers them to consumers. It decouples the sender from the receiver — the producer does not need to know who will consume the message, or whether the consumer is currently running.

In this lab RabbitMQ is used in Lab 2 Part B to replace the polling loop. MZinga publishes an event when a `Communications` document is saved; the Python worker subscribes and reacts immediately.

---

### Publish/Subscribe

The Publish/Subscribe pattern (Pub/Sub) is a messaging model where:

- **Publishers** send messages to a named channel without knowing who will receive them
- **Subscribers** declare interest in a channel and receive all messages sent to it
- The broker (RabbitMQ) sits in the middle and routes messages from publishers to subscribers

This is different from a direct queue where one producer sends to one consumer. In Pub/Sub, one published message can be delivered to multiple independent subscribers simultaneously.

---

### Exchanges

In RabbitMQ, producers do not send messages directly to queues. They send messages to an **exchange**. The exchange then routes the message to one or more queues based on rules called bindings.

There are several exchange types. This lab uses **topic exchanges**:

- Each message has a **routing key** — a dot-separated string (e.g. `HOOKSURL_COMMUNICATIONS_AFTERCHANGE`)
- Each queue binding specifies a **pattern** to match against routing keys
- `#` matches zero or more words; `*` matches exactly one word
- A queue bound with `#` receives every message regardless of routing key

MZinga declares two exchanges:

| Exchange | Type | Durable | Purpose |
|---|---|---|---|
| `mzinga_events` | topic | no | Transient — messages are lost if RabbitMQ restarts |
| `mzinga_events_durable` | topic | yes | Persistent — messages survive RabbitMQ restarts |

The `mzinga_events_durable` exchange is bound to receive all messages from `mzinga_events` via routing key `#`. MZinga publishes to `mzinga_events`; the durable exchange automatically receives a copy of every event. Workers should subscribe to `mzinga_events_durable` so messages are not lost if the worker is temporarily down.

---

### Queues

A **queue** is where messages wait until a consumer picks them up. Key properties:

**Durable** — a durable queue survives a RabbitMQ restart. Its definition is persisted to disk. Messages in a durable queue are also persisted if they were published as persistent. In this lab the worker declares a durable queue named `communications-email-worker` so that messages queued while the worker is down are not lost.

**Exclusive** — an exclusive queue is tied to the connection that created it and is deleted when that connection closes. The `servicebus-subscriber` example in the MZinga repo uses an exclusive queue — it is a temporary listener for inspection only, not a durable consumer.

**Auto-delete** — the queue is deleted when the last consumer disconnects.

**Prefetch count** — controls how many unacknowledged messages RabbitMQ delivers to a consumer at once. Setting `prefetch_count=1` means RabbitMQ delivers one message, waits for acknowledgement, then delivers the next. This is important when running multiple worker instances: it ensures each message goes to exactly one worker and is not processed twice.

---

### Message acknowledgement

When a consumer receives a message it must **acknowledge** it to tell RabbitMQ the message was processed successfully. Until acknowledged, RabbitMQ considers the message undelivered and will redeliver it if the consumer disconnects.

If the worker crashes after receiving a message but before acknowledging it, RabbitMQ redelivers the message to the next available consumer. This guarantees at-least-once delivery — the message will be processed, but possibly more than once if the worker crashes at the wrong moment. This is why the worker includes an idempotency guard that checks `status` before processing.

---

### Virtual hosts

A RabbitMQ **virtual host** (vhost) is a logical partition inside a single RabbitMQ instance. Each vhost has its own exchanges, queues, bindings, and permissions. Vhosts are completely isolated from each other — a queue in `/vhost-a` is invisible to a connection authenticated to `/vhost-b`.

Vhosts are used to:
- Separate environments (development, staging, production) on the same broker
- Separate tenants in a multi-tenant system
- Apply different permission policies to different groups of users

The default vhost is `/`. In this lab all connections use the default vhost, which is the RabbitMQ default when no vhost is specified in the connection string.

---

### Users and permissions

RabbitMQ has its own user management system, separate from the operating system. Each user has a set of permissions scoped to a vhost: configure (create/delete exchanges and queues), write (publish messages), and read (consume messages).

**The `guest` user** is a built-in default user created when RabbitMQ is first installed. It has full administrator permissions on the default vhost `/`.

By default, RabbitMQ restricts the `guest` user to connections from `localhost` only. However, when running RabbitMQ inside Docker and connecting from the host machine or from another container, the connection does not originate from `localhost` as seen by RabbitMQ. The Docker Compose configuration in this lab sets `RABBITMQ_DEFAULT_USER=guest` and `RABBITMQ_DEFAULT_PASS=guest` explicitly, which recreates the guest user without the localhost restriction.

**Why `guest:guest` is only acceptable in a lab:**

- The password is publicly known — it is the default for every RabbitMQ installation worldwide
- There is no TLS — credentials are transmitted in plaintext over the connection
- The user has full administrative access — it can create, delete, and inspect any exchange, queue, or binding
- The management UI at port 15672 is exposed with the same credentials, giving anyone on the network full visibility into the broker

In a production environment you would:
- Create a dedicated user with a strong password and the minimum permissions required
- Disable or restrict the `guest` user
- Enable TLS on the AMQP port (5671) and the management UI (15671)
- Use separate vhosts per environment or tenant
- Rotate credentials and store them in a secrets manager, not in `.env` files

---

**Previous:** [05 — Supporting Patterns Catalogue](05-supporting-patterns-catalogue.md) · **Next:** [06 — Lab 1 Step by Step](06-lab1-step-by-step.md)
