# Summary & Key Concepts — Chapter 7

---

## 1. Tổng hợp toàn chương

Chương 7 giải quyết câu hỏi: **Khi một node không đủ khả năng xử lý toàn bộ data và write throughput, làm sao chia nhỏ data ra nhiều nodes?**

### Nguyên tắc cốt lõi

```
MỤC TIÊU: Spread data và query load EVENLY qua nhiều nodes
TRÁNH: Hot spots (nodes disproportionately high load)

SHARDING ≠ REPLICATION:
   Replication: SAME data on MULTIPLE nodes (fault tolerance)
   Sharding: DIFFERENT data on DIFFERENT nodes (scalability)
   Thường COMBINE cả hai: mỗi shard được replicate
```

### 2 Chiến lược Sharding Chính

```
KEY RANGE SHARDING:
   ✓ Range queries hiệu quả (sorted keys trong shard)
   ✓ Adaptive số lượng shards
   ✗ Hot spots khi keys adjacent (timestamp)
   ✗ Shard split expensive

HASH SHARDING:
   ✓ Phân phối đều (hash function → uniform distribution)
   ✗ Range queries không hiệu quả
   Variants: Fixed number of shards, Hash-range, Consistent hashing

HASH-RANGE (Best of both):
   Partition key → hash → shard; sort key → sort trong shard
   ✓ Uniform distribution + range queries trên sort key
```

### 3 Request Routing Approaches

```
1. Any node + forwarding (Cassandra, Riak)
2. Routing tier / shard-aware LB (MongoDB mongos)
3. Shard-aware client (direct connection)

Coordination: ZooKeeper/etcd, Raft built-in, hoặc gossip (Riak)
```

### Secondary Indexes

```
Local: per-shard, writes easy, reads = scatter-gather
Global (term-partitioned): reads easy (1 index shard), writes complex
```

---

## 2. Bảng thuật ngữ (Glossary)

### Khái niệm cơ bản

| Thuật ngữ                     | Định nghĩa                                               |
| ----------------------------- | -------------------------------------------------------- |
| **Shard / Partition**         | Một phần của dataset trên 1 node                         |
| **Partition key / Shard key** | Column/field xác định shard chứa record                  |
| **Sort key / Clustering key** | Column xác định thứ tự trong shard (hash-range sharding) |
| **Hot shard**                 | Shard có load disproportionately cao                     |
| **Hot key**                   | Key có request rate cực cao                              |
| **Skewed workload**           | Workload không đều, một số keys hot hơn nhiều            |
| **Scatter-gather query**      | Query phải gửi đến TẤT CẢ shards và combine results      |
| **Pre-splitting**             | Configure initial set of shards trước khi có data        |
| **Rebalancing**               | Move shards giữa nodes khi cluster thay đổi              |
| **Multitenancy**              | Nhiều customers/tenants dùng cùng infrastructure         |
| **Cell-based architecture**   | Nhóm services + storage cho 1 set of tenants vào 1 cell  |

### Tên gọi Shard theo Database

| Database                       | Tên gọi Shard                           |
| ------------------------------ | --------------------------------------- |
| Kafka                          | Partition                               |
| CockroachDB                    | Range                                   |
| HBase, TiDB                    | Region                                  |
| Couchbase                      | vBucket                                 |
| Riak                           | vNode                                   |
| Cassandra                      | Token-range / vNode                     |
| Bigtable, YugabyteDB, ScyllaDB | Tablet                                  |
| PostgreSQL                     | Partition (local) / Shard (distributed) |

### Sharding Strategies

| Thuật ngữ                   | Định nghĩa                                                  |
| --------------------------- | ----------------------------------------------------------- |
| **Key range sharding**      | Shard = contiguous range of keys; range queries dễ          |
| **Hash sharding**           | Shard = range of hash values; phân phối đều                 |
| **Hash-range sharding**     | Partition key → hash → shard; sort key → sorted trong shard |
| **Fixed number of shards**  | Số shards cố định khi init; nhiều shards per node           |
| **Consistent hashing**      | Min key movement khi số shards thay đổi                     |
| **Rendezvous hashing**      | Highest random weight algorithm                             |
| **Jump consistent hashing** | Fast, minimal memory consistent hash                        |
| **Virtual nodes (vnodes)**  | Cassandra/ScyllaDB: nhiều token-ranges per node             |
| **Cluster columns**         | BigQuery/Snowflake: sort order trong partition              |

### Rebalancing Operations

| Thuật ngữ                 | Định nghĩa                                                |
| ------------------------- | --------------------------------------------------------- |
| **Automatic rebalancing** | System tự quyết định khi và như thế nào                   |
| **Manual rebalancing**    | Admin cấu hình shard assignment                           |
| **Shard split**           | Chia 1 shard thành 2+ smaller shards                      |
| **Shard merge**           | Gộp 2+ small adjacent shards thành 1                      |
| **Cascading failure**     | Overload → slow → assumed dead → more rebalancing → worse |
| **Proactive rebalancing** | Rebalance TRƯỚC khi expected traffic surge                |

### Request Routing

| Thuật ngữ                | Định nghĩa                                               |
| ------------------------ | -------------------------------------------------------- |
| **Request routing**      | Xác định node nào handle request cho 1 key               |
| **Routing tier**         | Shard-aware load balancer, không handle DB logic         |
| **Shard-aware client**   | Client biết shard assignment, connect trực tiếp          |
| **Coordination service** | ZooKeeper, etcd: lưu authoritative shard-to-node mapping |
| **Gossip protocol**      | Nodes disseminate cluster state đến nhau (Riak)          |
| **Cutover period**       | Transition period khi shard đang được moved              |
| **mongos**               | MongoDB routing daemon                                   |

### Secondary Indexes

| Thuật ngữ                      | Định nghĩa                                       |
| ------------------------------ | ------------------------------------------------ |
| **Local secondary index**      | Index chỉ cover records trong 1 shard; per-shard |
| **Document-partitioned index** | IR name cho local secondary index                |
| **Global secondary index**     | Index cover data từ TẤT CẢ shards                |
| **Term-partitioned index**     | IR name cho global secondary index               |
| **Postings list**              | List of record IDs matching 1 index entry        |
| **Scatter-gather**             | Query đến TẤT CẢ shards + combine results        |
| **Distributed transaction**    | Transaction spanning nhiều shards (Chapter 8)    |

---

## 3. Mindmap khái niệm

```
                            SHARDING
                               │
          ┌────────────────────┼──────────────────────┐
          │                    │                      │
   SHARDING STRATEGIES    REBALANCING &          SECONDARY
          │                ROUTING                  INDEXES
   ┌──────┴──────┐             │                      │
   │             │        ┌────┴────┐           ┌─────┴─────┐
KEY RANGE     HASH        │         │           │           │
   │             │    Auto vs   Request       Local       Global
 Sorted      Uniform    Manual    Routing    (per-shard) (term-partitioned)
 range       distribut     │         │           │           │
 queries     (no range     │    ┌────┴────┐    Writes    Writes
   │         queries)  Cascad  Any   Routing  simple    complex
   │             │     Failure node   Tier    Reads:    Reads:
   │         ┌───┴──┐     │     +     +     scatter-   1 index
 Hot spots  Hash  Consist  Proac  forw  Client   gather    shard
 (timestamp  Range ent Hash  tive  ard  (direct)    │           │
  key)       │   (Cassandra  Rebal │          Tail lat  Multi-condition
   │         │    ScyllaDB)   ance │          amplif.   → multiple
 Prefix    Fixed          ZooKeeper        shards
 with ID   Shards         etcd │
            │             Raft │
         Many shards      (built-in)
         per node         Gossip (Riak)
```

---

## 4. Câu hỏi tự kiểm tra (Self-Check Questions)

1. **Tại sao** cần Sharding? Phân biệt khi nào chỉ cần Replication vs cần Sharding.

2. Liệt kê **7 lợi ích** của per-tenant sharding trong multitenancy. Challenges nào có thể xảy ra?

3. Giải thích **Key Range Sharding**. Tại sao key ranges KHÔNG nhất thiết evenly spaced? Ví dụ về hot spot với timestamp keys và giải pháp.

4. Mô tả **quy trình shard split** trong key range sharding. Tại sao split lại expensive, đặc biệt khi shard đang hot?

5. Tại sao KHÔNG nên dùng **Java hashCode() hoặc Python hash()** cho sharding?

6. Giải thích **Fixed Number of Shards** approach. Ưu nhược điểm so với adaptive sharding.

7. Mô tả **Hash-Range Sharding**. Ví dụ cụ thể từ Cassandra (user_id, timestamp). Khi nào nó kém hiệu quả?

8. Giải thích **Consistent Hashing**. "Consistent" ở đây có nghĩa gì? Cassandra/ScyllaDB implement như thế nào?

9. Tại sao **hash sharding không đủ** để ngăn hot spots? Kỹ thuật **random suffix** giải quyết vấn đề gì và tạo ra vấn đề gì mới?

10. So sánh **Automatic vs Manual rebalancing**. Tại sao automatic rebalancing kết hợp với automatic failure detection có thể gây **cascading failure**?

11. Mô tả **3 approaches** cho Request Routing. Cái nào phù hợp với Cassandra, MongoDB, và HBase?

12. Giải thích vai trò của **ZooKeeper/etcd** trong shard assignment. Tại sao cần consensus protocol ở đây?

13. Tại sao **Riak** dùng gossip protocol chấp nhận được dù weaker consistency?

14. Giải thích **Local Secondary Index**. Tại sao scatter-gather query không scale khi thêm shards?

15. Giải thích **Global Secondary Index (term-partitioned)**. Tại sao writes phức tạp hơn? DynamoDB handle stale global indexes như thế nào?

16. Khi nào nên chọn **Local** vs **Global** secondary index?

---

## 5. Liên kết tới các chương khác

| Chapter 7 đề cập                                  | Mở rộng ở                                  |
| ------------------------------------------------- | ------------------------------------------ |
| Shared-nothing architecture                       | **Chapter 2** — Scalability                |
| LSM-tree, B-tree trong từng shard                 | **Chapter 4** — Storage and Retrieval      |
| Replication kết hợp với sharding                  | **Chapter 6** — Replication                |
| Multi-object transactions cho global index writes | **Chapter 8** — Transactions               |
| ZooKeeper, etcd, Raft — consensus                 | **Chapter 10** — Consistency and Consensus |
| Parallel query execution qua nhiều shards         | **Chapter 11** — Batch Processing          |
| Tail latency amplification (scatter-gather)       | **Chapter 2** — Describing Performance     |
| Service discovery (tương tự request routing)      | **Chapter 5** — Dataflow Through Services  |

---

## 6. Trích dẫn mở đầu chương (Epigraph)

> _"Clearly, we must break away from the sequential and not limit the computers. We must state definitions and provide for priorities and descriptions of data. We must state relationships, not procedures."_
> — **Grace Murray Hopper**, Management and the Computer of the Future (1962)

→ Grace Hopper — một trong những pioneers của computer science — đã thấy từ 1962 rằng sức mạnh của máy tính nằm ở khả năng xử lý **data relationships** song song, không bị giới hạn vào sequential processing. Đây chính là triết lý của sharding: **phá vỡ giới hạn của 1 node bằng cách distribute data và process song song**.

---

## 7. Bảng so sánh Sharding Strategies

| Strategy               | Key Ordering       | Range Queries          | Distribution | Adaptive Shards  | Best For                                |
| ---------------------- | ------------------ | ---------------------- | ------------ | ---------------- | --------------------------------------- |
| **Key Range**          | Preserved          | ✓ Excellent            | Uneven risk  | ✓ (via split)    | Time-series, sorted data                |
| **Hash (Fixed)**       | Destroyed          | ✗ Poor                 | ✓ Uniform    | ✗ Fixed at init  | Write-heavy, unknown data size at init  |
| **Hash-Range**         | Sort key preserved | ✓ Within partition key | ✓ Uniform    | ✓ (via split)    | Multi-tenant apps (user_id + timestamp) |
| **Consistent Hashing** | Destroyed          | ✗ Poor                 | ✓ Uniform    | ✓ (min key move) | Dynamic cluster size                    |
