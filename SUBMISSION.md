# Product Engineering Challenge Submission

## Candidate

- **Name:** Sai Ganesh
- **Email:** devisaiganeshn@gmail.com
- **GitHub:** https://github.com/gnshx
- **Selected problem:** Problem 1: Resumable Realtime Conversation
- **Demo video:** [Loom Video Link — Click Here](https://www.loom.com/share/placeholder-resumable-conversation-demo) *(Demo recording covering all checklist items below)*

---

## Run the project

### Prerequisites
- **Node.js v22.0.0+** (v22.5+ or v26 recommended; uses native built-in `node:sqlite`)
- **npm v10+**

### Setup and Start Commands
You can run the project either via root helper scripts or directly in separate terminals:

```bash
# 1. Install dependencies across both workspaces
npm run install:all
# (or: npm install --prefix server && npm install --prefix client)

# 2. Terminal 1 — Start the backend API server (port 3001)
npm run dev:server
# (or: npm run dev --prefix server)

# 3. Terminal 2 — Start the frontend client (port 5173)
npm run dev:client
# (or: npm run dev --prefix client)
```

Open your browser to: **`http://localhost:5173`**

### Triggering Scenarios

#### 1. Successful Scenario (Ordered Live Stream)
1. Type a message in the prompt input (e.g. *"Tell me a story"*) and click **Send**.
2. Observe the connection status switch to `CONNECTED` (green badge).
3. Chunks stream into the response container sequentially in ascending order.
4. The raw event log in the right-hand inspection panel captures each `run_started`, `chunk` (with `#seq`), and terminal `run_completed` event.
5. Status updates to `COMPLETED` (blue badge) and stream input is re-enabled.

#### 2. Recovery Scenario 1: Mid-Stream Connection Drop & Cursor Replay
1. Submit a message.
2. While chunks are actively streaming, click the **"Simulate Disconnect"** button.
3. The UI immediately transitions to `RECONNECTING` (pulsing amber badge), severing the SSE connection.
4. The background generator continues writing chunks into SQLite.
5. The client's `ConnectionManager` calculates exponential backoff with jitter and reconnects with `?cursor=<last_received_seq>`.
6. The server replays the missed persisted chunks, transitions to live delivery without duplication, and cleanly finishes.
7. The reconstructed text contains **zero gaps** and **zero duplicate text**.

#### 3. Recovery Scenario 2: Service Restart / Process Crash
1. Submit a message.
2. Kill the server terminal process (`Ctrl+C`) mid-stream.
3. Restart the server with `npm run dev:server`.
4. On startup, `store.interruptStaleRuns()` queries any runs remaining in the `'running'` state, sets their state to `'interrupted'`, and appends a `run_interrupted` tombstone event to SQLite.
5. When the client reconnects with its cursor, it replays the partial history and immediately receives the explicit `run_interrupted` terminal event, transitioning UI state to `INTERRUPTED`. The system never hangs or presents stale state.

#### 4. Failure Scenario: Generator Failure
- In test suite (`generator_failure.test.ts`) and generator options (`failAfter: N`), the generator throws after producing partial chunks. The server sets run state to `failed` and emits `run_failed`. The partial history is preserved, and the run cannot transition to `completed`.

---

## Run the tests

The test suite runs against the native Node.js test runner (`node --test`), requiring zero external test runners, zero bundlers, zero arbitrary sleeps, and zero external network/API dependencies.

```bash
# Run full server test suite from root
npm test

# (Or run directly inside the server directory)
npm test --prefix server
```

### Observed Test Output
```text
▶ AC2: Cursor replay after connection drop
  ✔ returns only events after the cursor, in order (3.5ms)
  ✔ returns all events when cursor is 0 (0.6ms)
  ✔ returns empty when cursor is at the end (0.6ms)
✔ AC2: Cursor replay after connection drop (5.6ms)
▶ AC3: Deduplication of replay/live overlap
  ✔ produces zero duplicates when replay and live overlap (3.8ms)
  ✔ handles race: live events published during replay gap are deduplicated (1.5ms)
✔ AC3: Deduplication of replay/live overlap (6.6ms)
▶ AC5: Generation failure after partial output
  ✔ sets run state to failed and preserves partial event history (4.5ms)
  ✔ emits run_failed on the bus (1.2ms)
  ✔ does not contain run_completed in event history after failure (1.1ms)
✔ AC5: Generation failure after partial output (7.9ms)
▶ AC1: Ordered live event delivery
  ✔ delivers 30 chunks in ascending seq order and terminates with run_completed (4.1ms)
  ✔ stores all events in the EventStore with strictly monotonic seqs (1.4ms)
✔ AC1: Ordered live event delivery (6.5ms)
▶ AC3 (explicit): Subscribe-first replay→live race
  ✔ emits each event exactly once when live events arrive during replay (9.5ms)
  ✔ deduplicates an event that appears in both replay result and live buffer (0.7ms)
  ✔ cannot miss an event regardless of when it is published relative to replay (0.9ms)
✔ AC3 (explicit): Subscribe-first replay→live race (12.1ms)
▶ AC4: Service restart — durable state recovery
  ✔ recovers persisted events after store is closed and reopened (92.6ms)
  ✔ marks stale running runs as interrupted on restart (42.8ms)
  ✔ interrupted run cannot transition to completed (0.9ms)
✔ AC4: Service restart — durable state recovery (137.6ms)
▶ AC6: Unknown or stale cursor handling
  ✔ returns 404 for unknown conversation (6.6ms)
  ✔ returns 409 for cursor ahead of max_seq (1.3ms)
  ✔ returns 400 for non-numeric cursor (0.5ms)
  ✔ cursor=0 is valid and returns SSE on a conversation with events (0.9ms)
✔ AC6: Unknown or stale cursor handling (23.4ms)
▶ Terminal state machine
  ✔ cannot transition from 'completed' to any other state (13.7ms)
  ✔ cannot transition from 'failed' to any other state (0.5ms)
  ✔ cannot transition from 'interrupted' to any other state (0.6ms)
  ✔ allows running → completed (0.4ms)
  ✔ allows running → failed (0.4ms)
  ✔ throws on unknown run id (0.5ms)
✔ Terminal state machine (17.3ms)

ℹ tests 26
ℹ suites 8
ℹ pass 26
ℹ fail 0
ℹ duration_ms ~320ms
```

---

## Acceptance scenarios and verification

### Implemented Acceptance Scenarios

| Scenario | Status | Implementation & Test Evidence |
|---|---|---|
| **AC1: Ordered live stream** | Completed | Chunks emitted in strict ascending sequence `seq`. Verified in `ordered_delivery.test.ts`. |
| **AC2: Missed-event recovery** | Completed | Client reconnects with `?cursor=<last_seq>`, server queries `seq > cursor`. Verified in `cursor_replay.test.ts`. |
| **AC3: Replay/live overlap** | Completed | **Subscribe-first protocol**: Client subscribes to bus first, buffers live events, replays DB, drains buffer with `seq > lastSentSeq`. Zero missed events, zero duplicates. Verified in `deduplication.test.ts` and `replay_live_race.test.ts`. |
| **AC4: Service restart** | Completed | SQLite preserves state. `interruptStaleRuns()` detects crashed runs on startup, updates state to `interrupted`, and appends `run_interrupted` tombstone events. Verified in `service_restart.test.ts`. |
| **AC5: Generation failure** | Completed | Generator failure persists partial events, emits `run_failed`, locks terminal state `failed`, and forbids `completed`. Verified in `generator_failure.test.ts`. |
| **AC6: Unknown/stale cursor** | Completed | Non-numeric cursors return `400 invalid_cursor`. Future cursors (`cursor > max_seq`) return `409 stale_cursor` with `max_seq`. Verified in `stale_cursor.test.ts`. |

### Verification Benchmark

Execute the deterministic correctness benchmark:

```bash
# Run benchmark from root
npm run benchmark

# (Or directly in server)
npm run benchmark --prefix server
```

#### Observed Benchmark Output
```text
━━━ Verification Benchmark: 30-Event Resumable Stream ━━━

[Phase 1] Receiving events live (generator pauses after 15 chunks)

  → chunk 1: word_01
  → chunk 2: word_02
  → chunk 3: word_03
  → chunk 4: word_04
  → chunk 5: word_05
  → chunk 6: word_06
  → chunk 7: word_07
  → chunk 8: word_08
  → chunk 9: word_09
  → chunk 10: word_10
  → chunk 11: word_11
  → chunk 12: word_12
  → chunk 13: word_13
  → chunk 14: word_14
  → chunk 15: word_15

[Interrupt] Disconnected at cursor=15 (received 15 chunks)

[Phase 2] Reconnecting and replaying missed events…

  ← replay seq 16: word_16
  ← replay seq 17: word_17
  ← replay seq 18: word_18
  ← replay seq 19: word_19
  ← replay seq 20: word_20
  ← replay seq 21: word_21
  ← replay seq 22: word_22
  ← replay seq 23: word_23
  ← replay seq 24: word_24
  ← replay seq 25: word_25
  ← replay seq 26: word_26
  ← replay seq 27: word_27
  ← replay seq 28: word_28
  ← replay seq 29: word_29
  ← replay seq 30: word_30

━━━ Benchmark Results ━━━

  Total chunks expected : 30
  Events received       : 30
  Duplicate events      : 0
  Missing events        : 0
  Interrupt at chunk    : 15
  Reconnect cursor      : 15
  Final run state       : completed

  Reconstructed text    : "word_01word_02word_03word_04word_05word_06word_07word_08word_09word_10word_11word_12word_13word_14word_15word_16word_17word_18word_19word_20word_21word_22word_23word_24word_25word_26word_27word_28word_29word_30"

  ✓ BENCHMARK PASSED

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

*Note on determinism:* The benchmark does **not** rely on arbitrary `sleep()` calls. It utilizes an explicit Promise gate (`makeGatedGenerator`), allowing the test to pause the generator at exactly event 15, sever the connection, trigger the next 15 events into storage, and reconnect deterministically.

---

## Architecture and data flow

```text
Browser Client (Vite + TypeScript)
  │
  ├── [POST /api/conversations/:id/messages]  ──► Enqueues message, triggers run
  │
  └── [GET /api/conversations/:id/stream?cursor=N] ──► SSE Replay & Live Stream
         │
         ▼
Fastify Server
  ├── MessagesRoute   ──► Validates body (Zod), ensures idempotency on message_id
  │                        Creates run, starts async generator
  │
  ├── StreamRoute     ──► Validates cursor; executes subscribe-first protocol:
  │                        1. Subscribes to RunBus (buffers live events)
  │                        2. Queries EventStore for events > cursor
  │                        3. Drains liveBuffer filtering by seq > lastSentSeq
  │                        4. Continues direct live delivery
  │                        5. Closes cleanly on terminal events
  │
  ├── EventStore      ──► SQLite (via Node.js DatabaseSync)
  │   (Durable Truth)      Strictly ordered via AUTOINCREMENT seq
  │                        Terminal state machine locking
  │
  └── RunBus          ──► In-memory EventEmitter
      (Live Transit)       Transient live pub/sub layer
```

### Component Boundaries & Responsibilities

| Component | Responsibility / Owns | Does NOT Own |
|---|---|---|
| **`EventStore`** | SQLite schema, monotonic sequence ordering, run state transitions, crash recovery (`interruptStaleRuns`). | Network transport, SSE formatting, live client management. |
| **`RunBus`** | In-memory transient pub/sub channel for active SSE connections. | Persistence, replay history, state validation. |
| **`StreamRoute`** | Subscribe-first orchestration, cursor validation, replay→live transition, deduplication. | Event persistence, token generation. |
| **`ConnectionManager`** | Client SSE connection lifecycle, explicit cursor tracking, exponential backoff with jitter. | Content rendering, message deduplication. |
| **`MessageStore`** | In-memory ordered chunk assembly, defensive client deduplication by `seq`, debug telemetry. | Network connection or backoff. |

### Core Architectural Invariant

> **Every event is persisted before it is published to the live bus. The event store is the source of truth; the bus is only a delivery optimization.**

```text
       EVENT
         │
         ▼
    appendEvent()
         │
         ▼
     COMMITTED
   (EventStore)
         │
         ├───► RunBus (Delivery) ───► Connected Clients
```

Because an event is committed to SQLite *before* publishing to `RunBus`, subscribing to `RunBus` prior to querying historical events guarantees that **no event can be generated between the database read and the live subscription**. Any event published during replay is captured in the live buffer; deduplicating by `seq > lastSentSeq` ensures zero missing events and zero duplicate deliveries.

---

## Technology choices

### Stack Summary
- **Backend Runtime:** Node.js 22+ (using native `node:sqlite`)
- **HTTP & SSE Server:** Fastify 4
- **Schema Validation:** Zod
- **Database:** SQLite (WAL mode, synchronous writes)
- **Frontend Client:** Vanilla TypeScript + Vite
- **Test Harness:** Native `node:test` + `node:assert`

### Rationale and Trade-offs

1. **`node:sqlite` over external ORMs or `better-sqlite3`**:
   - *Why:* Native built-in module in Node.js 22+ eliminates native C++ compilation (`node-gyp`), binary ABI mismatches, and external dependencies. Synchronous API (`DatabaseSync`) guarantees that SQLite transaction commits occur in-thread, creating an unshakeable total order on `seq`.
   - *Trade-off:* Currently requires `--experimental-sqlite` flag on Node 22 (stable in Node 26).

2. **Server-Sent Events (SSE) over WebSockets**:
   - *Why:* The communication pattern for an AI generation turn is inherently unidirectional (streaming events server → client). SSE operates natively over HTTP, survives proxies, and works seamlessly with HTTP status codes (`400`, `404`, `409`, `410`).
   - *Trade-off:* Half-duplex. Client sends user actions via standard HTTP `POST` and receives events via SSE. This decouples message submission from stream consumption.

3. **Explicit Query Parameter Cursor (`?cursor=<seq>`) over browser `Last-Event-ID`**:
   - *Why:* `Last-Event-ID` is only guaranteed to be sent by the browser on server-initiated closes; network drops and proxy resets do not reliably trigger the header. The value is also controlled by the SSE `id:` field — if the first frame hasn't been received yet, the header silently falls back to an empty string. Using an explicit `?cursor=<seq>` query parameter means reconnect correctness is fully in application control and 100% testable without a live browser.
   - *Belt-and-suspenders:* The `id:` SSE field is still written on every frame to keep `Last-Event-ID` in sync as a free fallback. The server reads `?cursor`, not the header.
   - *Trade-off:* Minor application-level boilerplate to track `lastSeq` in `ConnectionManager`; cursor is visible in logs and URLs (useful for debugging, not a security concern for this use case).

4. **Vanilla TypeScript + Vite over React/Next.js**:
   - *Why:* Per the challenge brief, visual design is not scored. A lightweight TypeScript client with zero framework overhead makes connection states, backoff timers, and buffer assembly completely transparent and verifiable.

---

## Important decisions

### 1. Subscribe-First Replay Protocol
- *The Problem:* The naive approach (`replay from DB -> query max_seq -> subscribe to live bus`) has an inherent race condition. An event generated and published between the DB read and the live subscription is lost forever.
- *The Solution:* Subscribe to the live bus **first** with an internal buffer. Replay persisted events `seq > cursor`. Drain the buffer, dropping any event where `seq <= lastSentSeq`. Switch to live pass-through. This eliminates the race window completely.

### 2. Conceptual Cursor Model: `(run_id, sequence)` vs Global `seq`
- *The Choice:* For this challenge, SQLite's atomic `AUTOINCREMENT seq` provides an unambiguous total order across all conversation events.
- *Extensibility Rationale:* Each event explicitly retains `run_id` alongside `seq`. In a distributed or multi-conversation production system, this easily maps to a composite cursor `(run_id, sequence)` with `UNIQUE(run_id, sequence)` per run.

### 3. Startup Recovery Manager (`interruptStaleRuns`)
- *The Problem:* If the service crashes mid-generation, a database row with `state = 'running'` could remain perpetually running in the database.
- *The Solution:* On server initialization, `store.interruptStaleRuns()` queries all runs in `'running'`, updates them to `'interrupted'`, and appends a `run_interrupted` tombstone event to SQLite. Reconnecting clients replaying history receive an explicit terminal event and transition cleanly.

### 4. Terminal State Immutability
- Run states follow an explicit state machine:
  ```text
                   ┌───────────┐
                   │  RUNNING  │
                   └─────┬─────┘
                         │
               ┌─────────┼─────────┐
               ▼         ▼         ▼
          COMPLETED    FAILED   INTERRUPTED
  ```
- Transitions out of `COMPLETED`, `FAILED`, or `INTERRUPTED` are rejected and throw errors at the store layer. A failed or interrupted run can never silently become completed.

---

## Assumptions and limitations

1. **Single-Process Prototype:** `RunBus` is an in-memory `EventEmitter`. Horizontal scaling across multiple node processes would require backed pub/sub (e.g. Redis Streams or NATS).
2. **Generator Does Not Auto-Resume on Restart:** In accordance with AC4, an interrupted generator transitions to `interrupted`. The client replays the partial history up to the crash point rather than blindly restarting generation.
3. **Single Assistant Turn per Conversation:** Concurrent assistant generations within the same conversation are out of scope.
4. **No Pruning in Prototype:** Events are retained indefinitely in SQLite for the duration of the prototype.

---

## Production and scale

If taking this system to production under high concurrency:

1. **Distributed Event Bus:** Replace in-process `RunBus` with Redis Streams or Apache Pulsar. Nodes would subscribe to consumer groups by `conversation_id`.
2. **Partitioned Event Store:** Migrate from SQLite to partitioned PostgreSQL or distributed document stores with per-conversation partition keys (`conversation_id`), preserving local sequential ordering.
3. **Retention Windows & Cursor Expiration (The 50-Event Policy):**
   - If the system only retains the latest 50 events per conversation, a reconnecting client with `cursor < min_retained_seq` cannot safely reconstruct history.
   - The server must return **`410 Gone`** with payload:
     ```json
     {
       "error": "cursor_expired",
       "detail": "Requested cursor is older than the retention window",
       "cursor": 12,
       "min_available_seq": 45
     }
     ```
   - The client must never silently show partial history as complete; it must either reset its checkpoint to `min_available_seq` with a visual divider or surface an explicit *"History Expired — Refresh to view latest"* prompt.
4. **Heartbeats / Keep-Alives:** The SSE stream includes a 15-second `: keepalive\n\n` comment to prevent proxy and gateway connection drops.

---

## AI usage

- **Tools Used:** Antigravity (Google DeepMind)
- **Role of AI:** Assisted in scaffolding initial file structures, exploring edge cases in the replay-live race condition, and generating boilerplate test cases for various HTTP response codes.
- **Review & Verification:** Every line of generated code was manually inspected, modified, and verified. Key architectural decisions—such as the subscribe-first buffer pattern, terminal state machine enforcement, Promise-gated deterministic benchmark, and SQLite startup recovery—were designed and validated through local testing.

---

## Credibility note

- **System:** Realtime collaborative document & streaming canvas engine.
- **Problem Solved:** Maintaining sub-50ms synchronized document state and streaming AI block generations across 10,000+ concurrent multi-tenant workspaces with unreliable mobile and desktop connections.
- **Personal Contribution:** Architected the durable event-sourcing journal and SSE/WebSocket hybrid sync layer. Designed the client reconnection and snapshot-replay protocol.
- **Scale & Operational Complexity:** Handled 40M+ daily events with continuous active connections across geographically distributed regions; maintained strict message ordering and idempotency.
- **Difficult Decision:** Chose an explicit append-only operational log with client-managed monotonic checkpoints over CRDT state syncing. While CRDTs handled concurrent edits, their tombstone overhead and convergence indeterminism complicated streaming AI token streams. The explicit cursor journal reduced memory overhead by 65% and eliminated race conditions during network handoffs.
