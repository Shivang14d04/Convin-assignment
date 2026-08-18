# Solution Report

## 1. What was broken, and why?

While reviewing the codebase and logs, I found four main issues that caused the server crashes, duplicate data, and lost tasks:

### Issue 1: Concurrent Map Write Crash (`internal/stats/cache.go`)
* **What went wrong:** Under high request traffic, the server crashed with a `fatal error: concurrent map writes` panic.
* **Why it happened:** Go maps aren't thread-safe. `Get()` was using a read lock (`c.mu.RLock()`), but `Record()` was writing to the map without locking `c.mu.Lock()`. When multiple webhooks hit the endpoint at the same time, simultaneous map writes crashed the program.

### Issue 2: Duplicate Webhooks & Inflated Call Stats (`internal/store/store.go` & `internal/ingest/service.go`)
* **What went wrong:** Retried webhooks kept creating duplicate rows in the database and inflating account call counts.
* **Why it happened:** The `events` table in Postgres didn't have a `UNIQUE` constraint on `event_id`. Also, checking `EventExists()` first and then calling `InsertEvent()` wasn't atomic. Two duplicate requests arriving at the exact same moment both passed the check, inserted duplicates, and incremented stats twice.

### Issue 3: Recordings Never Marked as Processed (`internal/ingest/service.go`)
* **What went wrong:** Background recording jobs failed silently, leaving `recording_processed = false` in the database.
* **Why it happened:** The background goroutine was given `r.Context()` from the HTTP request. As soon as the HTTP handler returned HTTP 200 to the user, Go canceled that context. When the background job tried to update Postgres after its delay, `MarkRecordingProcessed()` failed because the context was already canceled.

### Issue 4: In-Flight Tasks Lost During Server Shutdown (`cmd/server/main.go`)
* **What went wrong:** Whenever the server was restarted or redeployed, background recording jobs vanished.
* **Why it happened:** The server shut down HTTP listeners when receiving a shutdown signal (`SIGTERM`), but it exited immediately without waiting for background goroutines to finish work.

---

## 2. Deduplication Strategy & Why I Chose It

### My Approach: PostgreSQL `ON CONFLICT DO NOTHING`
I added a `UNIQUE` constraint on `event_id` in PostgreSQL (`migrations/001_init.sql`) and changed `InsertEvent()` to use an atomic SQL query:

```sql
INSERT INTO events (event_id, call_id, account_id, payload)
VALUES ($1, $2, $3, $4)
ON CONFLICT (event_id) DO NOTHING
RETURNING event_id;
```

If Postgres returns no rows, `InsertEvent()` returns `inserted = false`, and `Ingest()` immediately exits without updating stats or call records.

### Why I picked this over other solutions:
* **Database Atomic Guarantee:** Doing deduplication inside Postgres using `ON CONFLICT` is 100% thread-safe and eliminates race conditions cleanly without needing complex locking code in Go.
* **No Extra Infrastructure:** Postgres is already running as the main database. Using it avoids adding extra tools or external dependencies.

### Other options considered:
* **Redis `SETNX` (Key Locking):** Fast, but if Redis key expires or fails before Postgres saves the event, duplicates can still leak through.
* **In-Memory Go Mutex / Map:** Only works for a single server process and completely breaks if we run multiple server instances behind a load balancer.

---

## 3. How I would scale this to 10,000 Webhooks / Second

If traffic scales up to 10,000 requests/sec, here is how I would redesign the architecture:

1. **Use a Message Queue (Kafka / AWS SQS):**
   * The HTTP handler should just validate the request payload and quickly push the event into a message queue (like Kafka), returning `HTTP 202 Accepted` in under 5ms.
2. **Redis Ingress Deduplication:**
   * Place Redis `SETNX` (with a 24-hour TTL) at the API gateway layer to filter out duplicate webhooks before they reach the queue or database.
3. **Database Batch Inserts:**
   * Have background workers read messages from Kafka in batches (e.g. 500 events at a time) and do bulk inserts into PostgreSQL every 100ms instead of doing single-row inserts per request.
4. **Distributed Caching for Stats:**
   * Store and aggregate account call stats in Redis Cluster instead of in-memory Go maps so stats can be shared across all app instances.
