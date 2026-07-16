# Indexes, Embedded Databases, và In-Memory Databases

---

## 1. Secondary Indexes

### Primary Key Index — Nhắc lại

```
Primary key: ĐỊNH DANH DUY NHẤT 1 row trong relational table
             (hoặc 1 document trong document DB, 1 vertex trong graph DB)

Primary key index: dùng để RESOLVE references từ records khác
   (vd: foreign key lookup)
```

### Secondary Index

```
NGOÀI primary key, thường CẦN thêm secondary indexes
   để search theo CỘT KHÁC

Ví dụ: Schema của Figure 3-1:
   Tìm TẤT CẢ rows thuộc về CÙNG 1 user_id trong mỗi table
   → Cần secondary index trên cột user_id
```

### Xây dựng Secondary Index từ Key-Value Index

```
Khác biệt CHÍNH: trong secondary index, giá trị được index
   KHÔNG NHẤT THIẾT UNIQUE
   → NHIỀU rows/documents/vertices có thể có CÙNG index entry

Giải quyết bằng 1 trong 2 cách:
   1. Mỗi value trong index = LIST các row identifier phù hợp
      (giống "postings list" trong full-text index)
   2. Làm mỗi entry UNIQUE bằng cách APPEND row identifier vào nó

CẢ HAI: storage engine có in-place update (B-tree)
        VÀ log-structured storage đều CÓ THỂ implement secondary index
```

---

## 2. Lưu trữ Values TRONG Index

### 3 Loại Index theo cách lưu value

#### Loại 1: Clustered Index

```
Actual data (row, document, vertex) được lưu TRỰC TIẾP TRONG
cấu trúc index

Ví dụ:
   MySQL InnoDB: primary key LUÔN là clustered index
   SQL Server: CÓ THỂ chỉ định 1 clustered index/table
```

#### Loại 2: Non-clustered Index → Heap File

```
Value trong index = REFERENCE đến actual data:
   - Có thể là primary key của row (InnoDB dùng cho secondary index)
   - Hoặc direct reference đến VỊ TRÍ trên disk

HEAP FILE: nơi rows được lưu, KHÔNG CÓ THỨ TỰ cụ thể
   (có thể là append-only HOẶC track deleted rows
    để overwrite bằng data mới)

Ví dụ: PostgreSQL dùng heap file approach

Lợi ích: khi UPDATE giá trị (KHÔNG đổi key)
   Nếu value mới ≤ value cũ size:
     CÓ THỂ overwrite tại chỗ trong heap file
     (không cần update TẤT CẢ index!)
   Nếu value mới LỚN HƠN:
     Phải MOVE record sang vị trí mới trong heap
     → CẦN update TẤT CẢ index để trỏ vào vị trí heap MỚI
       HOẶC để lại FORWARDING POINTER tại vị trí heap cũ
```

#### Loại 3: Covering Index (Index with Included Columns)

```
Middle ground: lưu MỘT SỐ cột của table TRONG index
   (NGOÀI primary key/heap reference)

   → Một số query CÓ THỂ được answered bằng index ALONE
     không cần resolve primary key / look in heap file

   → "Index COVERS the query" = index alone đủ để answer query

Lợi ích: 1 số query NHANH HƠN
Nhược điểm: DATA DUPLICATION → index DÙNG NHIỀU disk space hơn,
             CHẬM write hơn (update index redundant data)
```

---

## 3. Embedded Storage Engines

### Định nghĩa

```
Hầu hết database chạy như SERVICE:
   Nhận queries qua NETWORK API

EMBEDDED DATABASE:
   KHÔNG expose network API
   Là LIBRARY chạy trong CÙNG PROCESS với application code
   Thường đọc/ghi file trên LOCAL DISK
   Bạn tương tác qua FUNCTION CALLS thông thường
```

### Ví dụ

| Embedded Storage Engine | Đặc điểm                      |
| ----------------------- | ----------------------------- |
| **RocksDB**             | LSM-based, high performance   |
| **SQLite**              | Relational, rất phổ biến, nhẹ |
| **LMDB**                | B-tree based, copy-on-write   |
| **DuckDB**              | Analytics (OLAP), columnar    |
| **KùzuDB**              | Graph database                |

### Khi nào dùng Embedded Database?

```
Phổ biến trong MOBILE APPS:
   Lưu data cục bộ của user

Trên backend: PHÙ HỢP khi:
   1. Data ĐỦ NHỎ để fit trên SINGLE MACHINE
   2. KHÔNG có NHIỀU concurrent transactions

Ví dụ BACKEND điển hình:
   MULTITENANT SYSTEM: mỗi tenant NHỎ và HOÀN TOÀN RIÊNG
   (không cần query kết hợp data từ nhiều tenant)
   → CÓ THỂ dùng SEPARATE EMBEDDED DB INSTANCE per tenant

Bluesky (social network) thực tế đã migrate sang single-tenant SQLite
```

---

## 4. In-Memory Databases

### Động lực: RAM ngày càng rẻ

```
Disk có 2 LỢI THẾ so với RAM:
   1. DURABLE (data không mất khi mất điện)
   2. CHI PHÍ/GB THẤP HƠN nhiều so với RAM

Khi RAM NGÀY CÀNG RẺ:
   → Argument về cost/GB yếu đi
   → Nhiều dataset ĐỦ NHỎ để fit HOÀN TOÀN trong memory
     (có thể distributed across several machines)
   → IN-MEMORY DATABASES xuất hiện
```

### Hai loại In-Memory DB

#### Loại 1: Cache-only (Không quan tâm Durability)

```
Ví dụ: MEMCACHED
   → Acceptable để DATA MẤT khi machine restart
   → Chỉ dùng cho CACHING USE CASE
```

#### Loại 2: Durable In-Memory Database

```
Cung cấp DURABILITY qua một trong các cách:
   1. Special hardware: BATTERY-POWERED RAM
   2. WRITE LOG OF CHANGES TO DISK (append-only log)
   3. PERIODIC SNAPSHOTS TO DISK
   4. REPLICATE in-memory state đến OTHER MACHINES

Khi restart:
   → Reload state từ DISK HOẶC QUA NETWORK từ replica

Disk chỉ dùng như APPEND-ONLY LOG cho durability
   READS: served HOÀN TOÀN từ MEMORY

Ví dụ:
   VoltDB, SingleStore, Oracle TimesTen (relational model)
   RAMCloud (open source, key-value, log-structured trong memory VÀ disk)
   Redis, Couchbase (WEAK durability — write to disk ASYNC)
```

### Tại sao In-Memory DB NHANH HƠN?

```
COUNTERINTUITIVE: KHÔNG phải vì không cần đọc từ disk!

   Disk-based storage engine: nếu có ĐỦ MEMORY,
   OS CACHE các disk block gần đây vào memory anyway
   → Về cơ bản CŨNG đọc từ memory

THỰC SỰ: In-memory DB NHANH hơn vì
   TRÁNH ĐƯỢC OVERHEAD của việc ENCODE in-memory
   data structures thành form CÓ THỂ viết ra disk
```

### Data Models khó implement với Disk-Based Indexes

```
In-memory databases CÒN cho phép TRIỂN KHAI
   data models KHÓ VỚI disk-based indexes

Ví dụ: Redis cung cấp database-like interface
   cho nhiều DATA STRUCTURE:
   - Priority queues
   - Sets
   - Sorted sets (ZSet)
   - Lists
   - Hash maps

   Vì giữ TẤT CẢ data trong memory
   → Implementation TƯƠNG ĐỐI ĐƠN GIẢN
```

---

## Tóm tắt phần này

```
SECONDARY INDEX:
  - Cần thiết để search theo column KHÁC primary key
  - Value trong index KHÔNG UNIQUE → list of row IDs / append row ID
  - Cả B-tree VÀ LSM-tree đều có thể implement

3 LOẠI INDEX THEO CÁCH LƯU VALUE:
  1. Clustered Index: actual data TRONG index structure
  2. Non-clustered → Heap File: reference đến VỊ TRÍ data
  3. Covering Index: lưu MỘT SỐ cột trong index
                    (query nào phù hợp → không cần vào heap)

EMBEDDED DATABASES:
  - LIBRARY trong cùng process, không có network API
  - Phù hợp cho: mobile apps, single-machine data, per-tenant model
  - Ví dụ: RocksDB, SQLite, LMDB, DuckDB, KùzuDB

IN-MEMORY DATABASES:
  - NHANH vì tránh encode/decode giữa memory và disk format
  - KHÔNG phải vì không đọc từ disk (OS cache anyway)
  - Durability: WAL / periodic snapshot / replication
  - Cho phép data structure phức tạp khó implement với disk-based
  - Ví dụ: Redis, Memcached, VoltDB, RAMCloud
```
