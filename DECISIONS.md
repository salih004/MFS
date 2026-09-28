# Decision Log

Short notes on non-obvious choices, written **when the decision is made**, not later. These feed the README "Design reasoning" section (FUL-035) and are your interview material.

Keep each entry to a few lines. Never delete an entry; if a decision changes, add a new one that supersedes it.

## Template

```
### D-XXX: <short title>
- **Date:**
- **Ticket:** FUL-XXX
- **Status:** Proposed | Accepted | Superseded by D-XXX
- **Context:** What problem or choice came up?
- **Options considered:** A / B / C
- **Decision:** What we chose
- **Why:** The reasoning
- **Trade-offs / consequences:** What we gave up, what to watch for
```

---

## Decisions

### D-001: Modular monolith, not microservices
- **Ticket:** FUL-001
- **Status:** Accepted
- **Context:** The domain has distinct areas (availability, ingestion, orchestration).
- **Options considered:** Separate services / single app with package boundaries
- **Decision:** One Spring Boot app, packages as module boundaries.
- **Why:** Solo project. Network hops add operational cost without teaching more than clean boundaries do.
- **Trade-offs:** No independent deploy/scale. Boundaries rely on discipline. Modules interact only via interfaces or events so a split stays possible.

### D-002: Postgres + Flyway from day one (no H2)
- **Ticket:** FUL-002
- **Status:** Accepted
- **Context:** H2 is convenient but differs from Postgres in locking, constraints and SQL behavior.
- **Decision:** Postgres in Docker Compose, Testcontainers in tests, Flyway migrations, `ddl-auto=validate`.
- **Why:** Concurrency and constraint-based idempotency must be proven against a real engine.
- **Trade-offs:** Tests need Docker. Slightly slower feedback.

### D-003: Message broker choice (OPEN, decide at FUL-020)
- **Ticket:** FUL-020
- **Status:** Proposed
- **Options considered:** Kafka (log-based, replay, partitions, consumer groups) / RabbitMQ (queues, simpler routing, built-in DLX)
- **Leaning:** Kafka via Spring Kafka (KRaft mode, single container). It is common in retail-scale systems and teaches partitioning, ordering and offsets.
- **Why it might change:** RabbitMQ is lighter to run and its dead-lettering is simpler. Either satisfies the tickets.
- **Action:** Record the final choice and reasoning here before starting FUL-020.

### D-004: Idempotency enforced by a database unique constraint
- **Ticket:** FUL-010 / FUL-012
- **Status:** Accepted
- **Context:** Webhook retries can arrive concurrently.
- **Decision:** Unique (source marketplace, idempotency key); catch the violation and return the original order.
- **Why:** "Check then insert" has a race window. The database is the only place that can guarantee uniqueness.
- **Trade-offs:** Requires handling the exception path explicitly and testing it under parallel load.

### D-005: Transactional outbox for event publishing
- **Ticket:** FUL-020
- **Status:** Accepted
- **Context:** Saving an order and publishing an event are two systems. Either can fail independently (dual-write problem).
- **Decision:** Write the order and an outbox row in one transaction; a separate publisher relays to the broker.
- **Why:** Guarantees no order without an event and no event without an order.
- **Trade-offs:** At-least-once delivery, so consumers must be idempotent (FUL-022). Adds a table and a publisher process.

### D-006: Inventory concurrency via optimistic locking (or conditional update)
- **Ticket:** FUL-018
- **Status:** Proposed
- **Options considered:** `@Version` + retry / `UPDATE ... WHERE qty >= :n` / pessimistic `SELECT FOR UPDATE`
- **Note:** Choose one and record why. Whichever is chosen, it must be proven by the parallel-request test.
