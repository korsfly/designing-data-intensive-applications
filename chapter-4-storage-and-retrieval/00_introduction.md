# DDIA 2nd Edition — Chapter 4: Storage and Retrieval

> **Sách:** Designing Data-Intensive Applications, 2nd Edition  
> **Tác giả:** Martin Kleppmann & Chris Riccomini  
> **Chương:** 4 — Storage and Retrieval

---

## Mục tiêu chương

Chương 3 thảo luận **data models và query languages** — format mà ta đưa data vào database, và interface để hỏi lại data đó. Chương 4 nhìn từ **góc độ của database**: database **lưu trữ** data như thế nào, và **tìm lại** data đó như thế nào khi được hỏi.

> Epigraph mở đầu chương:
> _"One of the miseries of life is that everybody names things a little bit wrong... A computer does not primarily compute in the sense of doing arithmetic. [...] They primarily are filing systems."_
> — Richard Feynman, Idiosyncratic Thinking seminar (1985)

### Tại sao Application Developer cần hiểu Storage Engine?

```
Bạn KHÔNG có khả năng (và thường KHÔNG cần) tự implement storage engine
NHƯNG bạn CẦN:
   1. Chọn storage engine PHÙ HỢP cho application của mình
   2. Hiểu workload để CONFIGURE storage engine hiệu quả
   3. Có ý niệm cơ bản về storage engine "DƯỚI MUI XE" đang làm gì
```

---

## Big Picture — Cấu trúc chương

```
┌─────────────────────────────────────────────────────────┐
│           STORAGE AND RETRIEVAL — 2 NHÓM CHÍNH           │
│                                                            │
│  1. OLTP STORAGE ENGINES                                  │
│        ├─ Log-Structured (LSM-Trees, SSTables)            │
│        └─ Update-in-Place (B-Trees)                       │
│                                                            │
│  2. ANALYTICS STORAGE ENGINES                              │
│        ├─ Column-Oriented Storage                          │
│        ├─ Compression (Bitmap encoding)                    │
│        ├─ Query Execution (Compilation/Vectorization)       │
│        └─ Materialized Views & Data Cubes                   │
│                                                            │
│  3. ADVANCED INDEXES                                        │
│        ├─ Multidimensional Indexes (R-trees)                 │
│        ├─ Full-Text Search (Inverted Index)                   │
│        └─ Vector Embeddings (Semantic Search)                  │
└─────────────────────────────────────────────────────────┘
```

---

## Cấu trúc tài liệu này

| File                                             | Nội dung                                                                                  |
| ------------------------------------------------ | ----------------------------------------------------------------------------------------- |
| `01_Log_Structured_Storage.md`                   | Đơn giản nhất: append-only log, hash index, SSTable, LSM-Trees, Bloom filters, Compaction |
| `02_B_Trees.md`                                  | B-Tree structure, reliability (WAL), variants, so sánh B-Tree vs LSM-Tree                 |
| `03_Indexes_Embedded_InMemory.md`                | Multicolumn/secondary index, clustered index, embedded databases, in-memory databases     |
| `04_Data_Storage_for_Analytics.md`               | Cloud data warehouses, column-oriented storage, compression, query execution              |
| `05_Multidimensional_FullText_Vector_Indexes.md` | R-trees, full-text search (inverted index), vector embeddings (semantic search)           |
| `06_Summary_and_Key_Concepts.md`                 | Tổng hợp toàn chương, glossary, mindmap, câu hỏi ôn tập                                   |

---

## Nguyên tắc nền tảng: Index là Trade-off

> Index là cấu trúc **PHỤ** (additional), được **DERIVE** từ primary data. Thêm/xóa index KHÔNG ảnh hưởng đến nội dung database — chỉ ảnh hưởng **performance của query**.

```
Index TỐT cho:     READ (tăng tốc query)
Index TRẢ GIÁ ở:   WRITE (mỗi write phải update thêm index)
                   DISK SPACE (lưu thêm cấu trúc)

→ Database KHÔNG tự động index MỌI THỨ
→ Developer/admin PHẢI tự chọn index dựa trên
  query pattern THỰC TẾ của application
```

---

## Các chương liên quan

- **Chapter 1:** Operational vs Analytical Systems (nền tảng OLTP/OLAP)
- **Chapter 3:** Data Models (Star/Snowflake schema, document model)
- **Chapter 5:** Encoding and Evolution (cách data được serialize)
- **Chapter 6, 7:** Replication, Sharding (scale storage qua nhiều máy)
- **Chapter 8:** Transactions (atomicity, crash recovery, durability)
- **Chapter 12:** Stream Processing (Materialize, maintaining materialized views)
