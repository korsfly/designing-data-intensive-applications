# DDIA 2nd Edition — Chapter 7: Sharding

> **Sách:** Designing Data-Intensive Applications, 2nd Edition  
> **Tác giả:** Martin Kleppmann & Chris Riccomini  
> **Chương:** 7 — Sharding

---

## Mục tiêu chương

**Sharding** (còn gọi là **partitioning**) = chia nhỏ dataset thành các **shard/partition**, lưu trên các nodes khác nhau. Kỹ thuật này cần thiết khi **một node không đủ khả năng** xử lý toàn bộ data hoặc write throughput.

> Epigraph mở đầu chương:
> _"Clearly, we must break away from the sequential and not limit the computers. We must state definitions and provide for priorities and descriptions of data. We must state relationships, not procedures."_
> — Grace Murray Hopper, Management and the Computer of the Future (1962)

---

## Sharding vs Replication

```
REPLICATION (Chapter 6):  Copy of SAME data on NHIỀU NODES
SHARDING (Chapter 7):     DIFFERENT data on DIFFERENT NODES

→ Thường COMBINE cả hai: mỗi shard được replicate sang nhiều nodes
  (fault tolerance + scalability)
```

---

## Cấu trúc tài liệu này

| File                                         | Nội dung                                                                  |
| -------------------------------------------- | ------------------------------------------------------------------------- |
| `01_Why_Sharding_and_Key_Range.md`           | Tại sao cần sharding, sharding by key range, hot spots, rebalancing       |
| `02_Hash_Sharding_and_Consistent_Hashing.md` | Sharding by hash, fixed shards, hash-range, consistent hashing, hot keys  |
| `03_Operations_and_Request_Routing.md`       | Auto vs manual rebalancing, routing tier, ZooKeeper/etcd, gossip protocol |
| `04_Secondary_Indexes_and_Sharding.md`       | Local vs global secondary indexes, scatter-gather queries                 |
| `05_Summary_and_Key_Concepts.md`             | Tổng hợp toàn chương, glossary, mindmap, câu hỏi ôn tập                   |

---

## Big Picture

```
┌──────────────────────────────────────────────────────────┐
│                       SHARDING                            │
│                                                            │
│  MỤC TIÊU: Phân phối data và query load đều qua nodes    │
│  TRÁNH: Hot spots (nodes disproportionately high load)    │
│                                                            │
│  2 CHIẾN LƯỢC CHÍNH:                                       │
│  ┌─────────────────────┬──────────────────────────┐       │
│  │  KEY RANGE SHARDING  │    HASH SHARDING          │      │
│  │  + Range queries dễ  │  + Phân phối đều         │      │
│  │  - Hot spots risk    │  - Range queries khó     │      │
│  │  - Shard split đắt  │  - Fixed vs adaptive     │      │
│  └─────────────────────┴──────────────────────────┘       │
│                                                            │
│  SECONDARY INDEXES:                                        │
│  Local (per-shard) vs Global (term-partitioned)            │
└──────────────────────────────────────────────────────────┘
```

---

## Tên gọi khác nhau của Shard

| Database                       | Tên gọi                                 |
| ------------------------------ | --------------------------------------- |
| Kafka                          | Partition                               |
| CockroachDB                    | Range                                   |
| HBase, TiDB                    | Region                                  |
| Couchbase                      | vBucket                                 |
| Riak                           | vNode                                   |
| Cassandra                      | Token-range                             |
| Bigtable, YugabyteDB, ScyllaDB | Tablet                                  |
| PostgreSQL                     | Partition (local) / Shard (distributed) |

---

## Các chương liên quan

- **Chapter 2:** Scalability — horizontal scaling, shared-nothing architecture
- **Chapter 4:** Storage engines (B-trees, LSM-trees trong từng shard)
- **Chapter 6:** Replication — kết hợp với sharding
- **Chapter 8:** Transactions — cross-shard transactions cần distributed transactions
- **Chapter 10:** Consensus — ZooKeeper, etcd, Raft cho shard assignment coordination
- **Chapter 11:** Batch processing — parallel query execution qua nhiều shards
