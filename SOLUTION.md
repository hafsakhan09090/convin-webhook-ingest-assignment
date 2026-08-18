# Solution

## What was broken

The in-memory stats cache updated a shared map without taking its mutex. Concurrent webhook deliveries could therefore race, lose updates, or crash with concurrent map writes. I added a concurrent regression test and protected `Cache.Record` with the existing mutex.

The webhook handler also started recording work in a goroutine using the HTTP request context. Once the handler returned, that context could be cancelled, causing recording processing to fail silently because its error was discarded. Recording work now receives a bounded background context and failures are logged.

The store already provides an atomic ingestion transaction. The service is now wired to use it, so event insertion, call upsert, and account-stat updates succeed or fail together.

## Idempotency strategy

I chose Postgres as the idempotency authority because `event_id` is stable and the database is shared by all service instances. The transaction inserts the event with `ON CONFLICT (event_id) DO NOTHING`. If no row is inserted, the delivery is treated as a successful duplicate; call and statistics updates are skipped. This prevents double-counting and avoids Redis expiry or cross-instance consistency issues.

Redis could reduce load with short-lived deduplication keys, but I would keep Postgres as the durable source of truth.

## At 10,000 webhooks/second

I would keep the transactional idempotency key, add connection-pool and database monitoring, and batch or partition durable writes where needed. Recording processing should move from an in-process goroutine to a durable outbox/queue with retry, backoff, and worker consumers. That would ensure work survives deployment or process failure while keeping webhook acknowledgement fast.