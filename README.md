# Marketplace Fulfillment Service

A mock backend simulating a retailer's **External Marketplace Services** team: a fulfillment integration for a fake "TikTok Shop" partner.

> **All data is fake.** No real retailer systems, no real vendor credentials, no real TikTok Shop API. Every SKU is prefixed `MOCK-`, every customer is fake, and every "external" call is stubbed inside this repo.

**Backend only.** Java + Spring Boot. Tested via curl, Postman and the terminal. No UI.

---

## Status

| Item | State |
|---|---|
| Backlog | Ready. See [`BACKLOG.md`](./BACKLOG.md) |
| Decision log | Started. See [`DECISIONS.md`](./DECISIONS.md) |
| Implementation | Not started (begin at FUL-001) |

Sections marked _(finalized in FUL-035)_ are filled in as the build progresses. Do not leave them stale.

---

## What this system does

1. A marketplace (mock TikTok Shop) checks whether a SKU can be fulfilled.
2. The marketplace submits an order in **its own JSON schema**.
3. The service translates it into an **internal order model**, guarantees it is never created twice, and persists it.
4. An event is published, and orchestration moves the order through a fulfillment lifecycle: validate stock, allocate inventory, ship (mocked), confirm.
5. Every step is observable: logs traceable by order ID, plus metrics.

## Why this project exists

It exercises the problems a real marketplace-integration team deals with: duplicate webhooks, partner-specific schemas, state machines, async messaging, race conditions on inventory, and flaky dependencies. The goal is engineering judgment, not feature count.

---

## Tech stack

| Layer | Choice |
|---|---|
| Language / framework | Java 17+, Spring Boot (Web, Data JPA, Validation, Actuator) |
| Database | PostgreSQL (Docker Compose), schema managed by Flyway |
| Messaging | Kafka (default) or RabbitMQ. See decision D-003 |
| Resilience | Resilience4j (retry, circuit breaker, time limiter) |
| Observability | Micrometer + Actuator, structured logs with MDC correlation ID |
| Testing | JUnit 5, MockMvc, Testcontainers, WireMock |
| API docs | springdoc OpenAPI (Swagger UI) + Postman collection |
| CI | GitHub Actions |

## Architecture

**Modular monolith.** One deployable Spring Boot app with clean package boundaries, not microservices. Modules only talk through defined interfaces or events, so they could be split later.

```
 Mock TikTok Shop (curl / Postman)
        │  GET  /availability/{sku}
        │  POST /orders/ingest   (marketplace-shaped JSON)
        ▼
┌───────────────────────┐
│ ingestion + adapter   │  validate → adapter.translateOrder() → idempotent save
└──────────┬────────────┘  (order row + outbox row in ONE transaction)
           │ outbox publisher
           ▼
     ┌───────────┐
     │  Broker   │──► dead-letter topic
     └─────┬─────┘
           ▼
┌───────────────────────┐      ┌────────────────────┐      ┌───────────────────┐
│ orchestration         │────► │ availability       │      │ mock carrier      │
│ state machine + steps │      │ inventory + locking│      │ (WireMock/stub)   │
└──────────┬────────────┘ ───────────────────────────────►  │ retry/CB/timeout  │
           ▼                                                └───────────────────┘
   PostgreSQL  +  /actuator/metrics  +  logs greppable by orderId
```

### Package layout

```
com.example.fulfillment
├── availability     # inventory entity, repo, service, controller
├── ingestion        # order ingest controller, DTOs, idempotency
├── orchestration    # state machine, step handlers, event consumers
├── adapter          # MarketplaceAdapter + TikTok/Generic implementations
├── messaging        # outbox, publisher, consumer config, DLQ
├── carrier          # client for the stubbed carrier/warehouse API
└── common           # error handling, logging/MDC, shared config
```

---

## Order state machine

| From | Allowed next states |
|---|---|
| `RECEIVED` | `VALIDATED`, `FAILED`, `CANCELLED` |
| `VALIDATED` | `ALLOCATED`, `FAILED`, `CANCELLED` |
| `ALLOCATED` | `SHIPPED`, `FAILED`, `CANCELLED` (releases inventory) |
| `SHIPPED` | `CONFIRMED` |
| `CONFIRMED` / `FAILED` / `CANCELLED` | none (terminal) |

Any other transition is rejected with `409 INVALID_TRANSITION`.

---

## API summary _(finalized in FUL-035)_

| Method | Path | Purpose |
|---|---|---|
| GET | `/availability/{sku}` | Stock, fulfillment source, ETA. 404 if unknown |
| POST | `/availability/bulk-check` | Batch availability, mixed known/unknown SKUs |
| POST | `/orders/ingest` | Submit a marketplace-shaped order (idempotent) |
| GET | `/orders/{id}` | Order details and current status |
| POST | `/orders/{id}/advance` | Debug/fallback manual transition |
| GET | `/actuator/health` | Health check |
| GET | `/actuator/metrics` | Operational metrics |

Errors use RFC 7807 `ProblemDetail`.

---

## Running locally _(finalized in FUL-035)_

Prerequisites: JDK 17+, Docker, Maven or Gradle wrapper (included), Postman (optional).

```bash
# 1. Start dependencies (Postgres, broker)
docker compose up -d

# 2. Run the app
./mvnw spring-boot:run        # or ./gradlew bootRun

# 3. Verify
curl localhost:8080/actuator/health

# 4. Run tests (Testcontainers needs Docker running)
./mvnw verify                 # or ./gradlew check
```

Import `/postman/fulfillment-service.postman_collection.json` into Postman and run the folders in order.

---

## Design reasoning _(finalized in FUL-035)_

Each section below is written from the decision notes in [`DECISIONS.md`](./DECISIONS.md).

- **Why idempotency:** marketplaces retry webhooks on timeout. A unique DB constraint, not an "exists then insert" check, guarantees one order per key even under concurrent retries.
- **Why async / event-driven:** decouples ingestion latency from fulfillment work. The transactional outbox avoids the dual-write problem.
- **Why an adapter layer:** isolates partner-specific schemas so new marketplaces don't touch core logic.
- **Why safe allocation:** two orders competing for the last unit must never both succeed. Optimistic locking plus a concurrency test.
- **How resilience is handled:** retry, circuit breaker and timeout on the carrier call, with fallbacks.
- **Why a modular monolith:** right size for the problem. Boundaries are enforced by packages, not network calls.

---

## Repo docs

| File | Purpose |
|---|---|
| `README.md` | This file. Overview, run instructions, design reasoning |
| `BACKLOG.md` | Epics, tickets, acceptance criteria, working rules |
| `DECISIONS.md` | Lightweight decision log (ADR style) |
| `/mocks` | Example marketplace payloads |
| `/postman` | Exported Postman collection (updated per ticket) |

## Stretch goals (post-backlog)

Rate limiting on ingestion, CLI tool to walk an order through its lifecycle, order cancellation flow with inventory release, per-state time metrics.
