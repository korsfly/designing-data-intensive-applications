# Stream Joins và Fault Tolerance

---

## 1. Three Types of Joins in Stream Processing

### Challenge: Streams Are Unbounded

```
BATCH JOIN (Chapter 11):
   Both inputs: BOUNDED, complete
   Sort-merge or hash join: can scan entire input

STREAM JOIN:
   One or both inputs: UNBOUNDED
   Cannot wait for "all" events → must join AS EVENTS ARRIVE

→ Need to BUFFER some state while waiting for matching events
→ WINDOW: how long to wait for matching events
```

### Join 1: Stream-Stream Join (Windowed Join)

```
SCENARIO: Click events stream + Search events stream
   Goal: For each click, find the search that led to it

APPROACH:
   Index search events IN TIME WINDOW (e.g., last 1 hour)
   When click arrives: look up matching search in index

   Index search events: (session_id → search query, timestamp)
   Click arrives: lookup search for same session_id in last 1 hour

WINDOW SIZE CHOICE:
   Too small: miss searches that happened more than 1 hour before click
   Too large: too much memory (index grows large)

USE CASES:
   Recommend search queries based on what user clicked
   Detect fraudulent patterns (join login events + purchase events)

ORDERING SENSITIVITY:
   Search may arrive AFTER click (late events)
   → Must buffer both sides and check both directions
```

### Join 2: Stream-Table Join (Enrichment)

```
SCENARIO: User activity events stream + User profiles database
   Goal: Enrich each activity event with user demographic info

APPROACH 1: Query database for EACH EVENT
   For each event: lookup user_id in database
   → Simple but: high latency, heavy database load, not scalable

APPROACH 2: LOCAL TABLE COPY (table broadcast join)
   Load user profile table into LOCAL STATE of stream processor
   For each event: lookup in local state → no network call

   BUT: How to keep local copy UP-TO-DATE?
   → Database changes → need to propagate to stream processor
   → Use CDC stream (Change Data Capture!)

CDC TABLE JOIN:
   User profile changes → CDC stream → update stream processor's local copy
   Activity events → enriched with current profile data

   This effectively makes stream processor maintain a CACHE of the table
   Updated via CDC → always fresh

USE CASES:
   Add user demographics to clickstream events
   Add product info to purchase events
   Add IP geolocation to web requests
```

### Join 3: Table-Table Join (Maintaining Materialized View)

```
SCENARIO: Twitter-like social network
   Users table: (user_id, username, avatar)
   Follows table: (follower_id, followee_id)
   Tweets table: (tweet_id, author_id, content, timestamp)

Goal: Maintain HOME TIMELINE for each user
   (sorted list of tweets from people user follows)

APPROACH: Process all three as CHANGE STREAMS via CDC
   When user changes profile → update all tweets mentioning them in timelines
   When follow/unfollow → add/remove tweets from timeline
   When new tweet → add to followers' timelines

ALL THREE TABLES AS STREAMS → stream processor maintains TIMELINE MATERIALIZED VIEW

This is essentially STREAM PROCESSING equivalent of SQL MATERIALIZED VIEW
   → Chapter 3, Chapter 2: materialized views
   → Real-world: Materialize.com, Flink SQL
```

### Time Ordering in Joins

```
ALL THREE JOIN TYPES share a challenge:
   Events may ARRIVE OUT OF ORDER

Stream-Stream: both streams can have late events
Stream-Table: table updates may arrive before/after events that reference them
Table-Table: CDC events may be delayed

→ Joins must handle ordering gracefully
→ Use event timestamps + watermarks to manage ordering
→ Accept some join results may be APPROXIMATE (late data)
```

---

## 2. Fault Tolerance in Stream Processing

### Challenge: Harder Than Batch

```
BATCH FAULT TOLERANCE: simple
   Failed task → retry → same result (immutable input)
   Delete partial output → rerun → correct output

STREAM FAULT TOLERANCE: hard
   Continuous process → cannot "restart from beginning"
   Input: unbounded → cannot re-read entire history
   Output: already emitted → cannot un-emit (external side effects)

→ Need mechanisms to:
   1. Recover state after failure
   2. Ensure correct output despite failures
```

### At-Least-Once vs Exactly-Once

```
AT-MOST-ONCE (fire and forget):
   Process event → if fail → don't retry → may MISS events
   Result: incorrect (missing data)
   Simple: no bookkeeping

AT-LEAST-ONCE:
   Process event → if fail → RETRY
   Result: events processed 1 OR MORE times → DUPLICATES possible
   Need idempotent operators to handle duplicates

EXACTLY-ONCE:
   Process event → EXACTLY ONCE → neither lost nor duplicated
   Very hard to achieve truly
   Most "exactly-once" systems actually = at-least-once + idempotent
```

### Micro-Batching

```
APPROACH: small batches (e.g., 1 second intervals)
   Each micro-batch: processed as MINI-BATCH JOB
   → Natural checkpoint boundaries between batches

   If batch fails: retry that micro-batch → correct result

   Apache Spark Structured Streaming (legacy DStream API): micro-batching
   Kafka: commit offsets per micro-batch

PROS:
   Natural exactly-once semantics within micro-batch
   Familiar batch fault tolerance model

CONS:
   Higher latency (must wait for micro-batch to accumulate)
   Less "streamlike" (not truly event-by-event)
```

### Checkpointing (Flink)

```
APPROACH: Periodically SAVE SNAPSHOT of all operator state

Between checkpoints: process events without persisting state
If failure: RESTORE from last checkpoint → reprocess events since then

CHANDY-LAMPORT ALGORITHM (distributed snapshots):
   Used by Flink to take CONSISTENT GLOBAL SNAPSHOT
   Inject "checkpoint barrier" into event stream
   When barrier reaches operator: operator takes state snapshot → writes to storage
   All operators: coordinate to get globally consistent snapshot

   BARRIER IN STREAM:
   ... event 100 | BARRIER_5 | event 101 ...

   When operator sees barrier: "save my state as checkpoint 5"
   After barrier: continue processing events from offset 101

If failure:
   Restore all operators to checkpoint 5 state
   Re-read events from offset corresponding to checkpoint 5
   → Process events 101, 102,... again from checkpoint

PROS:
   Low overhead (infrequent checkpoints)
   Low latency (don't wait for micro-batch)

CONS:
   Events between checkpoint and failure → processed TWICE (duplicates possible)
   → Need idempotent output or exactly-once sink
```

### Idempotency + At-Least-Once = Effectively-Once

```
If operations are IDEMPOTENT:
   Processing event twice = same result as processing once
   → At-least-once delivery is FINE

IDEMPOTENT EXAMPLES:
   SET operation: set key X = value Y (idempotent)
   → Setting it twice = same result

   MAX operation: take max of existing and new value (idempotent)

   Unique ID deduplication:
   → Track seen event IDs → if seen before → skip
   → Only works if can store all seen IDs (or use time-bounded window)

NON-IDEMPOTENT EXAMPLES:
   INCREMENT counter (NOT idempotent)
   → Processing twice = double-counted

   SEND EMAIL (NOT idempotent)
   → Sends email twice → bad UX
```

### Exactly-Once via Atomic Transactions

```
APPROACH: Group operations into ATOMIC TRANSACTION
   → Either all committed, or all aborted + retried

DATABASE-INTERNAL exactly-once:
   Producer writes to output database
   Commit offset in SAME TRANSACTION as writing output
   → If fail: transaction aborted → retry from last committed offset
   → Guaranteed: each event processed once in output database

   Used by: Kafka Streams (local RocksDB state + Kafka offset in same transaction)

CROSS-SYSTEM exactly-once:
   Output to database + commit offset to broker in SAME TRANSACTION
   Requires: distributed transaction (2PC) — expensive!

   Kafka: "exactly-once semantics" (EOS) mode
   → Producer: idempotent writes + transactions
   → Consumer: read-committed isolation
   → Together: atomic write to output topic + commit offset
   → Works for Kafka-to-Kafka pipelines
```

### Maintaining Derived Data and Views

```
STREAM PROCESSOR: maintains DERIVED STATE
   Input: stream of events (CDC changes, user actions,...)
   Output: derived state (materialized view, aggregate, index)

   Derived state CHANGES continuously as new events arrive

MAINTAINING MULTIPLE VIEWS from same stream:
   Same event log → multiple consumers → multiple derived views
   Each consumer independently maintains its view
   → DECOUPLED: adding new consumer = just add new subscriber

KAFKA: append-only log as system of record
   Multiple consumers each maintain their own derived state
   Each can be at different offset (different lag)
   → Rebuild derived state from scratch by replaying from offset 0
```

---

## 3. Stream Processing Frameworks

### Apache Flink

```
DESIGNED AS: streaming engine (batch = bounded stream)
FAULT TOLERANCE: Chandy-Lamport checkpointing
EXACTLY-ONCE: via checkpoints + idempotent sinks
STATE BACKEND: RocksDB (persistent), Memory (fast)
APIs: DataStream API, Table API, Flink SQL
WINDOWING: event time, processing time, ingestion time
WATERMARKS: built-in support, pluggable watermark strategies
USE CASES: real-time analytics, CDC processing, feature stores
```

### Apache Kafka Streams

```
DESIGNED AS: library (not a separate cluster, runs in your app)
STATE: LOCAL RocksDB + changes persisted to Kafka (changelog topics)
FAULT TOLERANCE: replay from changelog topic on restart
EXACTLY-ONCE: Kafka EOS (idempotent producer + transactions)
APIs: Streams DSL (high-level), Processor API (low-level)
WINDOWING: tumbling, hopping, sliding, session
USE CASES: microservice event processing, CQRS, Kafka-native processing
```

### Apache Spark Structured Streaming

```
DESIGNED AS: stream processing on top of Spark batch engine
EXECUTION: micro-batching (default) OR continuous processing (experimental)
FAULT TOLERANCE: checkpointing of offsets + state
EXACTLY-ONCE: via idempotent write + transactional sinks
APIs: DataFrame/SQL (same as Spark batch!)
WINDOWING: event time windows
ADVANTAGE: same code for batch and stream (Kappa architecture)
USE CASES: unified batch+stream, Spark-heavy organizations
```

### Comparison

| Feature              | Flink             | Kafka Streams  | Spark Structured     |
| -------------------- | ----------------- | -------------- | -------------------- |
| **Latency**          | Very low (ms)     | Low (ms)       | Higher (micro-batch) |
| **Throughput**       | Very high         | High           | Very high            |
| **Cluster needed**   | Yes               | No (library)   | Yes                  |
| **Exactly-once**     | Checkpoints       | Kafka EOS      | Transactional sinks  |
| **State management** | RocksDB, flexible | Local RocksDB  | Checkpointed         |
| **Event time**       | Excellent         | Good           | Good                 |
| **Batch+Stream**     | Good              | Kafka only     | Excellent            |
| **SQL support**      | Flink SQL         | ksqlDB (Kafka) | Spark SQL            |

---

## Tóm tắt phần này

```
3 STREAM JOIN TYPES:
   Stream-Stream (windowed): index one stream, probe with other
   Stream-Table (enrichment): local cache of table, updated via CDC
   Table-Table: all tables as CDC streams → maintain materialized view

FAULT TOLERANCE:
   Batch: simple retry from immutable input
   Stream: harder (continuous, already emitted output, stateful)

AT-LEAST-ONCE + IDEMPOTENT = EFFECTIVELY-ONCE (practical approach)

EXACTLY-ONCE MECHANISMS:
   Micro-batching (Spark DStream): mini-batches as atomic units
   Checkpointing (Flink): Chandy-Lamport snapshots, reprocess since checkpoint
   Atomic transactions (Kafka EOS): atomic output write + offset commit

FRAMEWORKS:
   Flink: streaming-native, lowest latency, best event time support
   Kafka Streams: library, no separate cluster, Kafka-native EOS
   Spark Structured: unified batch+stream, highest throughput
```
