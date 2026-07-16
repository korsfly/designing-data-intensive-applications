# Summary & Key Concepts — Chapter 5

---

## 1. Tổng hợp toàn chương

Chương 5 trả lời câu hỏi: **Khi code và data thay đổi theo thời gian, làm sao đảm bảo cả code cũ lẫn mới có thể cùng tồn tại và giao tiếp được?**

### Hai khái niệm trung tâm

```
BACKWARD COMPATIBILITY: Code MỚI đọc được data của code CŨ
                        (thường DỄ — biết format cũ, có thể handle explicitly)

FORWARD COMPATIBILITY:  Code CŨ đọc được data của code MỚI
                        (thường KHÓ — cần ignore additions của new code)

CÁ BIỆT KHĨÓ: Code cũ đọc data mới, rồi UPDATE rồi WRITE BACK
   → Nếu không cẩn thận → MẤT FIELDS mà code cũ không biết (Figure 5-1)
   → Giải pháp: old code KEEP INTACT unknown fields
```

### Tại sao cần cả 2?

```
ROLLING UPGRADES:
   Nodes được update TỪNG CÁI MỘT (không phải tất cả cùng lúc)
   → Trong lúc upgrade: old nodes ↔ new nodes giao tiếp
   → CẦN CẢ BACKWARD VÀ FORWARD COMPATIBILITY

CLIENT APPS:
   User có thể KHÔNG cài update → client cũ vẫn gọi server mới
   → CẦN FORWARD COMPATIBILITY trên response
```

---

## 2. Luồng kiến thức xuyên suốt chương

```
WHY? Evolvability → Rolling upgrades → Old/New code coexist
       │
       ▼
HOW? Encoding Formats
   Language-specific → BAD
   JSON/XML/CSV → OK nhưng vague, verbose
   Binary: Protobuf (field tags) / Avro (writer+reader schema) → COMPACT + EXPLICIT
       │
       ▼
WHERE? Modes of Dataflow
   Databases: "data outlives code", schema migration, compaction
   Services (REST/RPC): OpenAPI/gRPC, service discovery, load balancing, service mesh
   Workflows: durable execution (Temporal), exactly-once semantics
   Messages: message brokers (Kafka, RabbitMQ), pub-sub, actor model
```

---

## 3. Bảng thuật ngữ (Glossary)

### Encoding

| Thuật ngữ                                               | Định nghĩa                                                                                           |
| ------------------------------------------------------- | ---------------------------------------------------------------------------------------------------- |
| **Encoding / Serialization / Marshaling**               | In-memory → byte sequence                                                                            |
| **Decoding / Parsing / Deserialization / Unmarshaling** | Byte sequence → in-memory                                                                            |
| **Language-specific format**                            | Java Serializable, Python pickle, Ruby Marshal — KHÔNG nên dùng cho persistent/cross-service         |
| **Zero-copy format**                                    | Cap'n Proto, FlatBuffers — dùng được ở runtime và disk/network mà không cần conversion               |
| **MessagePack**                                         | Binary encoding cho JSON — compact hơn textual nhưng không nhiều                                     |
| **Protocol Buffers (Protobuf)**                         | Binary schema-driven encoding từ Google — dùng field tags                                            |
| **Apache Thrift**                                       | Binary schema-driven encoding từ Facebook — tương tự Protobuf                                        |
| **Apache Avro**                                         | Binary schema-driven encoding — không có field tags, dùng writer/reader schema resolution            |
| **Field tag**                                           | Số nguyên alias cho field name trong Protobuf — KHÔNG được thay đổi                                  |
| **Variable-length integer**                             | Encode số với số bytes tùy theo giá trị (nhỏ → ít bytes hơn)                                         |
| **repeated modifier**                                   | Protobuf equivalent của list/array                                                                   |
| **Writer's schema**                                     | Schema được compile vào app khi encode (Avro)                                                        |
| **Reader's schema**                                     | Schema mà app đọc mong đợi (Avro) — có thể khác writer's schema                                      |
| **Schema resolution**                                   | Avro process của việc match writer và reader schemas, fill defaults, skip unknowns                   |
| **Schema registry**                                     | Service lưu trữ valid schema versions và check compatibility (Confluent, Apicurio)                   |
| **Avro IDL**                                            | Avro schema language cho human editing                                                               |
| **Object container file**                               | Avro file format với schema ở đầu file                                                               |
| **Schema evolution**                                    | Thay đổi schema theo thời gian trong khi duy trì compatibility                                       |
| **ASN.1**                                               | Schema definition language từ 1984 — dùng tag numbers như Protobuf, vẫn dùng trong SSL certs (X.509) |
| **BSON, CBOR, UBJSON**                                  | Các binary encoding khác cho JSON                                                                    |
| **OpenAPI / Swagger**                                   | IDL cho RESTful JSON web services                                                                    |
| **AsyncAPI**                                            | OpenAPI equivalent cho messaging systems                                                             |
| **JSON Schema**                                         | Schema language cho JSON — powerful nhưng complex                                                    |

### Dataflow và Services

| Thuật ngữ                                  | Định nghĩa                                                                                   |
| ------------------------------------------ | -------------------------------------------------------------------------------------------- |
| **Data outlives code**                     | Nguyên tắc: data sống lâu hơn code — cần xem xét khi migrate schema                          |
| **Archival storage**                       | Snapshot của database cho backup/analytics — thường encode với latest schema                 |
| **Service / Web service**                  | API được expose qua network bởi server                                                       |
| **REST (Representational State Transfer)** | Design philosophy cho web services dựa trên HTTP principles                                  |
| **RESTful API**                            | API được design theo REST principles                                                         |
| **RPC (Remote Procedure Call)**            | Cố làm network request trông giống local function call                                       |
| **Location transparency**                  | Ý tưởng của RPC: ẩn đi sự khác biệt giữa local và remote call — FLAWED                       |
| **gRPC**                                   | Google's RPC framework dùng Protocol Buffers                                                 |
| **Service-oriented architecture (SOA)**    | Kiến trúc chia app thành services giao tiếp qua network                                      |
| **Idempotency**                            | Property của operation: thực hiện nhiều lần cho cùng kết quả như 1 lần                       |
| **IDL (Interface Definition Language)**    | Ngôn ngữ để define service API (OpenAPI, Protobuf IDL)                                       |
| **Service framework**                      | Spring Boot, FastAPI, gRPC — handle routing/metrics/auth, developer focus vào business logic |
| **Hardware load balancer**                 | Specialized equipment phân phối traffic                                                      |
| **Software load balancer**                 | NGINX, HAProxy — application làm nhiệm vụ load balancing                                     |
| **Service discovery**                      | Mechanism để client tìm địa chỉ của service (etcd, ZooKeeper)                                |
| **Heartbeat**                              | Tín hiệu định kỳ service gửi đến discovery system để signal vẫn available                    |
| **Service mesh**                           | Sidecar-based LB+service discovery (Istio, Linkerd) — handles encryption, observability      |
| **Sidecar**                                | Proxy container chạy cùng với application container để handle cross-cutting concerns         |
| **Semi-synchronous replication**           | 1 follower synchronous + các follower khác asynchronous                                      |

### Workflows và Events

| Thuật ngữ                              | Định nghĩa                                                                       |
| -------------------------------------- | -------------------------------------------------------------------------------- |
| **Workflow**                           | Graph of tasks/steps trong service-based architecture                            |
| **Task / Activity / Durable function** | Tên gọi khác nhau cho mỗi step trong workflow                                    |
| **Orchestrator**                       | Component schedule tasks để execute                                              |
| **Executor**                           | Component thực sự execute tasks                                                  |
| **Durable execution**                  | Framework cung cấp exactly-once semantics qua WAL logging của RPCs/state changes |
| **Temporal**                           | Durable execution framework phổ biến                                             |
| **Restate**                            | Durable execution framework khác                                                 |
| **BPEL**                               | Business Process Execution Language — markup language cho workflows              |
| **BPMN**                               | Business Process Model and Notation — graphical notation                         |
| **Event / Message**                    | Request trong event-driven architecture                                          |
| **Message broker**                     | Intermediary lưu messages tạm thời (Kafka, RabbitMQ, ActiveMQ, NATS)             |
| **Queue**                              | Point-to-point: 1 producer → 1 trong số nhiều consumers                          |
| **Topic**                              | Publish-subscribe: 1 publisher → TẤT CẢ subscribers                              |
| **Producer / Publisher**               | Bên gửi message                                                                  |
| **Consumer / Subscriber**              | Bên nhận message                                                                 |
| **Actor model**                        | Programming model: actors có local state, communicate qua async messages         |
| **Akka**                               | Distributed actor framework (Scala/Java)                                         |
| **Orleans**                            | Distributed actor framework (Microsoft/.NET)                                     |
| **Erlang/OTP**                         | Language + framework với actor model built-in                                    |
| **Distributed actor framework**        | Actor model + message broker integrated (Akka, Orleans, Erlang/OTP)              |

---

## 4. Mindmap khái niệm

```
                     ENCODING AND EVOLUTION
                              │
          ┌───────────────────┼──────────────────────┐
          │                   │                      │
    WHY NEEDED          ENCODING FORMATS        MODES OF DATAFLOW
          │                   │                      │
   Rolling Upgrade      ┌─────┴──────┐         ┌─────┴──────┐
   Old/New code         │            │          │     │      │
   coexist           Textual      Binary     Database Service Message
          │           │  │         │  │          │     │      │
   Backward:        JSON XML     Protobuf   Avro  │   REST   Broker
   new code reads    │   │       (tags)    (w+r  │   RPC    │  Actor
   old data          │   │         │       schema) │   │      │
                     │   │       Schema │  │   Data  IDL   Kafka
   Forward:        JSON CSV      evolu  │  │  outlives (Open  RabbitMQ
   old code reads  Schema         tion  │  │   code    API,   │
   new data                             │  │           gRPC)  Pub-Sub
   (harder!)                     Schema│  │               Service
                                Registry│  Archival     Discovery
                                (Confluent) Storage    (etcd,ZK)
                                                      Service
                                                       Mesh
                                                    (Istio,
                                                    Linkerd)
                                          Workflows
                                         /         \
                                    Durable     BPMN
                                    Execution   (Camunda)
                                  (Temporal,Restate)
                                  Exactly-once
                                  semantics
```

---

## 5. Câu hỏi tự kiểm tra (Self-Check Questions)

1. Phân biệt **Backward Compatibility** và **Forward Compatibility**. Cái nào thường dễ đạt được hơn? Tại sao?

2. Giải thích vấn đề trong **Figure 5-1**: Code cũ đọc data mới, update, rồi write back. Data nào có thể bị mất?

3. Tại sao **language-specific encoding** (Java Serializable, Python pickle) là bad idea cho production systems? Liệt kê 4 vấn đề.

4. Liệt kê 4 vấn đề của **JSON/XML/CSV** về mặt kỹ thuật. Khi nào thì chúng vẫn là lựa chọn phù hợp?

5. Giải thích vì sao **binary JSON** (MessagePack, BSON,...) chỉ tiết kiệm được ít space so với textual JSON.

6. So sánh size encoding của cùng 1 record: JSON (text) / MessagePack / Protocol Buffers / Avro. Tại sao Protobuf compact hơn MessagePack?

7. Trong **Protocol Buffers**, tại sao **field tags** là critical? Quy tắc gì khi thêm/xóa field?

8. Giải thích cơ chế **forward compatibility** của Protobuf khi code cũ gặp field tag không biết.

9. Avro khác Protobuf thế nào: **không có field tags**. Điều này có nghĩa gì cho encoding và schema evolution?

10. Giải thích **writer's schema** và **reader's schema** trong Avro. Avro resolve sự khác nhau như thế nào?

11. Trong Avro, khi nào CÓ THỂ thêm/xóa field mà vẫn maintain compatibility? Tại sao Avro không cho phép `null` là default mặc nhiên?

12. Giải thích **3 cách Avro reader biết writer's schema** tùy theo context (large file, database, network).

13. Tại sao **Avro tốt hơn Protobuf cho dynamically generated schemas**? Cho ví dụ cụ thể với relational database dump.

14. Liệt kê **4 merits của schema-based binary encoding** (Protobuf, Avro) so với JSON/XML.

15. Giải thích **"data outlives code"**. Tại sao database thường DEFER schema migration?

16. Giải thích **6 lý do** tại sao network request KHÔNG giống local function call (tại sao RPC "location transparency" là flawed).

17. So sánh **4 giải pháp service discovery**: DNS / Software LB / Service discovery systems / Service mesh.

18. Trong **durable execution** (Temporal), giải thích cơ chế exactly-once. Tại sao code changes "brittle"? Tại sao cần determinism?

19. Liệt kê **5 lợi ích của message broker** so với direct RPC. Phân biệt **queue** và **topic**.

20. Tại sao **location transparency** hoạt động TỐT HƠN trong **actor model** so với RPC?

---

## 6. Liên kết tới các chương khác

| Chapter 5 đề cập                               | Mở rộng ở                                            |
| ---------------------------------------------- | ---------------------------------------------------- |
| Evolvability, rolling upgrades                 | **Chapter 2** — Maintainability                      |
| Schema-on-read vs Schema-on-write              | **Chapter 3** — Data Models                          |
| LSM compaction rewrite data                    | **Chapter 4** — Storage and Retrieval                |
| WAL (Write-Ahead Log)                          | **Chapter 4** và **Chapter 8** — Transactions        |
| Request routing, sharding                      | **Chapter 7** — Sharding                             |
| Transactions (serialization nghĩa khác)        | **Chapter 8** — Transactions                         |
| Network failures, timeouts, unreliable clocks  | **Chapter 9** — The Trouble with Distributed Systems |
| Determinism và distributed systems             | **Chapter 9**                                        |
| Consensus, etcd, ZooKeeper                     | **Chapter 10** — Consistency and Consensus           |
| Change data capture (CDC), logical replication | **Chapter 12** — Stream Processing                   |
| Event sourcing, message brokers                | **Chapter 12** — Stream Processing                   |
| Archival storage, Parquet                      | **Chapter 11** — Batch Processing                    |

---

## 7. Trích dẫn mở đầu chương (Epigraph)

> _"Everything changes and nothing stands still."_
> — **Heraclitus of Ephesus**, as quoted by Plato in Cratylus (360 BCE)

→ Một triết lý 2400 năm tuổi vẫn hoàn toàn áp dụng được: ứng dụng luôn thay đổi, requirements thay đổi, schemas thay đổi. Hệ thống tốt phải được thiết kế để **thích nghi với thay đổi** mà không gây gián đoạn — đây là bản chất của Encoding and Evolution.

---

## 8. Bảng so sánh Encoding Formats

| Format               | Size (record mẫu) | Schema?                | Human-readable? | Field identification                  | Schema evolution                |
| -------------------- | ----------------- | ---------------------- | --------------- | ------------------------------------- | ------------------------------- |
| **JSON (text)**      | 81 bytes          | Optional (JSON Schema) | ✓               | Field names trong data                | Manual, phụ thuộc cách dùng     |
| **XML**              | Lớn hơn           | Optional (XML Schema)  | ✓               | Tag names trong data                  | Manual, phức tạp                |
| **CSV**              | Vừa               | ✗                      | ✓               | Position (theo cột)                   | Manual                          |
| **MessagePack**      | 66 bytes          | ✗                      | ✗               | Field names (vẫn trong data)          | Như JSON                        |
| **Protocol Buffers** | 33 bytes          | ✓ (required)           | ✗               | Field TAGS (numbers)                  | Tag-based (stable tags)         |
| **Avro**             | 32 bytes          | ✓ (required)           | ✗               | Field names (resolved at decode time) | Writer+Reader schema resolution |
