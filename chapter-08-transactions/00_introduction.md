# DDIA 2nd Edition — Chapter 8: Transactions

> **Sách:** Designing Data-Intensive Applications, 2nd Edition  
> **Tác giả:** Martin Kleppmann & Chris Riccomini  
> **Chương:** 8 — Transactions

---

## Mục tiêu chương

**Transactions** là một trong những **abstractions quan trọng nhất** trong database engineering — cho phép application "giả vờ" rằng một số lớp concurrency problems và hardware/software faults không tồn tại. Chương 8 đi sâu vào **ACID properties**, **isolation levels**, **concurrency control algorithms**, và **distributed transactions**.

> Epigraph mở đầu chương:
> _"Some authors have claimed that general two-phase commit is too expensive to support, because of the performance or availability problems that it brings. I say these authors do not know what they are talking about."_
> — James Gray et al., Transaction recovery and consistency (1976)

---

## Cấu trúc tài liệu này

| File                                        | Nội dung                                                                              |
| ------------------------------------------- | ------------------------------------------------------------------------------------- |
| `01_ACID_and_Single_Object_Transactions.md` | ACID properties, single-object writes, multi-object transactions, savepoints          |
| `02_Weak_Isolation_Levels.md`               | Read Committed, Snapshot Isolation (MVCC), read skew, lost updates                    |
| `03_Write_Skew_and_Phantoms.md`             | Write skew pattern, phantoms, meeting room booking, SELECT FOR UPDATE                 |
| `04_Serializability.md`                     | Serial execution, stored procedures, 2PL, predicate locks, SSI                        |
| `05_Distributed_Transactions.md`            | 2PC, coordinator failure, XA transactions, exactly-once, DB-internal vs heterogeneous |
| `06_Summary_and_Key_Concepts.md`            | Tổng hợp toàn chương, glossary, bảng isolation levels, mindmap, câu hỏi ôn tập        |

---

## Big Picture — Chapter 8

```
┌──────────────────────────────────────────────────────────┐
│                    TRANSACTIONS                           │
│                                                            │
│  ACID = Atomicity, Consistency, Isolation, Durability    │
│                                                            │
│  ISOLATION LEVELS (weak → strong):                        │
│  Read Uncommitted → Read Committed → Snapshot → Serializable│
│                                                            │
│  SERIALIZABILITY implementations:                          │
│  Serial execution | 2PL (pessimistic) | SSI (optimistic)  │
│                                                            │
│  DISTRIBUTED TRANSACTIONS:                                 │
│  Single-node atomic commit → 2PC (coordinator + prepare)  │
│  XA (heterogeneous, problematic) vs DB-internal (better)  │
└──────────────────────────────────────────────────────────┘
```

---

## Bảng tóm tắt Isolation Levels (Table 8-1)

| Isolation level    | Dirty reads | Read skew   | Phantom reads | Lost updates | Write skew  |
| ------------------ | ----------- | ----------- | ------------- | ------------ | ----------- |
| Read uncommitted   | ✗ Possible  | ✗ Possible  | ✗ Possible    | ✗ Possible   | ✗ Possible  |
| Read committed     | ✓ Prevented | ✗ Possible  | ✗ Possible    | ✗ Possible   | ✗ Possible  |
| Snapshot isolation | ✓ Prevented | ✓ Prevented | ✓ Prevented   | ? Depends    | ✗ Possible  |
| Serializable       | ✓ Prevented | ✓ Prevented | ✓ Prevented   | ✓ Prevented  | ✓ Prevented |

---

## Các chương liên quan

- **Chapter 2:** Reliability — transactions là 1 trong những tools chính
- **Chapter 4:** WAL — nền tảng của durability và crash recovery
- **Chapter 6:** Replication — single-leader replication + transactions trên leader
- **Chapter 7:** Sharding — cross-shard transactions cần distributed transactions
- **Chapter 9:** Unreliable clocks và network — vấn đề fundamental trong distributed txns
- **Chapter 10:** Consensus — atomic commitment, coordinator replication
- **Chapter 12:** Stream processing — exactly-once semantics
