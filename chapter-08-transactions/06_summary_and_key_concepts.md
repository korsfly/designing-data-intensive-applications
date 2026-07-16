# Summary & Key Concepts — Chapter 8

---

## 1. Tổng hợp toàn chương

Transactions cung cấp một **abstraction layer** quan trọng, cho phép application developer tập trung vào business logic thay vì phải xử lý toàn bộ các failure scenarios phức tạp của database systems.

### Luồng kiến thức chương 8

```
WHY TRANSACTIONS?
   → Concurrency + hardware/software faults → data inconsistencies
   → Transactions: "pretend" problems don't exist (abstraction)
        │
        ▼
ACID PROPERTIES:
   Atomicity: all-or-nothing, rollback on failure
   Consistency: application invariants (app's responsibility)
   Isolation: concurrent txns execute as if serial
   Durability: committed data not lost
        │
        ▼
ISOLATION LEVELS (weak → strong):
   Read Uncommitted → Read Committed → Snapshot → Serializable
        │
        ▼
CONCURRENCY ANOMALIES:
   Dirty reads → dirty writes → read skew → lost updates
   → write skew → phantoms
        │
        ▼
SERIALIZABILITY IMPLEMENTATIONS:
   Serial execution | 2PL (pessimistic) | SSI (optimistic)
        │
        ▼
DISTRIBUTED TRANSACTIONS:
   2PC (XA: problematic) | DB-internal (better)
   Exactly-once via idempotency (no distributed txn needed)
```

---

## 2. Bảng Isolation Levels và Anomalies (Table 8-1)

| Isolation Level        | Dirty Reads | Read Skew   | Phantom Reads | Lost Updates | Write Skew  |
| ---------------------- | ----------- | ----------- | ------------- | ------------ | ----------- |
| **Read Uncommitted**   | ✗ Possible  | ✗ Possible  | ✗ Possible    | ✗ Possible   | ✗ Possible  |
| **Read Committed**     | ✓ Prevented | ✗ Possible  | ✗ Possible    | ✗ Possible   | ✗ Possible  |
| **Snapshot Isolation** | ✓ Prevented | ✓ Prevented | ✓ Prevented   | ? Depends    | ✗ Possible  |
| **Serializable**       | ✓ Prevented | ✓ Prevented | ✓ Prevented   | ✓ Prevented  | ✓ Prevented |

---

## 3. Bảng thuật ngữ (Glossary)

### ACID

| Thuật ngữ            | Định nghĩa                                                                      |
| -------------------- | ------------------------------------------------------------------------------- |
| **Transaction**      | Group of reads/writes treated as 1 unit (all-or-nothing)                        |
| **ACID**             | Atomicity, Consistency, Isolation, Durability                                   |
| **Atomicity**        | All operations succeed or all rolled back (not partially applied)               |
| **Consistency**      | Application invariants maintained (app's responsibility, not DB)                |
| **Isolation**        | Concurrent transactions don't interfere — appears serial                        |
| **Durability**       | Committed data persists despite hardware/software failure                       |
| **BASE**             | Basically Available, Soft state, Eventually consistent (vague contrast to ACID) |
| **Abort / Rollback** | Undo all writes of a transaction                                                |
| **Savepoint**        | Named point within transaction for partial rollback                             |
| **Retry**            | Re-execute aborted transaction (must handle duplicates/side effects carefully)  |

### Concurrency Anomalies

| Thuật ngữ                          | Định nghĩa                                                                           |
| ---------------------------------- | ------------------------------------------------------------------------------------ |
| **Dirty read**                     | Reading uncommitted data from another transaction                                    |
| **Dirty write**                    | Overwriting uncommitted data from another transaction                                |
| **Read skew** (nonrepeatable read) | Seeing different parts of DB at different points in time                             |
| **Lost update**                    | Read-modify-write cycle: 1 write overwrites another without incorporating its change |
| **Write skew**                     | Both transactions read shared premise → write differently → combined state wrong     |
| **Phantom**                        | Write affects search condition of another transaction                                |
| **Race condition**                 | Outcome depends on timing of concurrent operations                                   |

### Isolation Levels and MVCC

| Thuật ngữ                                   | Định nghĩa                                                                      |
| ------------------------------------------- | ------------------------------------------------------------------------------- |
| **Read Committed**                          | No dirty reads, no dirty writes; most common default                            |
| **Snapshot Isolation**                      | Each transaction reads from consistent snapshot at start                        |
| **Repeatable Read**                         | Name used in MySQL for snapshot isolation                                       |
| **MVCC** (Multiversion Concurrency Control) | Multiple versions of each row; readers see old version while writes in progress |
| **consistent snapshot**                     | Snapshot showing only committed data as of transaction start                    |
| **Visibility rules**                        | Rules determining which version of row a transaction sees                       |
| **Transaction ID**                          | Monotonically increasing ID assigned to each transaction                        |
| **GC / Vacuum**                             | Process to delete old MVCC versions no longer needed                            |
| **Consistent snapshot**                     | View of DB at particular point in time                                          |
| **Atomic write operation**                  | Built-in operation that avoids read-modify-write cycle (increment, CAS)         |
| **Compare-and-set (CAS)**                   | Update only if value matches expected (e.g., `WHERE value = old`)               |
| **SELECT FOR UPDATE**                       | Lock rows returned by query to prevent concurrent modification                  |

### Serializability

| Thuật ngữ                                  | Định nghĩa                                                             |
| ------------------------------------------ | ---------------------------------------------------------------------- |
| **Serializability**                        | Strongest isolation: appears serial even if concurrent                 |
| **Serial execution**                       | Literally execute one transaction at a time, in sequence               |
| **Stored procedure**                       | Transaction logic stored in DB, executed without network round-trips   |
| **State machine replication**              | Same stored procedure executed on each replica (VoltDB)                |
| **Two-Phase Locking (2PL)**                | Writers block readers AND readers block writers; classic serializable  |
| **Shared lock**                            | Multiple readers can hold simultaneously                               |
| **Exclusive lock**                         | Only 1 writer; no other locks allowed                                  |
| **Deadlock**                               | A waiting for B, B waiting for A; DB detects + aborts 1                |
| **Predicate lock**                         | Lock on all objects matching search condition (even non-existing)      |
| **Index-range locking** (next-key locking) | Approximation of predicate lock attached to index entry                |
| **Serializable Snapshot Isolation (SSI)**  | Optimistic concurrency: proceed + check at commit → abort if conflict  |
| **Pessimistic concurrency control**        | 2PL: wait when potential danger detected                               |
| **Optimistic concurrency control**         | SSI: proceed + check at commit                                         |
| **Stale MVCC read**                        | Reading version that was already overwritten by committed transaction  |
| **Serialization conflict**                 | 2 transactions' reads/writes interact → if serial execution impossible |

### Distributed Transactions

| Thuật ngữ                             | Định nghĩa                                                                   |
| ------------------------------------- | ---------------------------------------------------------------------------- |
| **Distributed transaction**           | Transaction involving multiple nodes/systems                                 |
| **Atomic commitment problem**         | Ensuring all nodes either all commit or all abort                            |
| **Two-Phase Commit (2PC)**            | Protocol for atomic commit across multiple nodes                             |
| **Coordinator** (Transaction manager) | Component that orchestrates 2PC                                              |
| **Participant**                       | Database node in a distributed transaction                                   |
| **Prepare request**                   | Phase 1: coordinator asks participants if they can commit                    |
| **Commit point**                      | Moment coordinator writes commit decision to durable log (irrevocable)       |
| **In doubt** (Uncertain)              | Participant has voted YES, waiting for coordinator decision                  |
| **Orphaned transaction**              | In-doubt transaction whose coordinator cannot be resolved automatically      |
| **Heuristic decision**                | Participant unilaterally commits/aborts in-doubt tx (breaks atomicity)       |
| **Three-Phase Commit (3PC)**          | Non-blocking variant of 2PC (doesn't work in practice with unbounded delays) |
| **XA transaction**                    | Standard for heterogeneous 2PC (X/Open XA, 1991)                             |
| **JTA**                               | Java Transaction API — Java implementation of XA                             |
| **Database-internal transaction**     | Distributed txn where all nodes run same DB software                         |
| **Heterogeneous transaction**         | Distributed txn across different technologies                                |
| **Exactly-once semantics**            | Processing message exactly once despite failures                             |
| **Idempotent message processing**     | Recording message ID in DB → safe to retry                                   |

---

## 4. Mindmap khái niệm

```
                           TRANSACTIONS
                                │
          ┌─────────────────────┼──────────────────────┐
          │                     │                      │
        ACID             ISOLATION LEVELS          DISTRIBUTED
          │                     │                   TRANSACTIONS
   ┌──────┴──────┐       ┌──────┴──────┐                │
   │      │      │       │      │      │          ┌──────┴──────┐
 Atom  Consist  Isol   Read   Snap-   Serial-   2PC        DB-internal
   │      │     Dur    Comm   shot    izable      │          (better)
   │      │      │      │    (MVCC)    │          │
  WAL   App's   WAL   Dirty   Read   3 ways  Coordinator
  All-  respon  fsync  reads  skew    │       failure    XA (problems)
  or-   sibi-         pre-   pre-    │       "in doubt"
  noth  lity   Replic  vent   vent  Serial   holds locks
               ation   │       │    2PL   SSI    forever
                        │       │     │     │
                     Lost    Write  Pred  Optimistic
                    updates  skew  lock  check at
                      │      │   Range   commit
                    Atomic  FOR  lock     │
                     ops   UPDATE         Stale MVCC
                    CAS             Prior read detection
```

---

## 5. Câu hỏi tự kiểm tra (Self-Check Questions)

1. Giải thích **4 ACID properties**. Tại sao "C" thực ra là application's responsibility, không phải database's?

2. Phân biệt **Single-Object Writes** (atomic ops, CAS) và **Multi-Object Transactions** thực sự. Khi nào cần Multi-Object?

3. Giải thích **Dirty Read** và **Dirty Write**. Tại sao Dirty Write nguy hiểm hơn (ví dụ Alice/Bob buying car)?

4. Tại sao **Read Committed** KHÔNG ngăn **Read Skew**? Cho ví dụ cụ thể (Alice's bank accounts).

5. Giải thích **Snapshot Isolation** và **MVCC**. Transactions thấy "version" nào của data?

6. Tại sao cần cả **Snapshot Isolation** (không chỉ Read Committed) cho: backup, analytical queries, integrity checks?

7. Mô tả vấn đề **Lost Updates** với read-modify-write cycle. Liệt kê 5 cách giải quyết.

8. Giải thích **Write Skew** với ví dụ doctor on-call. Tại sao Snapshot Isolation KHÔNG đủ?

9. Tại sao **Phantom** khó hơn write skew để prevent? Tại sao `SELECT FOR UPDATE` không đủ cho Phantoms?

10. Mô tả **3 approaches to Serializability**: Serial Execution, 2PL, SSI. Pros/cons của mỗi approach?

11. Tại sao **Stored Procedures** là enabler cho Serial Execution? Pros/cons của stored procedures?

12. Giải thích **Two-Phase Locking (2PL)**. Tại sao nó có thể làm DB unavailable for writes?

13. Phân biệt **Predicate Locks** và **Index-Range Locks**. Cái nào efficient hơn và tại sao?

14. Giải thích **SSI** (optimistic): làm sao detect serialization conflicts? 2 cases nào cần detect?

15. Tại sao **2PL ≠ 2PC**? Tại sao nhầm lẫn này nguy hiểm?

16. Mô tả vấn đề **Atomic Commitment** trong distributed transactions. Tại sao không thể chỉ send commit to all và independently commit?

17. Giải thích **Two-Phase Commit (2PC)**: 2 phases, commit point, promises. Tại sao coordinator failure là "sticky situation"?

18. Tại sao **XA transactions** có nhiều vấn đề? Tại sao **DB-internal distributed transactions** tốt hơn?

19. Giải thích cách đạt **exactly-once semantics WITHOUT distributed transactions** (idempotent message processing).

20. Khi nào nên dùng: Serial Execution vs 2PL vs SSI?

---

## 6. Liên kết tới các chương khác

| Chapter 8 đề cập                           | Mở rộng ở                                            |
| ------------------------------------------ | ---------------------------------------------------- |
| WAL, crash recovery                        | **Chapter 4** — Storage and Retrieval                |
| Replication + transactions (single-leader) | **Chapter 6** — Replication                          |
| Cross-shard transactions                   | **Chapter 7** — Sharding                             |
| Distributed transactions (2PC, consensus)  | **Chapter 10** — Consistency and Consensus           |
| Unreliable clocks (for LWW timestamps)     | **Chapter 9** — The Trouble with Distributed Systems |
| Exactly-once message processing            | **Chapter 12** — Stream Processing                   |
| State machine replication (VoltDB)         | **Chapter 10**                                       |
| Post Office Horizon scandal                | **Chapter 2** — Reliability                          |

---

## 7. Trích dẫn mở đầu chương (Epigraph)

> _"Some authors have claimed that general two-phase commit is too expensive to support, because of the performance or availability problems that it brings. I say these authors do not know what they are talking about."_
> — **James Gray et al.**, Transaction recovery and consistency (1976)

→ Một lời nhắc nhở về **tension** đã tồn tại từ những ngày đầu của database research giữa **correctness** (transactional guarantees) và **performance** (throughput, availability). Cuộc tranh luận này vẫn tiếp tục đến ngày nay — SSI và NewSQL systems đang làm cho "expensive" ngày càng ít tốn kém hơn.
