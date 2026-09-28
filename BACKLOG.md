# Marketplace Fulfillment Service: Backlog

Mock backend simulating a retailer's External Marketplace Services team, integrating a fake "TikTok Shop" partner. Backend only. Java + Spring Boot. Tested via curl / Postman / terminal / automated tests.

**All data is fake.** No real systems, credentials or partner APIs. See `README.md` for architecture and `DECISIONS.md` for the decision log.

---

## Working rules

- Work **one ticket at a time, top to bottom**. Don't skip ahead.
- A ticket is done only when its **Acceptance Criteria (AC)**, its **Definition of Done (DoD)** and the **Global DoD** below are all met.
- No dates or estimates. Pick up the next ticket only when the current one is closed.
- If a ticket is too big mid-flight, split it explicitly (e.g. FUL-018a / FUL-018b) before moving on. Don't abandon.
- Newly discovered work goes at the bottom of the relevant epic as a new ticket. Don't silently skip it.
- Each ticket lists **Depends on** so the order is never a mystery.

## Global Definition of Done (applies to every ticket)

- [ ] Automated tests written **with** the ticket (unit and/or MockMvc/Testcontainers as appropriate) and passing locally
- [ ] CI is green on the branch
- [ ] New/changed endpoints have a request in the Postman collection (with example response)
- [ ] Schema changes are Flyway migrations, never `ddl-auto`
- [ ] If a non-obvious choice was made, a short entry is added to `DECISIONS.md`
- [ ] Code merged to `main` via PR (self-review counts for a solo project)

## Ticket ID map (vs. original backlog)

Renumbered because new tickets were added. Original FUL-001..030 map roughly in order; merged/new tickets are marked **[NEW]** or **[MERGED]**.

---

## Epic 0: Project Setup & Foundations

- [ ] ### FUL-001: Initialize Spring Boot project
  - **Description:** As the dev, I need a working skeleton so all modules have a home.
  - **AC:**
    - Java 17+, Maven or Gradle, starters: Web, Data JPA, Validation, Actuator
    - `GET /actuator/health` returns 200
    - Package structure per README: `availability`, `ingestion`, `orchestration`, `adapter`, `messaging`, `carrier`, `common`
    - `.gitignore`, wrapper committed
  - **DoD:** Runs via terminal command; health verified with curl; structure committed.
  - **Depends on:** none

- [ ] ### FUL-002: Postgres via Docker Compose + Flyway **[CHANGED]**
  - **Description:** Real database from day one; H2 hides production behavior. Schema managed by migrations.
  - **AC:**
    - `docker-compose.yml` runs Postgres; app connects on startup
    - Flyway enabled, `V1__baseline.sql` creates a dummy table; `spring.jpa.hibernate.ddl-auto=validate`
    - Dummy entity + repository round-trip verified by a **Testcontainers** integration test
    - Reusable Testcontainers base test class created
  - **DoD:** `docker compose up -d` + app start is clean; Testcontainers test passes.
  - **Depends on:** FUL-001

- [ ] ### FUL-003: CI pipeline **[NEW]**
  - **Description:** Tests only matter if they run automatically.
  - **AC:** GitHub Actions workflow builds and runs all tests on push/PR; badge in README.
  - **DoD:** A deliberately failing test turns CI red; reverting turns it green.
  - **Depends on:** FUL-002

- [ ] ### FUL-004: API conventions: error handling + OpenAPI **[NEW]**
  - **Description:** Consistent error shape and self-documenting API from the start.
  - **AC:**
    - `@RestControllerAdvice` returns RFC 7807 `ProblemDetail` for validation (400), not found (404), invalid transition (409) and unexpected errors (500, no stack trace leaked)
    - springdoc OpenAPI available at `/swagger-ui.html`
    - Rule documented: controllers use **DTOs**, never expose JPA entities
  - **DoD:** MockMvc tests assert error bodies for at least 400 and 404.
  - **Depends on:** FUL-001

- [ ] ### FUL-005: Seed mock product & inventory data
  - **Description:** As the system, I need fake product/inventory data since there's no real source.
  - **AC:**
    - 10 to 20 mock products (SKU, name, quantity, store/DC location) seeded via Flyway or a dev-profile seeder
    - All SKUs prefixed `MOCK-`; at least one SKU has quantity 0 and one has quantity 1 (for later edge-case tests)
  - **DoD:** Data queryable in DB on startup; documented in README as mock data.
  - **Depends on:** FUL-002

---

## Epic 1: Availability Service

- [ ] ### FUL-006: Availability entity & repository
  - **Description:** Model availability so it can be queried by SKU.
  - **AC:**
    - Entity: SKU (unique), available quantity, fulfillment source (store/DC), last-updated timestamp, **`@Version` field for optimistic locking** **[CHANGED]**
    - Flyway migration for the table
  - **DoD:** CRUD verified with a repository integration test.
  - **Depends on:** FUL-005

- [ ] ### FUL-007: GET /availability/{sku}
  - **Description:** As mock TikTok Shop, I need to check if a SKU can be fulfilled before ordering.
  - **AC:**
    - 200 with quantity + fulfillment source + estimated ETA (simple mock calculation) for a known SKU
    - 404 `ProblemDetail` for an unknown SKU
  - **DoD:** MockMvc tests for both cases; curl verified; Postman request added.
  - **Depends on:** FUL-004, FUL-006

- [ ] ### FUL-008: POST /availability/bulk-check
  - **Description:** Marketplaces check many SKUs at once.
  - **AC:**
    - Accepts a list of SKUs; returns an entry per SKU
    - Mixed known/unknown SKUs handled in one 200 response (unknown flagged, not an error)
    - Request validated (non-empty, max size, e.g. 100)
  - **DoD:** Tests with mixed payload, empty list (400) and oversized list (400); Postman request added.
  - **Depends on:** FUL-007

---

## Epic 2: Order Ingestion Service

- [ ] ### FUL-009: Define mock TikTok Shop order payload
  - **Description:** A realistic-looking fake external schema to build against. Use **their** field names, not internal ones.
  - **AC:** Example JSON with buyer, shipping address, **multiple line items**, marketplace order ID and idempotency key.
  - **DoD:** Saved at `/mocks/tiktok-order-example.json`, fields documented in a short table.
  - **Depends on:** none

- [ ] ### FUL-010: Internal Order + OrderItem entities **[CHANGED: multi-item is core]**
  - **Description:** BestBuy-style internal representation, separate from the marketplace schema.
  - **AC:**
    - `Order`: id, source marketplace, marketplace order ID, customer info (mock), status enum, idempotency key, timestamps
    - `OrderItem`: order reference, SKU, quantity (`Order 1 → N OrderItem`)
    - **Unique constraint** on (source marketplace, idempotency key)
    - Flyway migration
  - **DoD:** Round-trip integration test including multiple items; constraint violation verified by test.
  - **Depends on:** FUL-002

- [ ] ### FUL-011: POST /orders/ingest (TikTok-shaped)
  - **Description:** As mock TikTok Shop, I submit an order in my schema and it is accepted and translated to the internal format.
  - **AC:**
    - Accepts TikTok-shaped payload into an inbound DTO
    - Bean Validation on required fields; bad payload returns 400 with field-level errors
    - Translates to `Order` + `OrderItem`s and persists with status `RECEIVED`
    - Returns 201 with `orderId`, `status`, `source`
  - **DoD:** MockMvc tests for valid and invalid payloads; order visible in DB; curl + Postman verified.
  - **Depends on:** FUL-004, FUL-009, FUL-010

- [ ] ### FUL-012: Idempotency handling **[CHANGED: DB-enforced]**
  - **Description:** Marketplace webhooks retry on timeout. Must never double-create orders, even under concurrent retries.
  - **AC:**
    - Duplicate key returns the **original** order (200, `duplicate: true`), no second row
    - Implementation relies on the **unique constraint**; a constraint violation is caught and resolved to the original order (no "check then insert" race)
    - New key creates a new order as normal
  - **DoD:** Sequential duplicate test **and** a concurrent test (N parallel identical submissions, exactly one row). Curl verified.
  - **Depends on:** FUL-011

- [ ] ### FUL-013: GET /orders/{id}
  - **Description:** Fetch a single order's current state for verification/debugging.
  - **AC:** 200 with order, items and status; 404 for unknown ID.
  - **DoD:** MockMvc tests; curl verified; Postman request added.
  - **Depends on:** FUL-011

- [ ] ### FUL-014: Webhook signature check (optional realism) **[NEW, optional]**
  - **Description:** Real marketplaces sign webhooks. Verify authenticity of ingest calls.
  - **AC:** HMAC-SHA256 of the raw body validated against a shared secret from config; missing/invalid signature returns 401; constant-time comparison.
  - **DoD:** Tests for valid, invalid and missing signature; Postman pre-request script computes the signature.
  - **Depends on:** FUL-011

---

## Epic 3: Fulfillment Orchestration

- [ ] ### FUL-015: Order state machine + unit tests **[MERGED old FUL-012 + FUL-027]**
  - **Description:** Model the lifecycle after ingestion, with tests written alongside.
  - **AC:**
    - States: `RECEIVED → VALIDATED → ALLOCATED → SHIPPED → CONFIRMED`, plus `FAILED` and `CANCELLED` branches per the README table
    - Transition rules in one place (enum or dedicated class); terminal states have no exits
    - Transition table documented in README
  - **DoD:** Unit tests cover every valid transition and every invalid one (parameterized).
  - **Depends on:** FUL-010

- [ ] ### FUL-016: POST /orders/{id}/advance
  - **Description:** Manual advance for testing/debugging orchestration.
  - **AC:**
    - Advances to a requested/next valid state
    - Invalid transitions return 409 `INVALID_TRANSITION` naming from and to states
    - Status update is transactional and uses optimistic locking on `Order`
  - **DoD:** Curl walks an order through the full lifecycle; MockMvc tests for valid and invalid.
  - **Depends on:** FUL-013, FUL-015

- [ ] ### FUL-017: Auto-validation step (RECEIVED → VALIDATED)
  - **Description:** Validate SKUs exist and stock is sufficient, automatically.
  - **AC:**
    - Checks **every** order item against availability
    - All good → `VALIDATED`; unknown SKU or insufficient stock → `FAILED` with a recorded reason
    - Triggerable via service call/endpoint (event-driven trigger comes in Epic 4)
  - **DoD:** Tests for fulfillable, unknown-SKU, insufficient-stock and multi-item partial failure.
  - **Depends on:** FUL-008, FUL-016

- [ ] ### FUL-018: Allocation step (VALIDATED → ALLOCATED), concurrency-safe **[CHANGED]**
  - **Description:** Reserve inventory. Two orders competing for the last unit must not both succeed.
  - **AC:**
    - Quantity decremented on allocation for all items **atomically** (all or nothing)
    - Insufficient inventory → `FAILED`, nothing decremented
    - Concurrency protection via `@Version` with retry or a conditional `UPDATE ... WHERE qty >= :n`
    - Allocation records which items/quantities were reserved (needed later for cancellation release)
  - **DoD:** Inventory checked before/after via curl. **Concurrency test:** many parallel orders for a SKU with quantity 1, exactly one is `ALLOCATED`, quantity never negative.
  - **Depends on:** FUL-017

- [ ] ### FUL-019: Shipment + confirmation steps (mocked)
  - **Description:** Simulate remaining lifecycle since there's no warehouse system.
  - **AC:**
    - `ALLOCATED → SHIPPED` generates a fake tracking number (`MOCK-...`), `SHIPPED → CONFIRMED` completes
    - Tracking number stored and returned on `SHIPPED`+ orders
    - Cancelling an `ALLOCATED` order releases reserved inventory
  - **DoD:** Curl verified; test asserts tracking number present on SHIPPED+ and inventory restored on cancel.
  - **Depends on:** FUL-018

---

## Epic 4: Async Decoupling (Message Broker)

- [ ] ### FUL-020: Broker + OrderReceived event via transactional outbox **[CHANGED]**
  - **Description:** Decouple ingestion from orchestration. Avoid the dual-write problem: saving the order and publishing the event must not fail independently.
  - **AC:**
    - Broker added to `docker-compose.yml` (choice recorded in `DECISIONS.md`)
    - Ingest writes the `Order` and an `outbox_event` row in **one DB transaction**
    - A publisher process reads unsent outbox rows, publishes `OrderReceived`, marks them sent (at-least-once)
    - Orchestration consumes `OrderReceived` and starts processing; ingestion no longer calls orchestration directly
  - **DoD:** Curl submit → logs/DB show async pickup. Test: broker unavailable at ingest time → order still saved, event delivered after recovery.
  - **Depends on:** FUL-019

- [ ] ### FUL-021: Event-driven state transitions
  - **Description:** Each step is triggered by consuming an event and emits the next event.
  - **AC:**
    - Validation and allocation (at minimum) triggered by events, not direct calls
    - Events carry order ID and a unique event ID
    - Manual advance endpoint still works as a fallback/debug tool
  - **DoD:** Logs show the event flow for a full lifecycle; integration test reaches `CONFIRMED` without manual calls.
  - **Depends on:** FUL-020

- [ ] ### FUL-022: Duplicate and out-of-order event handling
  - **Description:** Brokers redeliver and reorder. Orchestration must be safe.
  - **AC:**
    - Consumer tracks processed event IDs (or is state-guarded) so redelivery causes no double processing
    - Event for an order already past that state is ignored and logged, not treated as an error
    - Inventory is never double-decremented
  - **DoD:** Test re-publishes the same event twice; assert single state change and single decrement.
  - **Depends on:** FUL-021

- [ ] ### FUL-023: Dead-letter handling **[PROMOTED from stretch]**
  - **Description:** Poison messages must not block the pipeline forever.
  - **AC:**
    - Consumer retries a failing event a configurable number of times, then routes it to a dead-letter destination with error metadata
    - Pipeline continues processing other orders
    - Dead-lettered order is left in a diagnosable state (log + reason)
  - **DoD:** Test injects a permanently failing event; it lands in DLQ and the next order still processes.
  - **Depends on:** FUL-022

---

## Epic 5: Adapter Layer

- [ ] ### FUL-024: Extract MarketplaceAdapter interface
  - **Description:** Put marketplace-specific translation behind an interface so new partners don't touch core logic.
  - **AC:** `MarketplaceAdapter` with `translateOrder(...)`; `TikTokShopAdapter` implements it; ingestion depends on the interface only.
  - **DoD:** All existing tests and curl calls still pass unchanged after the refactor.
  - **Depends on:** FUL-012

- [ ] ### FUL-025: Second mock adapter (GenericMarketplaceAdapter)
  - **Description:** Prove the abstraction with a differently-shaped payload.
  - **AC:**
    - New adapter handles a distinct schema (`/mocks/generic-order-example.json`)
    - Routing to the correct adapter by a `source` identifier (path variable or header)
    - Unknown source returns 400
  - **DoD:** Curl and tests verify both payload shapes ingest into the same internal model.
  - **Depends on:** FUL-024

---

## Epic 6: Resilience

- [ ] ### FUL-026: Stub carrier/warehouse HTTP service **[NEW]**
  - **Description:** Retries and circuit breakers are meaningless inside one JVM. Add a real network hop that can misbehave.
  - **AC:**
    - Shipment step calls a carrier API via HTTP client (`RestClient`/`WebClient`)
    - Stub implemented with WireMock or a small controller with **fault injection** (configurable latency, error rate, forced failure)
  - **DoD:** Happy path ships an order via the stub; each fault mode can be triggered on demand.
  - **Depends on:** FUL-019

- [ ] ### FUL-027: Retry with backoff
  - **Description:** Recover from transient failures on the carrier call.
  - **AC:** Resilience4j retry with configurable attempts and backoff; retries only on retryable errors (not 4xx).
  - **DoD:** Forced transient failure shows retry attempts in logs and eventual success; test covers exhausted retries.
  - **Depends on:** FUL-026

- [ ] ### FUL-028: Circuit breaker
  - **Description:** Prevent cascading failure if the carrier is down.
  - **AC:** Circuit breaker on the same call path; opens after threshold; fallback keeps the order in a retryable state (not silently lost); half-open recovery works.
  - **DoD:** Sustained failure shows open circuit + fallback in logs; recovery closes it. Breaker state visible in Actuator.
  - **Depends on:** FUL-027

- [ ] ### FUL-029: Timeouts
  - **Description:** Calls must not hang indefinitely.
  - **AC:** Explicit connect/read timeouts on the HTTP client; timeout triggers the retry/fallback path.
  - **DoD:** Slow stub response exceeds the timeout and behavior is confirmed by test and logs.
  - **Depends on:** FUL-028

---

## Epic 7: Observability

- [ ] ### FUL-030: Structured logging + correlation ID
  - **Description:** Trace an order's journey end to end.
  - **AC:**
    - Consistent log format; key events (received, validated, allocated, shipped, confirmed, failed, dead-lettered) include order ID
    - Correlation ID set via **MDC** at ingest and propagated through event headers to consumers
  - **DoD:** Grepping one order ID (or correlation ID) shows the full lifecycle.
  - **Depends on:** FUL-021

- [ ] ### FUL-031: Metrics
  - **Description:** Expose operational metrics like a real ops dashboard would pull.
  - **AC:** Micrometer counters: orders received, failed, confirmed, duplicates rejected, dead-lettered; timer for time-to-confirm if feasible.
  - **DoD:** Values visible at `/actuator/metrics/...` and change as orders are processed.
  - **Depends on:** FUL-030

---

## Epic 8: Packaging & Documentation

- [ ] ### FUL-032: End-to-end integration test
  - **Description:** Prove the full async pipeline automatically. (Per-ticket tests already exist; this is the capstone.)
  - **AC:** One Testcontainers test (Postgres + broker) runs ingest → `CONFIRMED`; a second covers an insufficient-stock order ending in `FAILED`.
  - **DoD:** Passes via terminal and in CI.
  - **Depends on:** FUL-025, FUL-031

- [ ] ### FUL-033: Dockerfile + full-stack compose
  - **Description:** One command to run everything.
  - **AC:** Multi-stage Dockerfile; compose runs app + Postgres + broker (+ stub) with health checks.
  - **DoD:** `docker compose up` on a clean checkout yields a working app.
  - **Depends on:** FUL-032

- [ ] ### FUL-034: Finalize Postman collection
  - **Description:** Polish the collection that has been built incrementally.
  - **AC:** Folders for availability, ingestion, order lookup, advance, fault-injection; example responses; environment file with variables; collection exported to `/postman/fulfillment-service.postman_collection.json`.
  - **DoD:** Fresh import runs every request successfully.
  - **Depends on:** FUL-033

- [ ] ### FUL-035: Finalize architecture README
  - **Description:** Explain the "why" behind design decisions.
  - **AC:** Complete the README sections marked _(finalized in FUL-035)_ using `DECISIONS.md`: overview, idempotency, async/outbox, adapters, safe allocation, resilience, run instructions.
  - **DoD:** A stranger can understand the design reasoning without reading the code.
  - **Depends on:** FUL-034

---

## Stretch (only after FUL-035)

- Rate limiting on the ingestion endpoint
- Cancellation endpoint + marketplace-facing cancel event
- CLI tool to walk an order through its lifecycle for demos
- Partial fulfillment (split shipments)
- Order search/list endpoint with pagination
