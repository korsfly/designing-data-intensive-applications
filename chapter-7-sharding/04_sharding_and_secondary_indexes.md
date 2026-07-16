# Sharding và Secondary Indexes

---

## 1. Vấn đề với Secondary Indexes khi Sharding

### Primary Key Index — Đơn giản

```
Các scheme sharding đã thảo luận: dựa vào client BIẾT partition key của record
   → Primary key: partition key = đầu tiên (hoặc toàn bộ) của primary key
   → Routing dễ: key → shard → node

VÍ DỤ: Key-value model:
   Primary key = partition key
   → Always route to correct node directly
```

### Secondary Indexes — Phức tạp hơn

```
Secondary index KHÔNG identify record uniquely
   → "Find all records WHERE value = X"
   → Phục vụ: search by non-primary-key columns

VẤND ĐỀ: Secondary indexes DON'T MAP NEATLY to shards
   → Có 2 approaches: LOCAL và GLOBAL secondary indexes
```

---

## 2. Local Secondary Indexes (Document-Partitioned)

### Nguyên tắc (Figure 7-9)

```
Mỗi shard independently maintain OWN SECONDARY INDEXES
   → Chỉ COVER RECORDS TRONG SHARD ĐÓ
   → Không quan tâm data ở shards khác

Write: Chỉ cần deal với 1 shard (shard chứa record đó)
   Thêm/xóa/update record → chỉ update index trên CÙNG SHARD

→ Gọi là "local index" hoặc "document-partitioned index" (IR terminology)
```

### Ví dụ: Used Car Website (Figure 7-9)

```
Schema: mỗi car listing có unique ID
Partition key: car ID
   IDs 0–499 → Shard 0
   IDs 500–999 → Shard 1

Secondary indexes: color, make (để search/filter)

Khi thêm red car vào database:
   → Shard tự động add car ID vào list của index entry "color:red"
   → Đây là POSTINGS LIST (Chapter 4)

Shard 0 "color:red": [IDs của red cars trong shard 0]
Shard 1 "color:red": [IDs của red cars trong shard 1]
```

### Đọc từ Local Secondary Index

```
BIẾT partition key → chỉ query ĐÚNG SHARD (đơn giản)

KHÔNG biết partition key → muốn TẤT CẢ red cars:
   → Phải query TẤT CẢ SHARDS
   → COMBINE results từ mọi shard
   → Gọi là SCATTER/GATHER query

NHƯỢC ĐIỂM của Scatter/Gather:
   Expensive! Ngay cả khi query shards SONG SONG (parallel)
   → Prone to TAIL LATENCY AMPLIFICATION (Chapter 2):
     Phải chờ SHARD CHẬM NHẤT → response time tăng

   Scalability limitation:
   Thêm shards → store more data ✓
   NHƯNG: KHÔNG tăng query throughput
      (mỗi shard phải process EVERY QUERY anyway)
   → Query throughput limited bởi SINGLE SHARD capacity
```

### Databases dùng Local Secondary Index

```
MongoDB, Riak, Cassandra, Elasticsearch, SolrCloud, VoltDB
→ WIDELY USED despite scatter-gather issue
```

### Tự implement Secondary Index

```
Nếu database chỉ support key-value → có thể tự implement secondary index
   trong APPLICATION CODE (maintain mapping từ values đến IDs)

CẢNH BÁO: Cần rất cẩn thận để index CONSISTENT với underlying data
   Race conditions + intermittent write failures → out of sync
   → Cần MULTI-OBJECT TRANSACTIONS (Chapter 8)
```

---

## 3. Global Secondary Indexes (Term-Partitioned)

### Nguyên tắc (Figure 7-10)

```
Construct 1 GLOBAL INDEX cover data trong TẤT CẢ shards
   → Thay vì mỗi shard có local index riêng

NHƯNG: Không thể lưu global index trên 1 node:
   → Would become BOTTLENECK, defeat purpose of sharding

→ Global index cũng phải được SHARDED
   (nhưng sharded KHÁC VỚI primary-key sharding)
```

### Ví dụ (Figure 7-10)

```
Global index sharding:
   Color index: colors a–r → Index shard 0; colors s–z → Index shard 1
   Make index: makes a–f → Index shard 0; makes g–z → Index shard 1

Red cars từ TẤT CẢ data shards → xuất hiện dưới "color:red" trong index shard 0

→ Gọi là "TERM-PARTITIONED index" (từ information retrieval)
   Term = value được index (color, make,...)
   Term là partition key của global index
```

### Reads từ Global Secondary Index

```
Query với SINGLE CONDITION (color = red):
   → Chỉ cần read từ 1 INDEX SHARD (chứa "color:red")
   → Postings list: IDs của red cars từ TẤT CẢ data shards

NHƯNG để fetch ACTUAL RECORDS (không chỉ IDs):
   → Vẫn phải read từ TẤT CẢ data shards chứa những IDs đó
   → Vẫn cần scatter-gather cho records

Multiple conditions (color AND make):
   → Terms có thể ở DIFFERENT INDEX SHARDS
   → System cần tìm IDs trong CẢ HAI postings lists → compute intersection
   → Gửi long postings lists qua network → SLOW
```

### Writes vào Global Secondary Index

```
Phức tạp hơn local indexes!

Write 1 record → có thể affect NHIỀU INDEX SHARDS:
   (mỗi term trong document có thể ở different shard)

→ Harder to keep index in sync with underlying data

OPTION 1: Distributed transaction để atomic update data shard + index shards
   (Chapter 8)

OPTION 2: Async update (DynamoDB approach):
   Writes asynchronously reflected trong global indexes
   → Reads từ global index CÓ THỂ STALE
   (tương tự replication lag — Chapter 6)
```

### Databases dùng Global Secondary Index

```
CockroachDB, TiDB, YugabyteDB → global secondary indexes
DynamoDB → hỗ trợ CẢ HAI local AND global secondary indexes

Global indexes useful khi:
   Read throughput > write throughput
   Postings lists không quá dài
```

---

## 4. So sánh Local vs Global Secondary Indexes

| Khía cạnh                   | Local Index                   | Global Index                         |
| --------------------------- | ----------------------------- | ------------------------------------ |
| **Write**                   | Đơn giản (chỉ 1 shard update) | Phức tạp (nhiều index shards)        |
| **Read (single condition)** | Scatter-gather (ALL shards)   | 1 index shard (chỉ lấy IDs)          |
| **Read (actual records)**   | Scatter-gather                | Vẫn scatter-gather                   |
| **Consistency**             | Dễ (cùng shard)               | Khó; DynamoDB: async (stale)         |
| **Scalability**             | Query throughput giới hạn     | Read throughput tốt hơn              |
| **Use when**                | Write-heavy                   | Read-heavy                           |
| **Examples**                | MongoDB, Cassandra, VoltDB    | CockroachDB, TiDB, DynamoDB (global) |

---

## Tóm tắt phần này

```
SECONDARY INDEXES VÀ SHARDING:
   Khó hơn primary key vì không map neatly to shards

LOCAL SECONDARY INDEX (per-shard):
   ✓ Writes đơn giản (chỉ cùng shard)
   ✗ Reads: scatter-gather QUA TẤT CẢ SHARDS
   ✗ Tail latency amplification
   ✗ Query throughput không scale khi thêm shards

GLOBAL SECONDARY INDEX (term-partitioned):
   ✓ Reads (single condition): 1 index shard → postings list
   ✗ Writes phức tạp: nhiều index shards có thể bị affect
   ✗ Multi-condition reads: vẫn cần phối hợp nhiều index shards
   ✗ Consistency: cần distributed transactions HOẶC accept async (stale)

THỰC TẾ:
   Hầu hết databases chọn 1 trong 2, trade-off theo workload
   DynamoDB: hỗ trợ cả 2 loại (flexible nhất)
```
