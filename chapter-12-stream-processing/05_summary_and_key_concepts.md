# Summary & Key Concepts — Chapter 12

---

## 1. Tổng hợp toàn chương

Chương 12 mở rộng từ batch processing (bounded) sang stream processing (unbounded), đồng thời giải quyết vấn đề keeping multiple derived data systems in sync.

### Luồng kiến thức

```
EVENTS everywhere:
   Web logs, user actions, metrics, sensor data, payments
        │
        ▼
TRANSMITTING EVENTS:
   Direct messaging (low latency, limited durability)
   Traditional brokers (AMQP/JMS): transient, per-message ack
   Log-based brokers (Kafka): durable, replayable, offset-based
        │
        ▼
DATABASES AS STREAMS:
   CDC: observe DB changes → stream to derived systems
   Event Sourcing: events as source of truth
   Immutability: state = result of events
   Log compaction: full state in compact log
        │
        ▼
PROCESSING STREAMS:
   Stateless (filter/map) vs Stateful (aggregate/join/window)
   Time: event time vs processing time
   Windows: tumbling/hopping/sliding/session
   Watermarks: heuristic for "window complete"
        │
        ▼
JOINS AND FAULT TOLERANCE:
   Stream-Stream (windowed), Stream-Table (CDC enrichment),
   Table-Table (materialized view maintenance)
   At-least-once + idempotence = effectively-once
   Micro-batching, checkpointing, Kafka EOS
```

---

## 2. Bảng thuật ngữ (Glossary)

### Messaging Systems

| Thuật ngữ                   | Định nghĩa                                                                               |
| --------------------------- | ---------------------------------------------------------------------------------------- |
| **Event**                   | Small, immutable object describing something that happened at a point in time            |
| **Topic / Stream**          | Named group of related events (like a file in batch)                                     |
| **Producer / Publisher**    | Generates and publishes events to topic                                                  |
| **Consumer / Subscriber**   | Reads and processes events from topic                                                    |
| **Message broker**          | Intermediary storing messages between producers and consumers                            |
| **AMQP**                    | Advanced Message Queuing Protocol (RabbitMQ, ActiveMQ)                                   |
| **JMS**                     | Java Message Service API                                                                 |
| **Load balancing**          | Each message → ONE consumer (work distribution)                                          |
| **Fan-out**                 | Each message → ALL consumers (broadcast)                                                 |
| **Acknowledgment**          | Consumer signals broker: message successfully processed                                  |
| **Dead Letter Queue (DLQ)** | Messages that failed too many times → moved here for inspection                          |
| **Direct messaging**        | Producer → consumer without broker (UDP, ZeroMQ, webhooks)                               |
| **Log-based broker**        | Kafka, Kinesis: append-only log, offset-based, durable, replayable                       |
| **Partition**               | Kafka's name for a shard of a topic                                                      |
| **Offset**                  | Position in partition log; consumer tracks own offset                                    |
| **Consumer group**          | Kafka: group of consumers sharing work; load balance within group, fan-out across groups |
| **Log compaction**          | Keep only latest value per key; discard old intermediate values                          |
| **Tiered storage**          | Older Kafka messages → object store (S3); hot messages → local disk                      |
| **Consumer lag**            | How far behind a consumer is from the latest offset                                      |

### CDC and Event Sourcing

| Thuật ngữ                     | Định nghĩa                                                                    |
| ----------------------------- | ----------------------------------------------------------------------------- |
| **CDC** (Change Data Capture) | Observe all DB changes → extract as stream for derived systems                |
| **Dual writes**               | App writes to multiple systems → race condition + partial failure risk        |
| **Debezium**                  | Open source CDC tool for many databases                                       |
| **Logical replication**       | Row-level change stream from database (basis for CDC)                         |
| **Initial snapshot**          | Full database dump at consistent point before CDC begins                      |
| **Outbox pattern**            | Write to internal tables + outbox table in same transaction; CDC reads outbox |
| **Event Sourcing**            | Application-level pattern: events as source of truth, state derived           |
| **CQRS**                      | Command Query Responsibility Segregation (Chapter 3)                          |
| **Crypto-shredding**          | Encrypt personal data with per-user key; "delete" = delete key                |
| **Append-only log**           | Immutable sequence of events; current state = result of events                |
| **Materialized view**         | Derived read-optimized view, maintained from event stream                     |
| **Percolator**                | Elasticsearch pattern: store queries, match against incoming documents        |

### Time and Windowing

| Thuật ngữ               | Định nghĩa                                                            |
| ----------------------- | --------------------------------------------------------------------- |
| **Event time**          | When event actually happened (timestamp in event payload)             |
| **Processing time**     | When event arrives at stream processor (processor's clock)            |
| **Ingestion time**      | When event was appended to log broker                                 |
| **Clock skew**          | Difference between clocks on different machines                       |
| **Late arrival**        | Event arrives AFTER its event-time window should have closed          |
| **Tumbling window**     | Fixed-size, non-overlapping; each event in exactly 1 window           |
| **Hopping window**      | Fixed-size, overlapping; each event in N windows                      |
| **Sliding window**      | Continuous; window moves with each new event                          |
| **Session window**      | Variable-size; grouped by user activity; gap threshold closes session |
| **Watermark**           | Claim: "all events with timestamp ≤ T received" → window complete     |
| **Perfect watermark**   | Exact knowledge of when all events arrive (rare)                      |
| **Heuristic watermark** | Estimate based on observed timestamps (common)                        |
| **Trigger**             | Apache Beam: define when to emit window results (early/late/on-time)  |

### Joins and Fault Tolerance

| Thuật ngữ              | Định nghĩa                                                          |
| ---------------------- | ------------------------------------------------------------------- |
| **Stream-stream join** | Windowed join of two event streams                                  |
| **Stream-table join**  | Enrich events with table data (local copy updated via CDC)          |
| **Table-table join**   | All tables as CDC streams → maintain materialized view              |
| **At-most-once**       | May lose events; never duplicates                                   |
| **At-least-once**      | Never loses events; may duplicate                                   |
| **Exactly-once**       | No loss, no duplicates — hard to achieve truly                      |
| **Effectively-once**   | At-least-once + idempotent = practically equivalent to exactly-once |
| **Idempotent**         | Operation applied multiple times = same result as applying once     |
| **Micro-batching**     | Divide stream into small time intervals; process each as mini-batch |
| **Checkpointing**      | Periodically save operator state snapshot; restore on failure       |
| **Chandy-Lamport**     | Distributed snapshot algorithm; used by Flink for checkpointing     |
| **Checkpoint barrier** | Special marker injected into event stream to trigger checkpoint     |
| **Kafka EOS**          | Exactly-once semantics via idempotent producer + transactions       |
| **RocksDB**            | Embedded key-value store used by Flink and Kafka Streams for state  |
| **Changelog topic**    | Kafka Streams: topic recording state changes for fault recovery     |
| **State backend**      | Where stream processor stores its operator state (Memory, RocksDB)  |

### Frameworks

| Thuật ngữ                      | Định nghĩa                                                           |
| ------------------------------ | -------------------------------------------------------------------- |
| **Apache Flink**               | Streaming-native framework; lowest latency; best event time support  |
| **Kafka Streams**              | Library (no separate cluster); Kafka-native EOS; local RocksDB state |
| **Spark Structured Streaming** | Unified batch+stream; micro-batching; DataFrame/SQL API              |
| **Apache Beam**                | Unified programming model; runners: Flink, Spark, Dataflow           |
| **ksqlDB**                     | SQL-based stream processing on Kafka                                 |
| **Materialize**                | Streaming SQL database; incremental view maintenance                 |

---

## 3. Mindmap khái niệm

```
                       STREAM PROCESSING
                              │
          ┌───────────────────┼──────────────────────┐
          │                   │                      │
   MESSAGING              DATABASES              PROCESSING
   SYSTEMS              AND STREAMS              STREAMS
          │                   │                      │
   ┌──────┴──────┐     ┌──────┴──────┐        ┌──────┴──────┐
   │             │     │             │        │             │
Traditional  Log-Based   CDC     Event    Operators    Time
(AMQP/JMS)   (Kafka)     │     Sourcing      │           │
   │             │     Debezium  immutable  Filter    Event time
Transient   Durable    Outbox    append    Map       vs Process
Per-msg ack Replayable  pattern  -only     Aggregate  time
DLQ        Offset-based  │     State=     Window         │
Fan-out    Log compact  CDC vs  result          │     Watermarks
Load bal.  Tiered store Event   of events  Tumbling  (heuristic)
           Consumer     Sourcing         Hopping   Late events
           group: LB             Dual   Sliding
           across      Crypto-  writes  Session
           partitions  shredding →race
                       per-user  cond.
                       log       problem
                              │
               ┌──────────────┼──────────────┐
               │              │              │
          STREAM-STREAM  STREAM-TABLE   TABLE-TABLE
          JOIN            JOIN           JOIN
               │              │              │
          Windowed index  Local cache   All tables
          one stream,    updated via   as CDC streams
          probe other    CDC           → materialized
                                         view
                              │
               FAULT TOLERANCE:
               ┌──────────────┼──────────────┐
               │              │              │
          Micro-batch    Checkpoint    Kafka EOS
          (Spark)        (Flink:       (idempotent
                        Chandy-Lamport) producer +
                        RocksDB state  transactions)
                              │
               At-least-once + idempotent
               = Effectively-once (practical)
```

---

## 4. Câu hỏi tự kiểm tra (Self-Check Questions)

1. Phân biệt **Stream Processing** và **Batch Processing**. Tại sao batch xử lý unbounded data bằng cách chia thành time intervals lại không đủ tốt?

2. Phân biệt **Load Balancing** và **Fan-out** trong message brokers. Kafka Consumer Groups kết hợp cả hai như thế nào?

3. Tại sao **Acknowledgments** cần thiết? Điều gì xảy ra nếu consumer crash TRƯỚC khi ack? Sau khi ack?

4. Giải thích **Dead Letter Queues (DLQ)**. Tại sao chúng quan trọng cho operational health?

5. So sánh **Traditional Message Brokers (AMQP/JMS)** và **Log-Based Brokers (Kafka)**: durability, replay, fan-out, slow consumers.

6. Giải thích **Consumer Offsets** trong Kafka. Tại sao offset-based tracking hiệu quả hơn per-message acknowledgment?

7. Mô tả **Log Compaction**. Tại sao compacted log = current database state?

8. Tại sao **Dual Writes** nguy hiểm (race condition + partial failure)? CDC giải quyết vấn đề này như thế nào?

9. Phân biệt **CDC** và **Event Sourcing** theo: database model, log compaction, adoption.

10. Tại sao **State = Result of Past Events**? Giải thích lợi ích của immutable event log (debugging, audit, replay, multiple views).

11. Giải thích vấn đề **GDPR với immutable logs**. Crypto-shredding là gì và tại sao nó chỉ là giải pháp gần đúng?

12. Phân biệt **Event Time** và **Processing Time**. Khi nào hai giá trị này khác nhau đáng kể?

13. Mô tả **4 loại Windows** (tumbling, hopping, sliding, session). Cho ví dụ use case cho mỗi loại.

14. Giải thích **Watermarks**. Tại sao không thể có perfect watermark trong hầu hết hệ thống thực tế? Trade-off giữa latency và accuracy?

15. Mô tả **3 loại Stream Joins** (stream-stream, stream-table, table-table). Ví dụ thực tế cho mỗi loại.

16. Phân biệt **At-most-once, At-least-once, Exactly-once, Effectively-once**.

17. Giải thích **Chandy-Lamport checkpointing** trong Flink. Checkpoint barrier là gì?

18. Tại sao **Kafka EOS** chỉ hoạt động cho Kafka-to-Kafka pipelines? Cần gì để exactly-once với external systems?

19. So sánh **Flink, Kafka Streams, Spark Structured Streaming** theo: latency, cluster requirement, exactly-once mechanism.

20. Tại sao **Kappa Architecture** (stream-only) tốt hơn **Lambda Architecture** (batch+stream)?

---

## 5. Liên kết tới các chương khác

| Chapter 12 đề cập                        | Liên quan đến                                        |
| ---------------------------------------- | ---------------------------------------------------- |
| Event Sourcing, CQRS, materialized views | **Chapter 3** — Data Models                          |
| Logical replication (CDC basis)          | **Chapter 6** — Replication                          |
| Message brokers, actor model             | **Chapter 5** — Dataflow Through Services            |
| Transactions, exactly-once, 2PC          | **Chapter 8** — Transactions                         |
| Unreliable clocks (event time problems)  | **Chapter 9** — The Trouble with Distributed Systems |
| Total order broadcast = shared log       | **Chapter 10** — Consistency and Consensus           |
| Batch vs stream, Kappa architecture      | **Chapter 11** — Batch Processing                    |
| Flink checkpointing (Chandy-Lamport)     | **Chapter 10** — Snapshot algorithms                 |
| Coordinator failure (Kafka EOS)          | **Chapter 8** — Distributed Transactions             |

---

## 6. Trích dẫn mở đầu chương

> _"A complex system that works is invariably found to have evolved from a simple system that works. The inverse proposition also appears to be true: A complex system designed from scratch never works and cannot be made to work."_
> — **John Gall**, Systemantics (1975)

→ Lời khuyên này đặc biệt đúng với stream processing: đừng cố xây dựng hệ thống phức tạp ngay từ đầu. Bắt đầu từ simple (direct messaging, batch ETL) → evolve thành complex (CDC, event sourcing, exactly-once streaming) khi thực sự cần thiết.

---

## 7. Bảng So sánh: Batch vs Stream

| Đặc điểm            | Batch (Chapter 11)                  | Stream (Chapter 12)                  |
| ------------------- | ----------------------------------- | ------------------------------------ |
| **Input**           | Bounded (finite)                    | Unbounded (infinite)                 |
| **Processing**      | Periodic jobs                       | Continuous                           |
| **Latency**         | Minutes to hours                    | Milliseconds to seconds              |
| **Fault tolerance** | Simple (retry from immutable input) | Complex (checkpoints, idempotency)   |
| **Output**          | Complete snapshot                   | Incremental updates                  |
| **Time semantics**  | Batch boundary is natural window    | Event time vs processing time        |
| **State**           | No state between jobs (usually)     | Stateful operators common            |
| **Frameworks**      | Spark, Flink (batch), MapReduce     | Flink, Kafka Streams, Spark SS       |
| **Use cases**       | ETL, ML training, reporting         | Fraud detection, real-time analytics |
| **Kappa**           | Replay as bounded stream            | Live stream                          |
