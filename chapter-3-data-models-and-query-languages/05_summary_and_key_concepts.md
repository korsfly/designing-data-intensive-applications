# Summary & Key Concepts — Chapter 3

---

## 1. Tổng hợp toàn chương

Chương 3 khám phá **data models** như một quyết định thiết kế CỐT LÕI — ảnh hưởng sâu sắc đến cách viết code và cách suy nghĩ về vấn đề.

```
┌─────────────────────────────────────────────────────────┐
│              DATA MODELS TRONG CHƯƠNG 3                │
│                                                          │
│  RELATIONAL MODEL (>50 năm tuổi, VẪN quan trọng)        │
│     ├─ Tốt cho: data warehousing, business analytics    │
│     └─ Star/Snowflake schemas, SQL ubiquitous            │
│                                                          │
│  DOCUMENT MODEL                                          │
│     └─ Tốt cho: self-contained JSON, ÍT relationship     │
│                  giữa documents                          │
│                                                          │
│  GRAPH MODELS                                            │
│     └─ Tốt cho: MỌI THỨ có thể liên quan đến MỌI THỨ,    │
│                  cần multi-hop traversal queries          │
│        (Cypher, SPARQL, Datalog — recursive queries)     │
│                                                          │
│  DATAFRAMES                                              │
│     └─ Cầu nối giữa database và multidimensional arrays  │
│        cho ML, statistical analysis, scientific computing│
└─────────────────────────────────────────────────────────┘
```

### Insight quan trọng: Một Model có thể Emulate Model Khác

```
VÍ DỤ: Graph data CÓ THỂ biểu diễn trong relational database
   NHƯNG kết quả có thể VỤNG VỀ
   (như đã thấy: Cypher 4 dòng vs SQL recursive CTE 31 dòng)

→ Database chuyên biệt được phát triển cho TỪNG data model
  NHƯNG có XU HƯỚNG database "lấn sân" hỗ trợ model khác:
     Relational DB → thêm JSON column support
     Document DB → thêm relational-like joins
     SQL → dần cải thiện support cho graph data
```

### Event Sourcing — Một Model riêng biệt

```
Event Sourcing: biểu diễn data dưới dạng APPEND-ONLY LOG of
                immutable events

Lợi thế: TỐT cho VIẾT (append-only — Chapter 4)
Để hỗ trợ query hiệu quả: cần translate qua CQRS
                          → read-optimized materialized views
```

### Điểm chung của Nonrelational Models

```
Document/Graph models THƯỜNG:
   KHÔNG enforce schema cho data lưu trữ
   → Dễ adapt application khi requirements thay đổi

NHƯNG: Application VẪN ASSUME data có CẤU TRÚC NHẤT ĐỊNH
       → Chỉ là vấn đề: schema EXPLICIT (enforce khi viết)
                        hay IMPLICIT (assume khi đọc)
```

---

## 2. Bảng thuật ngữ (Glossary)

### Relational & Document

| Thuật ngữ                           | Định nghĩa                                                                 |
| ----------------------------------- | -------------------------------------------------------------------------- |
| **Impedance mismatch**              | Sự không khớp giữa object model trong code và relational table model       |
| **ORM** (Object-Relational Mapping) | Framework giảm boilerplate cho translation layer (ActiveRecord, Hibernate) |
| **N+1 Query Problem**               | Vấn đề performance khi ORM thực hiện N+1 query thay vì 1 join              |
| **Normalization**                   | Lưu data "có nghĩa với người" CHỈ MỘT NƠI, dùng ID để reference            |
| **Denormalization**                 | Lặp lại data ở nhiều nơi để tránh cần join khi đọc                         |
| **Shredding**                       | Chia document-like structure thành nhiều bảng relational                   |
| **One-to-few**                      | One-to-many relationship với SỐ LƯỢNG NHỎ items                            |
| **Hydrating IDs**                   | Resolve ID thành thông tin human-readable, thực hiện ở application code    |
| **Star Schema**                     | Fact table ở giữa, dimension tables xung quanh                             |
| **Snowflake Schema**                | Star schema với dimension tables chia nhỏ thành sub-dimensions             |
| **One Big Table (OBT)**             | Gộp toàn bộ dimension vào fact table, bỏ hẳn dimension tables              |
| **Schema-on-read**                  | Cấu trúc data implicit, chỉ interpret khi đọc (document DB)                |
| **Schema-on-write**                 | Cấu trúc data explicit, enforce khi viết (relational DB)                   |
| **Data locality**                   | Lợi thế performance khi data liên quan được lưu GẦN NHAU                   |

### Graph Models

| Thuật ngữ                                | Định nghĩa                                              |
| ---------------------------------------- | ------------------------------------------------------- |
| **Vertex (Node)**                        | Đối tượng trong graph                                   |
| **Edge (Relationship/Arc)**              | Liên kết giữa 2 vertex                                  |
| **Adjacency List**                       | Mỗi vertex lưu ID các vertex láng giềng                 |
| **Adjacency Matrix**                     | Ma trận 2D biểu diễn graph (0/1)                        |
| **Property Graph**                       | Model với vertex/edge có label + properties             |
| **Triple Store**                         | Model lưu data dạng (subject, predicate, object)        |
| **Cypher**                               | Query language cho property graph (Neo4j)               |
| **SPARQL**                               | Query language cho triple store/RDF                     |
| **Datalog**                              | Query language relational, mạnh về recursive query      |
| **RDF** (Resource Description Framework) | Data model cho Semantic Web                             |
| **Turtle**                               | Format encode RDF data (subset của Notation3)           |
| **Hypergraph**                           | Graph mà edge có thể liên kết NHIỀU HƠN 2 vertex        |
| **GQL**                                  | ISO standard cho graph query language (dựa trên Cypher) |

### GraphQL & Event Sourcing

| Thuật ngữ                                       | Định nghĩa                                                       |
| ----------------------------------------------- | ---------------------------------------------------------------- |
| **GraphQL**                                     | Query language cho client-server OLTP, client chỉ định field cần |
| **Event Sourcing**                              | Dùng events immutable làm source of truth                        |
| **CQRS**                                        | Command Query Responsibility Segregation — tách read/write model |
| **Command**                                     | Request từ user, cần validate trước khi thành event              |
| **Materialized View / Projection / Read Model** | Representation derived từ event log                              |
| **Crypto-shredding**                            | Mã hóa data rồi xóa key để "xóa" data (cho GDPR compliance)      |

### DataFrames

| Thuật ngữ            | Định nghĩa                                               |
| -------------------- | -------------------------------------------------------- |
| **DataFrame**        | Data model giống table, dùng cho data science/ML         |
| **Merge**            | Tên gọi của "join" trong DataFrame                       |
| **One-hot encoding** | Kỹ thuật chuyển categorical data thành numeric columns   |
| **Sparse matrix**    | Ma trận mà PHẦN LỚN giá trị là 0/rỗng                    |
| **Array Database**   | Database chuyên biệt cho multidimensional array (TileDB) |

---

## 3. Mindmap khái niệm

```
                    DATA MODELS & QUERY LANGUAGES
                              │
       ┌──────────────┬───────┴───────┬───────────────┐
       │              │               │               │
  RELATIONAL      DOCUMENT          GRAPH          DATAFRAME
       │              │               │               │
   ┌───┴───┐     ┌────┴────┐    ┌─────┴──────┐   ┌─────┴─────┐
   │       │     │         │    │            │   │           │
 Tables  JOIN  JSON    Schema  Property    Triple Matrix   One-hot
   │       │   self-   -on-    Graph       Store  for ML  encoding
 Star/  Many-  contain  read     │           │      │        │
 Snow-  to-              │    Cypher      SPARQL  Sparse   Linear
 flake  Many              │               /RDF    arrays   Algebra
   │            Object-Relational      Datalog
 Normali  Mismatch (ORM)              (Recursive)
 -zation/                                  │
 Denorma-                          ┌───────┴────────┐
 -lization                    GraphQL          Event Sourcing
                          (Client-Server,            │
                           NO recursion)        CQRS + Materialized
                                                      Views
```

---

## 4. Câu hỏi tự kiểm tra (Self-Check Questions)

1. Giải thích **Object-Relational Mismatch (Impedance Mismatch)**. Nó ảnh hưởng thế nào đến quyết định dùng ORM?
2. Mô tả **N+1 Query Problem**. Tại sao nó xảy ra với ORM?
3. So sánh ưu/nhược điểm giữa lưu **ID (normalized)** và lưu **text trực tiếp (denormalized)**.
4. Giải thích case study **hydrating IDs** trong materialized timeline của X (Twitter). Tại sao họ KHÔNG denormalize like count vào timeline?
5. Phân biệt **Star Schema**, **Snowflake Schema**, và **One Big Table (OBT)**. Khi nào denormalize mạnh trong analytics là OK?
6. Khi nào nên chọn **Document Model** thay vì **Relational Model**? Cho ví dụ cụ thể (reorderable list).
7. Phân biệt **Schema-on-read** và **Schema-on-write**. So sánh với static/dynamic type checking.
8. Giải thích **Property Graph Model**. Vertex và Edge có những thành phần gì?
9. So sánh độ phức tạp giữa query **Cypher** và **SQL recursive CTE** cho cùng 1 bài toán graph traversal.
10. Phân biệt **Property Graph** và **Triple Store**. Cho ví dụ cụ thể về cách biểu diễn `(lucy, marriedTo, alain)`.
11. Giải thích cách **Datalog rules** hoạt động — tại sao nó mạnh cho recursive query?
12. Tại sao **GraphQL** KHÔNG cho phép recursive query, khác với Cypher/SPARQL/Datalog?
13. Giải thích **Event Sourcing** và **CQRS**. Liệt kê 3 ưu điểm và 2 nhược điểm.
14. Tại sao **immutability** trong Event Sourcing tạo vấn đề với **GDPR**? Giải pháp nào được đề xuất?
15. **DataFrame** khác **Relational Table** như thế nào về cách manipulate data?
16. Giải thích kỹ thuật **One-hot encoding**. Tại sao cần nó khi chuyển data sang matrix cho ML?

---

## 5. Liên kết tới các chương khác

| Chapter 3 đề cập                                   | Mở rộng ở                                      |
| -------------------------------------------------- | ---------------------------------------------- |
| Storage engine cho các data model này              | **Chapter 4** — Storage and Retrieval          |
| JSON, XML schema evolution                         | **Chapter 5** — Encoding and Evolution         |
| REST/RPC (GraphQL tooling cần convert sang)        | **Chapter 5**                                  |
| Event Sourcing cần xử lý đúng thứ tự (distributed) | **Chapter 10** — Consistency and Consensus     |
| State machine replication                          | **Chapter 10** — "Using shared logs"           |
| DataFrames trong Spark/Flink                       | **Chapter 11** — Batch Processing              |
| Message brokers (Kafka) cho Event Log              | **Chapter 12** — Stream Processing             |
| Full-text search, vector search                    | **Chapter 4** — "Full-Text Search"             |
| GDPR, right to be forgotten                        | **Chapter 1** — Data Systems, Law, and Society |

---

## 6. Các Data Model "Bỏ ngỏ" (Chưa thảo luận sâu)

Chương 3 đề cập NGẮN một vài data model khác không thuộc phạm vi chính:

| Data Model                          | Use Case                         | Database/Công nghệ                        |
| ----------------------------------- | -------------------------------- | ----------------------------------------- |
| **Genome Sequence Data**            | Sequence similarity search (DNA) | GenBank                                   |
| **Double-Entry Ledger**             | Hệ thống tài chính, kế toán      | TigerBeetle                               |
| **Distributed Ledger / Blockchain** | Cryptocurrency, value transfer   | Blockchain systems                        |
| **Full-Text Search**                | Information retrieval            | Search indexes, vector search (Chapter 4) |

---

## 7. Trích dẫn mở đầu chương (Epigraph)

> _"The limits of my language mean the limits of my world."_
> — **Ludwig Wittgenstein**, _Tractatus Logico-Philosophicus_ (1922)

→ Phản ánh chính xác **luận điểm trung tâm** của chương: data model bạn chọn KHÔNG CHỈ là chi tiết kỹ thuật — nó **GIỚI HẠN (hoặc MỞ RỘNG)** cách bạn có thể **suy nghĩ và biểu diễn** vấn đề đang giải quyết.
