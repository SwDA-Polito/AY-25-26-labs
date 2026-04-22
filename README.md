# AY-25-26-labs
Laboratories

## Documentation for Lab1 and Lab2

Read in order — each document builds on the previous one.

| # | Document | Contents |
|---|---|---|
| 1 | [Laboratory Introduction](docs/01-laboratory-introduction.md) | What MZinga is, why a real system matters, and the four-state architecture journey |
| 2 | [Architecture Evolution: Four States from Monolith to Event-Driven](docs/02-architecture-evolution.md) | Pattern-by-pattern walkthrough of each architectural state with code references |
| 3 | [Communications Email Flow & Decoupling Guide](docs/03-communications-email-flow.md) | Line-by-line walkthrough of the current email flow and the specific code changes to decouple it |
| 4 | [The Strangler Fig Pattern](docs/04-strangler-fig-pattern.md) | Deep dive into the primary migration pattern: origin, mechanics, and limitations |
| 5 | [Supporting Patterns Catalogue](docs/05-supporting-patterns-catalogue.md) | Full catalogue of patterns relevant across all four states |
| 5b | [Infrastructure Reference: MongoDB and RabbitMQ](docs/05b-infrastructure-reference.md) | MongoDB standalone vs replica set, RabbitMQ exchanges, queues, vhosts, and auth |
| 6 | [Lab 1 Step by Step](docs/06-lab1-step-by-step.md) | DB-coupled Python worker, feature flag, status field, end-to-end verification |
| 6b | [Lab 1 Code Snippets](docs/06-lab1-code-snippets.md) | All code snippets for Lab 1 with macOS, Linux, and Windows variants |
| 7 | [Lab 2 Step by Step](docs/07-lab2-step-by-step.md) | REST API worker (core) + event-driven RabbitMQ worker (optional extension) |
| 7b | [Lab 2 Code Snippets](docs/07-lab2-code-snippets.md) | All code snippets for Lab 2 with macOS, Linux, and Windows variants |