# Tại sao Sharding và Key Range Sharding

---

## 1. Tại sao Cần Sharding?

### Primary Reason: Scalability

```
SHARDING là giải pháp khi:
   - Volume of data QUÁ LỚN cho 1 node
   - Write throughput QUÁ CAO cho 1 node

READ THROUGHPUT thấp? → Chỉ cần REPLICATION (Chapter 6), không cần sharding

Lý thuyết: 10 nodes = handle được 10x data và 10x read/write throughput
   của 1 node (bỏ qua replication overhead)
   → Chỉ đúng nếu sharding ĐỀU
```

### Single-Node Sharding (trong cùng 1 máy)

```
Một số systems dùng sharding ngay cả trên single machine:
   1 process per CPU core → tận dụng parallelism trong CPU
   HOẶC: NUMA architecture (some memory banks closer to some CPUs)

Ví dụ: Redis, VoltDB, FoundationDB → 1 process/core
→ Sharding spread load qua CPU cores trong cùng machine
```

---

## 2. Sharding for Multitenancy

### Định nghĩa

```
SaaS products: nhiều TENANTS (customers), mỗi tenant có dataset riêng biệt
→ Có thể dùng sharding để implement multitenant systems
```

### Cách tiếp cận

```
OPTION 1: Mỗi tenant = 1 separate shard
OPTION 2: Nhiều small tenants trong 1 shard lớn hơn
OPTION 3: Cell-based architecture (services + storage cho 1 set of tenants = 1 cell)
```

### 7 Lợi ích của Sharding cho Multitenancy

| Lợi ích                       | Giải thích                                                  |
| ----------------------------- | ----------------------------------------------------------- |
| **Resource isolation**        | 1 tenant làm expensive operation ít ảnh hưởng tenant khác   |
| **Permission isolation**      | Bug trong access control ít likely expose cross-tenant data |
| **Cell-based architecture**   | Fault trong 1 cell không lan sang cells khác                |
| **Per-tenant backup/restore** | Restore 1 tenant không ảnh hưởng tenant khác                |
| **GDPR compliance**           | Data export/deletion đơn giản (chỉ cần xử lý 1 shard)       |
| **Data residence**            | Assign shard của tenant vào specific region                 |
| **Gradual schema rollout**    | Migrate schema từng tenant → detect issues sớm              |

### Challenges

```
1. Mỗi tenant phải ĐỦ NHỎ để fit trên 1 node
   (nếu không → cần sharding TRONG tenant → phức tạp hơn)

2. Nhiều SMALL tenants → overhead nếu mỗi cái 1 shard
   (grouping nhiều small tenants → cần cơ chế move khi tenant grow)

3. Cross-tenant features → JOIN qua nhiều shards → KHÓ
```

---

## 3. Mục tiêu và Thách thức của Sharding

### Mục tiêu: Phân phối Đều

```
Mục tiêu: Spread data và query load EVENLY across nodes

Nếu KHÔNG đều → SKEWED:
   Một số shards có nhiều data/queries hơn → kém hiệu quả

HOT SHARD: Shard với load DISPROPORTIONATELY CAO
HOT KEY: Key với request rate CỰC CAO (vd: celebrity profile trong social network)
```

### Cần thuật toán: partition_key → shard_number

```
Input: partition key của record
Output: shard chứa record đó

Phải amenable to REBALANCING (giảm hot spots khi thêm/bớt nodes)
```

### Partition Key là gì?

```
Key-value store: partition key = key (hoặc phần đầu của key)
Relational model: partition key = 1 column trong table
   (KHÔNG nhất thiết phải là primary key)
```

---

## 4. Sharding by Key Range

### Nguyên tắc

```
Assign CONTIGUOUS RANGE của partition keys cho mỗi shard
→ Giống cuốn bách khoa toàn thư nhiều tập: mỗi tập chứa 1 khoảng chữ cái

Ví dụ (Figure 7-2): Encyclopedia
   Tập 1: A-B
   Tập 2: C-D
   ...
   Tập 12: T-Z (nhiều chữ cái vì ít entries hơn A-B!)

→ Key ranges KHÔNG nhất thiết evenly spaced:
   Data distribution ảnh hưởng → boundaries phải adapt
```

### Ưu điểm

```
RANGE QUERIES: Rất hiệu quả và dễ dàng
   → Scan sorted keys theo thứ tự
   → Shard biết keys nào thuộc về mình

KEY AS CONCATENATED INDEX:
   Fetch several related records trong 1 query

Ví dụ: Sensor network database, key = timestamp
   → Range scan cho tất cả readings trong 1 tháng:
     chỉ cần 1 shard (tháng đó) → HIỆU QUẢ
```

### Ai dùng Key Range Sharding?

```
Manual: Vitess (sharding layer cho MySQL)
Automatic: Bigtable, HBase, MongoDB (range-based option), CockroachDB,
           RethinkDB, FoundationDB
Manual + Automatic: YugabyteDB
```

### Nhược điểm: Hot Spots với Adjacent Keys

```
VẤN ĐỀ: Nếu key là TIMESTAMP, shards = time ranges
   Ví dụ: 1 shard per month
   → Tất cả WRITES vào CÙNG SHARD (tháng hiện tại) → OVERLOADED
     Các shard khác ngồi idle

GIẢI PHÁP: Prefix key với SENSOR ID (không dùng timestamp làm first element)
   Key = [sensor_id, timestamp]
   → Write load spread EVENLY qua sensors
   → NHƯNG: Muốn fetch values của NHIỀU SENSORS trong time range:
     Phải thực hiện SEPARATE RANGE QUERY cho mỗi sensor
```

---

## 5. Rebalancing Key Range Shards

### Pre-splitting

```
Một số databases (HBase, MongoDB) cho phép configure INITIAL SET OF SHARDS
trên empty database = PRE-SPLITTING

Yêu cầu: biết trước data distribution → chọn key range boundaries phù hợp
```

### Split khi Shard quá lớn/heavy

```
Khi data volume + write throughput tăng:
   Key range sharding GROWS bằng cách SPLIT shard thành 2+ smaller shards
   Mỗi shard giữ CONTIGUOUS SUBRANGE của key range gốc

Split trigger: Shard đạt configured size (HBase default: 10 GB)
   HOẶC: Write throughput vượt threshold (hot shard)
   → Hot shard có thể được split DÙ không có nhiều data

Kết quả: Smaller shards distribute qua nhiều nodes
Ngược lại: Nếu nhiều data bị delete → merge adjacent small shards
```

### Chi phí Split

```
Split shard: EXPENSIVE operation
   → Cần REWRITE ALL DATA vào new files (tương tự compaction trong LSM-tree)
   → Shard cần split thường ĐANG UNDER HIGH LOAD
   → Cost of splitting có thể EXACERBATE load → risk of overload

Số lượng shards: Adapts to DATA VOLUME
   Ít data → ít shards; nhiều data → nhiều shards
   Mỗi shard có configured max size
```

---

## Tóm tắt phần này

```
TẠI SAO SHARDING:
   Write throughput hoặc data volume vượt khả năng 1 node
   Đọc nhiều? → Replication đủ rồi, không cần sharding

MULTITENANCY:
   Mỗi tenant 1 shard → isolation, compliance, gradual rollout
   Challenges: tenant size limits, cross-tenant features, tenant movement

KEY RANGE SHARDING:
   ✓ Range queries hiệu quả (sorted keys trong shard)
   ✓ Adapt to data distribution
   ✗ Hot spots với adjacent key writes (ví dụ: timestamp key)
   ✗ Split expensive (rewrite + thêm load khi đang hot)

GIẢI PHÁP HOT SPOT: Thêm prefix (sensor ID) trước timestamp
   → Spread write load, nhưng range queries phức tạp hơn
```
