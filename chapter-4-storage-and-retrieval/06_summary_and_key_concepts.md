# Summary & Key Concepts — Chapter 4

---

## 1. Tổng hợp toàn chương

Chương 4 nhìn "bên dưới mui xe" của database: storage engines lưu trữ data như thế nào, và index nào giúp truy vấn nhanh. Hai thế giới hoàn toàn khác nhau:

```
OLTP Storage Engines:
   ├─ Log-Structured (LSM-Trees): tốt cho WRITE-HEAVY
   └─ Update-in-Place (B-Trees): tốt cho READ-HEAVY

Analytics Storage Engines:
   ├─ Column-Oriented Storage: chỉ load COLUMNS CẦN THIẾT
   ├─ Bitmap Compression: bitwise ops NHANH
   ├─ Query Execution: Compilation (JIT) hay Vectorization
   └─ Materialized Views / Data Cubes: precompute aggregates

Advanced Indexes:
   ├─ Multidimensional (R-tree): geospatial + multi-condition
   ├─ Inverted Index: full-text search
   └─ Vector Index (IVF, HNSW): semantic search
```

### Một Nguyên tắc Xuyên Suốt

> **Index là Trade-off:** Index TỐT cho READ, CHẬM cho WRITE và TỐN disk space thêm.

---

## 2. Bảng thuật ngữ (Glossary)

### Log-Structured Storage

| Thuật ngữ                                | Định nghĩa                                                                     |
| ---------------------------------------- | ------------------------------------------------------------------------------ |
| **Log**                                  | Append-only sequence of records trên disk (general sense trong sách này)       |
| **Hash index**                           | In-memory hash map: key → byte offset trong log file                           |
| **SSTable** (Sorted String Table)        | Log file với key-value pairs SORTED THEO KEY, mỗi key 1 lần                    |
| **Sparse index**                         | Index chỉ lưu MỘT SỐ keys (thay vì tất cả), đủ để navigate đến đúng block      |
| **Memtable**                             | In-memory ordered map data structure (red-black tree, skip list, trie)         |
| **Segment**                              | SSTable file được write ra disk từ memtable                                    |
| **Compaction**                           | Merge các segment files, loại bỏ giá trị bị overwrite/xóa                      |
| **Tombstone**                            | Deletion record đặc biệt trong log-structured storage, báo hiệu xóa key        |
| **LSM-tree** (Log-Structured Merge-tree) | Kiến trúc storage engine dựa trên memtable + SSTable + background merge        |
| **Bloom filter**                         | Probabilistic data structure: check NHANH xem 1 key CÓ trong SSTable hay không |
| **False positive**                       | Bloom filter báo "có" nhưng thực tế key KHÔNG tồn tại                          |
| **Size-tiered compaction**               | Merge nhỏ → lớn dần; write-heavy                                               |
| **Leveled compaction**                   | SSTable size cố định, nhóm thành levels; read-heavy                            |

### B-Trees

| Thuật ngữ                    | Định nghĩa                                                            |
| ---------------------------- | --------------------------------------------------------------------- |
| **Page/Block**               | Đơn vị cố định của B-tree (4-16 KiB), có thể overwrite tại chỗ        |
| **Root page**                | Page bắt đầu của mọi B-tree lookup                                    |
| **Leaf page**                | Page chứa individual keys và values (hoặc reference đến values)       |
| **Branching factor**         | Số lượng references đến child pages trong 1 page                      |
| **Page split**               | Chia 1 page đầy thành 2 pages nửa-đầy khi insert                      |
| **WAL** (Write-Ahead Log)    | Append-only log phải viết TRƯỚC KHI áp dụng thay đổi vào B-tree pages |
| **Torn page**                | Page chỉ được viết một phần do crash (problem với B-tree)             |
| **Copy-on-Write**            | Alternative của WAL: viết modified page vào VỊ TRÍ KHÁC               |
| **Write amplification**      | Tỉ lệ bytes thực sự viết / bytes cần viết nếu dùng append-only log    |
| **Random writes**            | Nhiều write nhỏ scattered (pattern của B-tree)                        |
| **Sequential writes**        | Ít write lớn tập trung (pattern của LSM-tree)                         |
| **Fragmentation**            | Pages không dùng trong B-tree sau khi xóa nhiều keys                  |
| **Garbage collection (SSD)** | Process dọn dẹp blocks trên SSD trước khi erase                       |

### Indexes & Databases

| Thuật ngữ              | Định nghĩa                                                            |
| ---------------------- | --------------------------------------------------------------------- |
| **Secondary index**    | Index trên column KHÔNG PHẢI primary key                              |
| **Clustered index**    | Actual data lưu TRỰC TIẾP trong cấu trúc index                        |
| **Heap file**          | Nơi lưu rows không có thứ tự, được reference bởi non-clustered index  |
| **Covering index**     | Index lưu thêm MỘT SỐ columns, cho phép answer query CHỈ từ index     |
| **Embedded database**  | Library chạy trong cùng process với application, không có network API |
| **In-memory database** | Database phục vụ reads HOÀN TOÀN từ memory (disk chỉ cho durability)  |
| **Forwarding pointer** | Pointer tại vị trí heap cũ trỏ đến vị trí mới khi record bị di chuyển |

### Analytics Storage

| Thuật ngữ                                    | Định nghĩa                                                                          |
| -------------------------------------------- | ----------------------------------------------------------------------------------- |
| **Column-oriented / Columnar storage**       | Lưu values của cùng COLUMN gần nhau (thay vì cùng row)                              |
| **Wide-column / column-family**              | Mô hình khác (Bigtable, HBase) — thực ra ROW-ORIENTED, đừng nhầm                    |
| **Bitmap encoding**                          | n distinct values → n bitmaps, 1 bit per row                                        |
| **Run-length encoding**                      | Đếm consecutive 0s/1s thay vì lưu từng bit                                          |
| **Roaring bitmap**                           | Switch giữa hai bitmap representations, dùng cái nén nhất                           |
| **Vectorized processing**                    | Xử lý NHIỀU values từ 1 column CÙNG LÚC (batch), dùng SIMD                          |
| **Query compilation / JIT**                  | Generate + compile machine code từ SQL query (LLVM-based)                           |
| **SIMD** (Single-Instruction, Multiple-Data) | CPU instruction xử lý nhiều data cùng lúc                                           |
| **Materialized aggregate**                   | Precomputed aggregate (COUNT, SUM,...) được cache                                   |
| **Data cube / OLAP cube**                    | Grid of aggregates grouped by multiple dimensions                                   |
| **Query engine**                             | Parse SQL, optimize, execute (Trino, DataFusion, Presto)                            |
| **Storage format**                           | Cách encode rows thành bytes trong file (Parquet, ORC, Lance, Nimble)               |
| **Table format**                             | Quản lý immutable files: định nghĩa table + schema + insert/delete (Iceberg, Delta) |
| **Data catalog**                             | Định nghĩa tables trong database; create/rename/drop (Polaris, Unity Catalog)       |
| **Time travel**                              | Khả năng query table tại thời điểm TRƯỚC ĐÂY (table format feature)                 |

### Advanced Indexes

| Thuật ngữ                                | Định nghĩa                                                             |
| ---------------------------------------- | ---------------------------------------------------------------------- |
| **Concatenated index**                   | Combine nhiều fields thành 1 key theo thứ tự                           |
| **Multidimensional index**               | Query NHIỀU conditions/dimensions đồng thời                            |
| **R-tree**                               | Spatial index chia space sao cho điểm gần nhau cùng subtree            |
| **Bkd-tree**                             | Dynamic scalable variant của kd-tree, dùng trong geospatial            |
| **Space-filling curve**                  | Translate 2D location → 1 số để dùng B-tree index                      |
| **Full-text search**                     | Search documents bằng keywords xuất hiện trong text                    |
| **Inverted index**                       | Key = term, Value = postings list (doc IDs chứa term)                  |
| **Postings list**                        | Danh sách document IDs chứa 1 term cụ thể                              |
| **n-gram / trigram**                     | Substrings độ dài n, dùng để build inverted index cho substring search |
| **Edit distance**                        | Số lần add/remove/replace 1 ký tự để đổi từ này sang từ kia            |
| **Levenshtein automaton**                | Finite state automaton cho phép fuzzy search trong given edit distance |
| **Vector embedding**                     | Array of floats đại diện document trong multidimensional space         |
| **Embedding model**                      | Model (thường neural network) tạo vector embeddings                    |
| **Semantic search**                      | Search dựa trên NGHĨA, không chỉ keyword matching                      |
| **RAG** (Retrieval-Augmented Generation) | Tích hợp semantic search vào LLM output                                |
| **Cosine similarity**                    | Đo cosine góc giữa 2 vectors (khoảng cách)                             |
| **Euclidean distance**                   | Đo đường thẳng giữa 2 điểm trong space                                 |
| **Flat index**                           | Lưu vectors nguyên vẹn, exact search nhưng chậm                        |
| **IVF index**                            | Cluster vectors → search trong clusters gần nhất (approximate)         |
| **HNSW index**                           | Multi-layer graph, traverse từ sparse top → dense bottom (approximate) |
| **Centroid**                             | Trung tâm của 1 cluster trong IVF index                                |
| **Probe**                                | Số partitions cần check trong IVF query                                |
| **Faiss**                                | Facebook's vector search library (IVF + HNSW variants)                 |
| **pgvector**                             | PostgreSQL extension cho vector search                                 |

---

## 3. Mindmap khái niệm

```
                    STORAGE AND RETRIEVAL
                             │
           ┌─────────────────┼──────────────────────┐
           │                 │                      │
      OLTP STORAGE      ANALYTICS STORAGE       ADVANCED INDEXES
           │                 │                      │
    ┌──────┴──────┐    ┌──────┴──────┐        ┌─────┴──────┐
    │             │    │             │        │     │      │
 LOG-STRUCT   B-TREE  COLUMNAR     QUERY    MULTI-  FULL-  VECTOR
  (LSM)               STORAGE     EXEC.    DIM.   TEXT   EMBED.
    │             │    │             │      (R-tree)(Inv.   (HNSW
  SSTable    Page    Values of    JIT or  Space-  Index)  IVF)
  Memtable   split   same COL     Vectorize fill         Semantic
  Compaction  WAL    together   Materialized  Curv      Search
  Bloom filt  WA-   Bitmap Enc  Views/Cubes         RAG
              Amp   Sort Order  Sort+Compress
              Frag  Write: Batch
```

---

## 4. Câu hỏi tự kiểm tra (Self-Check Questions)

1. Mô tả **"database đơn giản nhất thế giới"** (2 bash functions). Tại sao `db_set` nhanh nhưng `db_get` chậm với nhiều records?
2. Giải thích **hash index** (in-memory). Liệt kê 4 vấn đề của nó.
3. **SSTable** khác **append-only log** ở điểm gì? Tại sao sparse index hiệu quả hơn full index?
4. Mô tả 4 bước của **LSM-tree** (memtable → SSTable → compaction). Khi nào xảy ra tombstone?
5. **Bloom filter** hoạt động như thế nào? Tại sao false positive không vấn đề trong LSM context?
6. So sánh **size-tiered compaction** vs **leveled compaction**. Dùng cái nào cho write-heavy workload?
7. Giải thích cấu trúc **B-tree** (page, root, leaf, branching factor). B-tree với 4 levels và branching factor 500 lưu được bao nhiêu data?
8. Tại sao **page split** là rủi ro trong B-tree? WAL giải quyết vấn đề này như thế nào?
9. So sánh **B-tree vs LSM-tree** theo 5 khía cạnh: read performance, range query, write throughput, write amplification, fragmentation.
10. Phân biệt **clustered index**, **heap file (non-clustered)**, và **covering index**.
11. Khi nào **embedded database** (SQLite, DuckDB,...) là lựa chọn phù hợp?
12. Tại sao **in-memory database** NHANH HƠN disk-based (dù OS có thể cache disk blocks)? Durability được đảm bảo như thế nào?
13. Giải thích **column-oriented storage**. Tại sao nó tốt hơn row-oriented cho analytics query?
14. Mô tả **bitmap encoding** + **run-length encoding** trong columnar storage. Cho ví dụ bitwise AND query.
15. Tại sao **sort order** trong columnar storage giúp cả query performance lẫn compression?
16. So sánh **query compilation (JIT)** và **vectorized processing**. Hai kỹ thuật CPU nào giúp cả 2 approach nhanh?
17. Phân biệt **materialized view** và **virtual view**. **Data cube** là gì và nhược điểm gì?
18. Tại sao **concatenated index** không thể query geospatial data (lat + lng) hiệu quả? Giải pháp là gì?
19. Giải thích **inverted index**. Tại sao bitwise AND của 2 postings list bitmaps cho kết quả nhanh?
20. Phân biệt **vector embedding** và **vectorized processing** (hai nghĩa khác nhau của "vector"). Mô tả 3 loại **vector index** (Flat, IVF, HNSW).

---

## 5. Liên kết tới các chương khác

| Chapter 4 đề cập                         | Mở rộng ở                                                |
| ---------------------------------------- | -------------------------------------------------------- |
| Crash recovery, WAL, durability          | **Chapter 8** — Transactions                             |
| B-tree copy-on-write, snapshot isolation | **Chapter 8** — "Snapshot Isolation and Repeatable Read" |
| Scale storage qua nhiều machines         | **Chapter 6** — Replication, **Chapter 7** — Sharding    |
| Immutable SSTable segment files          | **Chapter 12** — "Immutability"                          |
| Maintaining materialized views at scale  | **Chapter 12** — "Maintaining materialized views"        |
| Star/Snowflake schema                    | **Chapter 3** — Data Models                              |
| Data Lake, object storage                | **Chapter 1** — Cloud Native Architecture                |
| Column-oriented formats (Parquet)        | **Chapter 11** — Batch Processing                        |

---

## 6. Trích dẫn mở đầu chương (Epigraph)

> _"One of the miseries of life is that everybody names things a little bit wrong... A computer does not primarily compute in the sense of doing arithmetic. [...] They primarily are filing systems."_
> — **Richard Feynman**, Idiosyncratic Thinking seminar (1985)

→ Feynman nhìn nhận rằng máy tính thực chất là **hệ thống lưu trữ và truy vấn thông tin** — cái mà chương 4 phân tích một cách chi tiết từ góc nhìn của database storage engine.
