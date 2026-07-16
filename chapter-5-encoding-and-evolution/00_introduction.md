# DDIA 2nd Edition — Chapter 5: Encoding and Evolution

> **Sách:** Designing Data-Intensive Applications, 2nd Edition  
> **Tác giả:** Martin Kleppmann & Chris Riccomini  
> **Chương:** 5 — Encoding and Evolution

---

## Mục tiêu chương

Ứng dụng **luôn thay đổi** theo thời gian. Khi data format hoặc schema thay đổi, code cũng cần thay đổi — nhưng trong hệ thống lớn, code **không thể thay đổi đồng thời** trên tất cả nodes (rolling upgrade, client-side apps). Chương 5 tập trung vào câu hỏi: **làm sao encode data để cả code cũ và mới có thể cùng tồn tại?**

> Epigraph mở đầu chương:
> _"Everything changes and nothing stands still."_
> — Heraclitus of Ephesus, as quoted by Plato in Cratylus (360 BCE)

---

## Hai Khái niệm Trung tâm

```
BACKWARD COMPATIBILITY:  Code MỚI có thể đọc data viết bởi code CŨ
FORWARD COMPATIBILITY:   Code CŨ có thể đọc data viết bởi code MỚI
```

---

## Cấu trúc tài liệu này

| File                                    | Nội dung                                                                        |
| --------------------------------------- | ------------------------------------------------------------------------------- |
| `01_Encoding_Formats.md`                | Language-specific, JSON/XML/CSV, MessagePack, Protocol Buffers, Avro            |
| `02_Schema_Evolution.md`                | Protobuf field tags, Avro writer/reader schema, merits of schema-based encoding |
| `03_Dataflow_Databases_and_Services.md` | Data outlives code, REST/RPC, vấn đề với RPC, load balancing, service mesh      |
| `04_Workflows_and_Events.md`            | Durable execution (Temporal), message brokers, actor model                      |
| `05_Summary_and_Key_Concepts.md`        | Tổng hợp toàn chương, glossary, mindmap, câu hỏi ôn tập                         |

---

## Big Picture

```
┌──────────────────────────────────────────────────────────┐
│                ENCODING & EVOLUTION                      │
│                                                           │
│  WHY: Rolling upgrades → old/new code chạy đồng thời     │
│  GOAL: Backward + Forward Compatibility                   │
│                                                           │
│  ENCODING FORMATS:                                        │
│    Language-specific → BAD (vendor lock-in, security)    │
│    JSON/XML/CSV → OK nhưng vague về datatypes            │
│    Binary (Protobuf, Avro) → COMPACT, explicit schema    │
│                                                           │
│  MODES OF DATAFLOW:                                       │
│    Databases (data outlives code)                        │
│    Services: REST/RPC (client-server)                    │
│    Workflows: Durable Execution (Temporal, Restate)      │
│    Messages: Brokers (Kafka, RabbitMQ) + Actor Model     │
└──────────────────────────────────────────────────────────┘
```

---

## Các chương liên quan

- **Chapter 2:** Evolvability (nền tảng tư duy)
- **Chapter 3:** Schema-on-read vs Schema-on-write
- **Chapter 4:** LSM-tree rewrite data trong compaction
- **Chapter 7:** Sharding, request routing, service discovery
- **Chapter 8:** Transactions (serialization — nghĩa KHÁC với encoding)
- **Chapter 9:** Network failures, timeouts, unreliable clocks
- **Chapter 10:** Consensus (coordination services: etcd, ZooKeeper)
- **Chapter 12:** Event sourcing, stream processing, change data capture
