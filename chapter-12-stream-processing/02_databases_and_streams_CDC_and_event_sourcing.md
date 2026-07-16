# Databases và Streams: CDC và Event Sourcing

---

## 1. Vấn đề: Keeping Systems in Sync

### Heterogeneous Data Systems

```
Thực tế: NO SINGLE SYSTEM can satisfy ALL needs
   → Most applications COMBINE multiple technologies:

   OLTP database: serve user requests (fast reads/writes)
   Cache: speed up common requests
   Full-text search index: handle search queries
   Data warehouse: analytics

   → Each has its OWN COPY of data, optimized for its own purpose
   → All copies must be KEPT IN SYNC
```

### Dual Writes — Vấn đề và Cạm bẫy

```
APPROACH 1: Periodic full database dumps + ETL
   → Too slow (daily ETL = 24-hour data lag)

APPROACH 2: DUAL WRITES
   Application code explicitly writes to EACH system:
   db.update(x, "A")
   searchIndex.update(x, "A")
   cache.invalidate(x)

PROBLEM 1: Race Condition (Figure 12-4)
   Client 1 sets x=A, Client 2 sets x=B simultaneously

   Database: sees C1 first → x=A, then C2 → x=B  (final: B)
   Search:   sees C2 first → x=B, then C1 → x=A  (final: A)

   → Database says x=B, Search says x=A → PERMANENTLY INCONSISTENT

PROBLEM 2: Partial failure
   One write succeeds, another fails
   → Two systems inconsistent

FIX for race condition: Atomic commit (2PC) across systems
   → Very expensive (Chapter 8)

BETTER APPROACH: Make ONE system the LEADER → others follow its log
```

---

## 2. Change Data Capture (CDC)

### Định nghĩa

```
CDC: process of OBSERVING ALL DATA CHANGES written to a database
   and EXTRACTING them in a form that can be replicated to other systems

Key: changes available as STREAM, immediately as they are written

DATABASE = LEADER (source of truth)
OTHER SYSTEMS = followers (consume change stream)
   → Search index, cache, data warehouse: all just CONSUMERS

Fixes Figure 12-4 race condition (Figure 12-5):
   Database decides ORDER of concurrent writes → writes to replication log
   Other systems CONSUME LOG IN SAME ORDER
   → Permanent consistency guaranteed
```

### Implementing CDC

```
BASIS: Logical replication logs (Chapter 6, "Logical/Row-based log replication")
   → Each row INSERT/UPDATE/DELETE → event in CDC stream

Open source tools:
   DEBEZIUM: CDC source connectors for MySQL, PostgreSQL, Oracle, SQL Server,
             Cassandra, many others
             → Attaches to replication log → standard event schema
             → Handles schema changes, updates modeling

   KAFKA CONNECT: CDC connectors for many databases + Kafka
   MAXWELL: MySQL binlog parsing
   PGCAPTURE: PostgreSQL CDC
   GOLDENGATE: Oracle CDC

CDC is ASYNCHRONOUS:
   Source DB does NOT wait for consumer before committing
   → Similar to async replication lag (Chapter 6)
   → Adding slow consumer: minimal impact on source DB
```

### Initial Snapshot

```
PROBLEM: Cannot start CDC from the beginning of time
   → Log too old or truncated → missing historical data

SOLUTION: Start with CONSISTENT SNAPSHOT of entire database
   Then apply CDC changes FROM that snapshot's offset

DEBEZIUM: Netflix's DBLog watermarking algorithm
   → Provides INCREMENTAL SNAPSHOTS (don't need to pause writes)
```

### Log Compaction

```
PROBLEM: Keeping full change history = too much disk space

LOG COMPACTION: periodically discard old entries
   Keep only MOST RECENT value for each key
   (Tombstone marker for deleted keys)

Result: Compacted log contains current state of ALL keys
   New consumer starts from offset 0 → gets FULL DATABASE SNAPSHOT
   → No need to take snapshot from source database!

Kafka supports log compaction:
   → Message broker usable as DURABLE STORAGE, not just transient messaging
   → Very powerful: message broker = database + notification system
```

### CDC vs Database Schema Changes

```
PROBLEM: CDC exposes internal database schema as PUBLIC API
   Removing/renaming column → breaks downstream consumers

OUTBOX PATTERN:
   Developer writes to:
   1. Internal domain model tables (for application use)
   2. OUTBOX TABLES (separate, CDC-exposed schema)
   Both writes in SAME TRANSACTION → no dual-write race condition!

   Downstream consumers read from outbox (not internal tables)
   → Developers can change internal schema freely
   → Outbox schema = public contract

TRADE-OFFS of Outbox:
   - Must maintain transformation: internal schema → outbox schema
   - More data written to database (performance impact)
```

---

## 3. CDC vs Event Sourcing

| Aspect             | CDC                                          | Event Sourcing                                 |
| ------------------ | -------------------------------------------- | ---------------------------------------------- |
| **Level**          | Low-level (database replication log)         | Application level (business events)            |
| **Database model** | MUTABLE (update/delete records freely)       | IMMUTABLE (append-only event log)              |
| **Event type**     | State changes extracted from replication log | Intent/business events designed explicitly     |
| **Log compaction** | ✓ Safe (newer event = full new state)        | ✗ Harder (events are semantic, not full state) |
| **Adoption**       | Add to existing system with minimal changes  | Significant architectural change               |
| **Who knows**      | Application may not know CDC is occurring    | Application logic BUILT on events              |

```
CDC: "I updated record X to value Y"
Event Sourcing: "User added item to cart" (intent, not mechanics)

CDC → log compaction: new update event = full new state for key → safe to discard old
Event Sourcing → log compaction: hard, because events are semantic,
   later events don't override earlier ones → need full history

Both: can REPLAY event log to reconstruct current state
Both: well-suited with log-based message brokers
```

---

## 4. State, Streams, and Immutability

### State = Result of Past Events

```
Current state: RESULT OF EVENTS that mutated it over time
   List of available seats = result of all reservation events
   Account balance = result of all credits and debits
   Response time graph = aggregation of individual response times

→ State CHANGES, but there's always a SEQUENCE OF EVENTS causing changes
→ Even "done and undone" actions: BOTH events still happened
   (undo = new event, not deletion of old event)
```

### Mutable State vs Immutable Events

```
MUTABLE DATABASE (traditional):
   Current state: optimized for reads, queries
   History: LOST (overwritten by updates)

IMMUTABLE EVENT LOG:
   Full history: preserved forever
   Current state: DERIVED from events (materialized view)

Same data, different perspectives:
   "Account balance = 500" (mutable state view)
   [+1000, -200, -300] (immutable event sequence → sum = 500)

ADVANTAGES OF IMMUTABILITY:
   1. Debugging: can inspect what happened at any point
   2. Audit trail: full history for compliance
   3. Replay: reprocess history with different code → different derived data
   4. Multiple views: different materialized views from same event log
   5. Undo: cancellation event is a new event (not delete old event)
```

### Advantages of Event-Driven Architecture

```
1. MULTIPLE DERIVED DATA SYSTEMS:
   Same event log → search index + warehouse + cache + ML features
   All derived from SAME EVENTS → consistent by construction

2. DEBUGGING:
   Bug in derived view code?
   → Fix code → reprocess from event log → correct derived view
   (vs: traditional DB: corrupt mutable state → hard to fix)

3. CONCURRENCY:
   Each consumer processes events in order → no locking needed

4. TEMPORAL QUERIES:
   "What was account balance at 2pm yesterday?"
   → Replay events until 2pm → compute balance at that point
```

### Limitations of Immutability

```
PRACTICAL CHALLENGES:

1. DISK SPACE:
   High-churn data: millions of updates/day → huge log
   Financial trading: data after use = not needed → MUST DELETE
   → Not everything can be kept forever

2. PRIVACY / GDPR (Right to be Forgotten):
   Must delete personal data → hard with immutable log!

   SOLUTIONS:
   a) CRYPTO-SHREDDING:
      Encrypt personal data with per-user key
      "Delete" = delete encryption key → data unreadable
      (But data still there, just unreadable)

   b) PER-USER LOG:
      Store each user's events in separate log
      "Delete" = delete that user's entire log segment

3. ADMINISTRATIVE ERRORS:
   "Delete all messages to user X" (they complained about spam)
   Truly unwanted data: offensive content, dangerous information
   → Need ability to DELETE from immutable log (exceptional cases)

4. LOG COMPACTION LIMITS:
   CDC: safe to compact (newer event = full new state)
   Event sourcing: harder (need full history to reconstruct state)
```

---

## Tóm tắt phần này

```
KEEPING SYSTEMS IN SYNC:
   Dual writes: race conditions + partial failure → inconsistency
   Better: single leader (database) → CDC stream → followers

CDC:
   Observe ALL changes to database → stream to derived systems
   Same order as database → permanent consistency
   Tools: Debezium, Kafka Connect, Maxwell, pgcapture
   Initial snapshot + CDC changes from snapshot offset
   Log compaction: compacted log = current database state
   Outbox pattern: decouple internal schema from CDC schema

CDC vs Event Sourcing:
   CDC: low-level, mutable DB, easy to add retroactively
   Event Sourcing: application-level, immutable, big architectural change

STATE, STREAMS, IMMUTABILITY:
   State = result of events (events = source of truth)
   Immutable events: debugging, audit, replay, multiple views
   Limitations: disk space, GDPR (crypto-shredding), admin errors
```
