# DDIA 2nd Edition — Chapter 12: Stream Processing

> **Sách:** Designing Data-Intensive Applications, 2nd Edition  
> **Tác giả:** Martin Kleppmann & Chris Riccomini  
> **Chương:** 12 — Stream Processing

---

## Mục tiêu chương

Chương 11 xử lý **bounded datasets** (batch). Chương 12 chuyển sang **unbounded streams** — dữ liệu đến liên tục, không bao giờ hoàn chỉnh. Stream processing là nền tảng của real-time analytics, fraud detection, event-driven architectures, và nhiều hơn nữa.

> Epigraph:
> _"A complex system that works is invariably found to have evolved from a simple system that works. The inverse proposition also appears to be true: A complex system designed from scratch never works and cannot be made to work."_
> — John Gall, Systemantics (1975)

---

## Cấu trúc tài liệu này

| File                              | Nội dung                                                                                                         |
| --------------------------------- | ---------------------------------------------------------------------------------------------------------------- |
| `01_Messaging_Systems.md`         | Direct messaging, Message Brokers (AMQP/JMS vs Log-based), Consumer offsets, Fan-out vs Load balancing           |
| `02_CDC_and_Event_Sourcing.md`    | Keeping systems in sync, Dual writes problems, Change Data Capture, Log compaction, CDC vs Event Sourcing        |
| `03_Processing_Streams.md`        | Stream operators, Windowing (tumbling/hopping/sliding/session), Time (event time vs processing time), Watermarks |
| `04_Joins_and_Fault_Tolerance.md` | Stream-stream join, Stream-table join, Table-table join, Exactly-once semantics, Idempotence                     |
| `05_Summary_and_Key_Concepts.md`  | Tổng hợp, glossary, mindmap, câu hỏi ôn tập                                                                      |

---

## Big Picture

```
┌──────────────────────────────────────────────────────────┐
│                  STREAM PROCESSING                        │
│                                                            │
│  TRANSMITTING EVENTS:                                      │
│  Direct messaging → Message Brokers (AMQP/JMS)           │
│  Log-based brokers (Kafka): durable, replayable           │
│                                                            │
│  DATABASES AND STREAMS:                                    │
│  CDC: capture DB changes → stream to derived systems      │
│  Event Sourcing: event log as source of truth             │
│  State = result of past events (immutability principle)   │
│                                                            │
│  PROCESSING STREAMS:                                       │
│  Operators: filter, transform, aggregate, join            │
│  Windowing: tumbling, hopping, sliding, session           │
│  Time: event time vs processing time; watermarks          │
│                                                            │
│  FAULT TOLERANCE:                                          │
│  At-least-once + idempotence = effectively-once           │
│  Exactly-once via atomic commit or determinism            │
└──────────────────────────────────────────────────────────┘
```

---

## Các chương liên quan

- **Chapter 3:** Event Sourcing, CQRS (nền tảng)
- **Chapter 5:** Message brokers, actor model (nền tảng)
- **Chapter 6:** Replication logs (CDC dựa trên logical replication)
- **Chapter 8:** Transactions, exactly-once semantics
- **Chapter 9:** Unreliable clocks (event time vs processing time)
- **Chapter 10:** Consensus, total order broadcast (stream ordering)
- **Chapter 11:** Batch processing (stream = unbounded batch)
