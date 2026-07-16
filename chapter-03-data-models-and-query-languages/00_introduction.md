# DDIA 2nd Edition — Chapter 3: Data Models and Query Languages

> **Sách:** Designing Data-Intensive Applications, 2nd Edition  
> **Tác giả:** Martin Kleppmann & Chris Riccomini  
> **Chương:** 3 — Data Models and Query Languages

---

## Mục tiêu chương

Data model có thể là phần **quan trọng nhất** của việc phát triển software — nó ảnh hưởng sâu sắc không chỉ đến **cách viết code**, mà còn đến **cách ta nghĩ về vấn đề** đang giải quyết.

> Epigraph mở đầu chương:
> _"The limits of my language mean the limits of my world."_
> — Ludwig Wittgenstein, Tractatus Logico-Philosophicus (1922)

---

## Các Layer của Data Model trong một Application

```
1. Real world (người, tổ chức, hàng hóa, hành động, dòng tiền,...)
        │ (App developer modeling)
        ▼
2. General-purpose data model (JSON/XML documents, relational tables,
                                 graph vertices/edges) ← CHƯƠNG NÀY
        │ (Database engineers)
        ▼
3. Bytes trong memory/disk/network (Storage engine) ← Chapter 4
        │ (Hardware engineers)
        ▼
4. Electrical currents, light pulses, magnetic fields
```

→ Mỗi layer **ẩn complexity** của layer dưới, cho phép các nhóm người khác nhau (database vendor engineers, application developers) làm việc hiệu quả cùng nhau.

---

## Cấu trúc tài liệu này

| File                                  | Nội dung                                                                                            |
| ------------------------------------- | --------------------------------------------------------------------------------------------------- |
| `01_Relational_vs_Document_Models.md` | Object-relational mismatch, ORM, Normalization/Denormalization, Many-to-Many, Star/Snowflake schema |
| `02_Graph_Like_Data_Models.md`        | Property Graph, Cypher, Triple Stores/RDF, SPARQL, Datalog                                          |
| `03_GraphQL_and_Event_Sourcing.md`    | GraphQL, Event Sourcing, CQRS                                                                       |
| `04_DataFrames_Matrices_Arrays.md`    | DataFrames, Matrices cho ML/Data Science                                                            |
| `05_Summary_and_Key_Concepts.md`      | Tổng hợp toàn chương, glossary, mindmap, câu hỏi ôn tập                                             |

---

## Big Picture — Các Data Model chính

```
┌────────────────────────────────────────────────────────┐
│              DATA MODELS TRONG CHƯƠNG 3                │
│                                                          │
│  1. RELATIONAL MODEL    — tables, SQL, JOIN             │
│        ⇄ trade-off ⇄                                    │
│     DOCUMENT MODEL      — JSON, schema-on-read          │
│                                                          │
│  2. GRAPH MODELS                                        │
│        ├─ Property Graph (Cypher)                       │
│        ├─ Triple Store / RDF (SPARQL)                    │
│        └─ Datalog (recursive relational queries)         │
│                                                          │
│  3. GraphQL — query language cho client-server OLTP      │
│                                                          │
│  4. Event Sourcing & CQRS — append-only log + views      │
│                                                          │
│  5. DataFrames / Matrices — cho ML & scientific computing│
└────────────────────────────────────────────────────────┘
```

---

## Thuật ngữ quan trọng: Declarative Query Languages

> SQL, Cypher, SPARQL, Datalog đều là **declarative**: bạn chỉ ra **pattern** của data muốn lấy (điều kiện, sort, group, aggregate), KHÔNG cần chỉ ra **thuật toán cụ thể** để đạt được nó. Query optimizer của database tự quyết định cách thực thi tối ưu (index nào, join algorithm nào, thứ tự thực thi).

```
Declarative (SQL, Cypher,...)        Imperative (Python, Java,...)
   "Lấy gì, điều kiện gì"      vs       "Làm CHÍNH XÁC theo bước nào"
   Database tự tối ưu                   Người viết code tự tối ưu
```

→ Lợi ích: Database có thể cải thiện performance (parallel execution, indexes) **mà không cần sửa query**.

---

## Các chương liên quan

- **Chapter 1:** Operational vs Analytical Systems (nền tảng cho Star/Snowflake schemas)
- **Chapter 2:** Systems of Record and Derived Data (liên quan đến Normalization/Denormalization)
- **Chapter 4:** Storage and Retrieval (cách lưu trữ data models này)
- **Chapter 5:** Encoding and Evolution (JSON, XML, schema evolution)
- **Chapter 10:** Consistency and Consensus (event ordering trong Event Sourcing)
- **Chapter 11:** Batch Processing (DataFrames trong Spark/Flink)
- **Chapter 12:** Stream Processing (Event sourcing, message brokers như Kafka)
