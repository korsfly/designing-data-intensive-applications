# Hash Sharding và Consistent Hashing

---

## 1. Hash Sharding — Nguyên tắc cơ bản

### Ý tưởng

```
Thay vì assign key ranges → Apply HASH FUNCTION lên partition key
→ Assign RANGE OF HASH VALUES cho mỗi shard

Hash function:
   Input: partition key (bất kỳ string/number nào)
   Output: số nguyên trong range cố định (vd: 0–65,535 cho 16-bit hash)

Tính chất quan trọng:
   Dù input keys RẤT GIỐNG NHAU (vd: consecutive timestamps)
   → Hash values ĐƯỢc DISTRIBUTE ĐỀU (uniformly distributed)
   → Giảm hot spots
```

### Ví dụ (Figure 7-5)

```
16-bit hash → range 0 đến 65,535

Shard 0: hash values 0 – 16,383
Shard 1: hash values 16,384 – 32,767
Shard 2: hash values 32,768 – 49,151
Shard 3: hash values 49,152 – 65,535
```

### Lưu chọn Hash Function

```
KHÔNG cần cryptographically strong
CẦN: uniformly distributed (Murmur3, Fowler-Noll-Vo (FNV),...)

CẢNH BÁO: KHÔNG dùng built-in hash functions của programming languages!
   → Thường designed for in-memory hash tables
   → KHÔNG generate consistent values across processes/languages/versions
   → Java: hashCode() CÓ THỂ return DIFFERENT values trong different JVM instances
   → Python: hash() THAY ĐỔI mỗi lần interpreter restart (từ Python 3.3+)
   → → SẼ ASSIGN key đến WRONG SHARD sau restart!
```

---

## 2. Fixed Number of Shards

### Cách tiếp cận đơn giản

```
Create NHIỀU SHARDS hơn số nodes
   → Assign NHIỀU SHARDS per node

Ví dụ: 10 nodes → create 1,000 shards
   → Ban đầu: ~100 shards per node
   → Shard là ĐƠN VỊ cho rebalancing, KHÔNG PHẢI node

Khi node được THÊM VÀO:
   → Move vài shards sang node mới (Figure 7-4)
   → Balance load

Khi node bị XÓA:
   → Shards của nó phân tán sang nodes còn lại

Ví dụ thực tế: Riak, Elasticsearch, Couchbase, Voldemort, Redis Cluster
   Cassandra/ScyllaDB: tương tự nhưng shards được gọi là "virtual nodes" (vnodes)
```

### Ưu điểm

```
Rebalancing: chỉ cần MOVE ENTIRE SHARDS
   (không cần split/merge shard hay move individual records)
   → Overhead nhỏ hơn key range rebalancing

Backup/restore per shard → granular
```

### Nhược điểm

```
Phải CHỌN số lượng shards KHI KHỞI TẠO DATABASE
   → Số shard KHÔNG THAY ĐỔI sau đó (trừ khi toàn bộ database restructure)

KHÔNG PHÙ HỢP nếu không biết trước total data size:
   Quá ÍT shards: mỗi shard quá LỚN → rebalancing/recovery tốn kém
   Quá NHIỀU shards: overhead management cao (vì mỗi shard có resources riêng)

Best performance khi shard size "JUST RIGHT"
   → Khó đạt nếu số shards cố định mà data size thay đổi nhiều
```

---

## 3. Hash-Range Sharding (Kết hợp)

### Vấn đề với Pure Hash Sharding

```
Hash sharding ĐỀU tốt → BUT: phá vỡ ORDERING của keys
→ RANGE QUERIES KHÔNG HIỆU QUẢ:
   Keys gần nhau bị hash thành values khác nhau hoàn toàn
   → Scatter across shards
```

### Giải pháp: Hash-Range Sharding

```
Dùng hash để ASSIGN shard, nhưng VẪN SORT keys TRONG shard

Nếu key gồm nhiều columns:
   PARTITION KEY: column đầu tiên (hash để assign shard)
   SORT KEY: các columns còn lại (sorted trong cùng shard)

Kết quả: Records với CÙNG PARTITION KEY → CÙNG SHARD, sorted theo sort key
→ Range query trên sort key: RẤT HIỆU QUẢ (trong phạm vi 1 partition key)

Ví dụ: Cassandra table với primary key (user_id, update_timestamp)
   → Tất cả activity của 1 user trong cùng shard
   → Range query: "all updates by user X in time range" → 1 shard
```

### Số lượng Shards Adaptive

```
Hash-range sharding CHO PHÉP shard số lượng adapt dễ dàng:
   Shard quá lớn → SPLIT tại hash boundary
   → Flexible như key-range sharding

Nhược điểm so với key-range: Range queries TRÊN PARTITION KEY không hiệu quả
   (keys trong range bị scatter across all shards)
```

### Dùng bởi

```
YugabyteDB, DynamoDB: hash-range sharding
MongoDB: hash-based option
Cassandra, ScyllaDB: variant của hash-range (xem Figure 7-6)
```

### Data Warehouses và Cluster Columns

```
BigQuery: partition key → shard assignment; "cluster columns" → sort order trong shard
Snowflake: auto-assigns records đến "micro-partitions", định nghĩa cluster keys per table
Delta Lake: manual/automatic partition, hỗ trợ cluster keys

Clustering không chỉ cải thiện range scan performance:
   Còn cải thiện COMPRESSION và FILTERING performance
```

---

## 4. Consistent Hashing

### Định nghĩa

```
Consistent hashing: hash function map keys đến số shards sao cho:
   1. Số keys mapped đến mỗi shard: ROUGHLY EQUAL
   2. Khi số shards THAY ĐỔI: ÍT KEYS NHẤT CÓ THỂ phải move

"Consistent" ở đây KHÔNG liên quan đến:
   - Replica consistency (Chapter 6)
   - ACID consistency (Chapter 8)
   → Chỉ nói về xu hướng key CÒN Ở CÙNG SHARD khi số shards thay đổi
```

### Cassandra/ScyllaDB Variant (Figure 7-6)

```
Space of hash values (0–1024) → chia thành NHIỀU RANGES với RANDOM BOUNDARIES

Default: Cassandra: 16 ranges/node; ScyllaDB: 256 ranges/node

Mỗi node = nhiều ranges (tokens) → imbalances giữa các ranges TEND TO EVEN OUT

Khi thêm node 3:
   Node 1 chuyển một phần ranges → Node 3
   Node 2 chuyển một phần ranges → Node 3
   → New node có APPROXIMATELY FAIR SHARE, without moving more than necessary
```

### Các Consistent Hashing Algorithms khác

| Algorithm                                  | Mô tả                                    |
| ------------------------------------------ | ---------------------------------------- |
| Original consistent hashing (Karger, 1997) | Ring-based, tương tự Cassandra/ScyllaDB  |
| Highest random weight (Rendezvous hashing) | Mỗi key → node với highest weighted hash |
| Jump consistent hashing                    | Fast, minimal memory                     |

```
Với consistent hashing: thay vì split existing shard thành subranges,
   new node được assigned INDIVIDUAL KEYS scattered across all other nodes
   → Khác với approach của Cassandra/ScyllaDB
   → Cái nào phù hợp hơn: depends on application
```

---

## 5. Skewed Workloads và Hot Keys

### Vấn đề: Hash không đảm bảo even load

```
Consistent hashing đảm bảo keys uniformly distributed
NHƯNG không đảm bảo ACTUAL LOAD uniformly distributed!

HOT KEY: Key có REQUEST RATE cao hơn hẳn
   Ví dụ: Justin Bieber có 3% Twitter servers dedicated for him
   Celebrity post → storm of reads/writes đến SAME KEY
   → HOT SHARD ngay cả khi data size nhỏ
```

### Giải pháp Flexible Sharding Policy

```
1. PUT HOT KEY IN ITS OWN SHARD:
   (Even dedicated machine)

2. APPLICATION-LEVEL COMPENSATION:
   Key đã biết là hot → append RANDOM NUMBER vào key

   Ví dụ: key "bieber" → "bieber_00" đến "bieber_99" (2 digits random)
   → Writes spread đều qua 100 keys → 100 shards khác nhau

   NHƯNG: Reads phải đọc từ TẤT CẢ 100 keys → COMBINE results
   → Volume of reads per shard KHÔNG giảm, chỉ write load được split

3. BOOKKEEPING needed:
   Hot key mechanism chỉ áp dụng cho ÍT keys hot
   → Cần track: keys nào đang split
   → Process để convert regular key → specially managed hot key

4. CHANGING LOAD OVER TIME:
   Viral post: hot vài ngày → calm down
   Some keys hot for writes, others hot for reads → different strategies

5. AUTOMATED APPROACH (cloud services):
   Amazon "heat management" / "adaptive capacity" cho DynamoDB
   Details beyond scope of book
```

---

## Tóm tắt phần này

```
HASH SHARDING:
   Hash(key) → hash range → shard
   ✓ Phân phối đều qua nodes
   ✗ Range queries không hiệu quả

FIXED NUMBER OF SHARDS:
   Nhiều shards per node, move entire shards khi rebalance
   ✓ Simple rebalancing
   ✗ Phải chọn số shards ban đầu, khó adjust nếu data size thay đổi nhiều

HASH-RANGE SHARDING (Best of both worlds):
   Hash key → assign shard; SORT các column còn lại trong shard
   ✓ Range queries hiệu quả cho sort key (trong cùng partition key)
   ✓ Số shards adaptive
   ✗ Range queries trên partition key không hiệu quả

CONSISTENT HASHING:
   Min key movement khi số shards thay đổi
   Cassandra/ScyllaDB: nhiều ranges/node → balances even out

HOT KEYS:
   Hash không giải quyết hot keys (high request rate, ít keys)
   Solution: Random suffix, dedicated shard, cloud automated heat management
```
