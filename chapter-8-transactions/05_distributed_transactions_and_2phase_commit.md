# Distributed Transactions và Two-Phase Commit

---

## 1. Vấn đề Atomic Commit trong Distributed Systems

### Single-Node Transaction Commit

```
Single node: atomicity của commit = ORDER in which data written to disk
   FIRST: transaction data (WAL)
   THEN: commit record

Nếu crash TRƯỚC commit record: → ABORT khi recover
Nếu crash SAU commit record: → COMMIT (data visible to others)

→ 1 device (disk controller) quyết định → ATOMIC
```

### Distributed Transaction Commit — Vấn đề

```
Distributed transaction: NHIỀU NODES tham gia
Không thể simply send commit request to all nodes + commit independently:

Ví dụ (Figure 8-12): Transaction write sang DB1 và DB2
   DB1 commits OK
   DB2: constraint violation → aborts
   → INCONSISTENCY: DB1 has committed, DB2 has aborted

Các lý do failure:
   - Constraint violation ở 1 node
   - Commit request bị mất trên network → timeout → abort
   - Node crash trước khi commit record written

Khi 1 node commit nhưng 1 node abort:
   KHÔNG THỂ retract commit (data already visible to others)
   → PHẢI ENSURE: tất cả nodes EITHER ALL COMMIT or ALL ABORT
   → Gọi là: ATOMIC COMMITMENT PROBLEM
```

---

## 2. Two-Phase Commit (2PC)

### Tổng quan

```
2PC: Algorithm đảm bảo atomic commit qua MULTIPLE NODES
   Không nhầm với 2PL (Two-Phase Locking)!

Thành phần mới: COORDINATOR (transaction manager)
   Thường là library trong application process
   Ví dụ: Narayana, JOTM, BTM, MSDTC
```

### Phase 1: PREPARE

```
1. Application yêu cầu commit
2. Coordinator gửi PREPARE REQUEST đến TẤT CẢ participant nodes
3. Mỗi participant: check có thể commit không
   - Write tất cả transaction data to disk (kể cả crash sau, phải commit được)
   - Check constraints, conflicts
   - Reply YES (có thể commit) hoặc NO (abort)
4. Coordinator track responses
```

### Phase 2: COMMIT hoặc ABORT

```
Nếu ALL YES:
   → Coordinator ghi COMMIT DECISION vào transaction log (durable)
   → Gửi COMMIT request đến all participants
   → Participants commit và release locks

Nếu ANY NO:
   → Coordinator ghi ABORT DECISION
   → Gửi ABORT request đến all participants

Nếu commit request fail/timeout:
   → Coordinator RETRY FOREVER đến khi success
   (Participant đã vote YES → phải commit khi được yêu cầu)
```

### Hệ thống Promises — Tại sao 2PC đảm bảo Atomicity

```
2 điểm "không thể quay lại":

1. Participant vote YES:
   → Hứa sẽ commit nếu được yêu cầu, BẤT KỂ HOÀN CẢNH nào
   → PHẢI write all data durably trước khi reply YES
   → Từ bỏ quyền abort unilaterally

2. Coordinator ghi commit decision to disk (COMMIT POINT):
   → Quyết định IRREVOCABLE
   → PHẢI enforce, dù cần bao nhiêu retries

→ "Marriage ceremony": Cả 2 nói "I do" → cannot retract
   Nếu ngất sau "I do": khi tỉnh dậy → đã kết hôn (query transaction ID)
```

---

## 3. Coordinator Failure — Vấn đề Lớn Nhất của 2PC

### Tình huống

```
Coordinator gửi PREPARE → participant vote YES
NHƯNG coordinator CRASHES TRƯỚC KHI gửi COMMIT/ABORT

→ Participants: đã vote YES, KHÔNG THỂ abort unilaterally
   (chỉ coordinator mới có quyền quyết định)
   → Participants CHỜ ("in doubt" hoặc "uncertain")

Dù timeout → participants KHÔNG THỂ self-resolve:
   Unilaterally commit: có thể conflict với participant khác abort
   Unilaterally abort: có thể conflict với participant khác commit

→ Chỉ coordinator recovery mới giải quyết được
```

### Holding Locks While In Doubt

```
Participants đang giữ LOCKS trên modified rows
→ CANNOT RELEASE until transaction commits/aborts
→ Other transactions muốn access same rows: BLOCKED

Nếu coordinator crash và mất 20 phút để restart:
   → Locks held FOR 20 MINUTES

Nếu coordinator log LOST:
   → Locks held FOREVER (or until manual intervention)

→ Large parts of application UNAVAILABLE (anyone accessing those rows blocked)
```

### Recovery

```
Coordinator restart → read transaction log → resolve in-doubt transactions:
   Transaction có commit record → COMMIT (retry if needed)
   Transaction không có commit record → ABORT

NHƯNG: Orphaned in-doubt transactions (log lost/corrupted):
   Mặc dù coordinator restart → KHÔNG THỂ auto-resolve
   → Sit forever in database, holding locks
   → PHẢI ADMIN MANUALLY decide: commit hay rollback
   (Under high stress during production outage!)

Heuristic decisions: Escape hatch trong XA
   Participant unilaterally decide (break atomicity guarantee)
   → "Heuristic" = euphemism for "probably violating atomicity"
   → Only for catastrophic situations
```

---

## 4. Three-Phase Commit (3PC) — Lý thuyết

```
3PC: Proposed alternative để solve blocking problem của 2PC

NHƯNG: 3PC assumes bounded network delay + bounded node response times
→ Trong practice: unbounded delays (Chapter 9) → 3PC KHÔNG guarantee atomicity

Better solution: Replace single-node coordinator với FAULT-TOLERANT consensus
   → Chapter 10 (Paxos, Raft)
```

---

## 5. Distributed Transactions Across Different Systems

### 2 Loại Distributed Transactions

#### Database-Internal Distributed Transactions

```
Tất cả nodes: cùng database software
   → Không cần compatible với other systems
   → Có thể dùng better protocols (không phải lowest common denominator)

Ví dụ: CockroachDB, TiDB, FoundationDB, Spanner, VoltDB, MySQL Cluster NDB
        Kafka (internal transactions)

→ Thường WORK QUITE WELL
   Coordinator replicated (automatic failover)
   Coordinator + shards communicate directly
   Coupled với distributed concurrency control (deadlock detection, consistent reads)
```

#### Heterogeneous Distributed Transactions (XA Transactions)

```
Participants: 2+ different technologies (different DB vendors, message brokers,...)
   → Must ensure atomic commit dù systems entirely different under the hood
   → MUCH MORE CHALLENGING
```

### XA Transactions (eXtended Architecture)

```
XA: Standard 1991 cho implementing 2PC across heterogeneous technologies
   Supported: PostgreSQL, MySQL, Db2, SQL Server, Oracle
   Message brokers: ActiveMQ, HornetQ, MSMQ, IBM MQ
   Java: JTA (Java Transaction API), JDBC, JMS

XA = C API (not a network protocol)
   Application driver calls XA API
   Driver asks: "Is this operation part of distributed transaction?"
   If yes: sends necessary info to DB server

Coordinator: Library in application process (NOT separate service)
   Keeps track of participants
   Log on LOCAL DISK: commit/abort decisions
```

### Vấn đề của XA Transactions

| Vấn đề                              | Chi tiết                                                              |
| ----------------------------------- | --------------------------------------------------------------------- |
| **SPOF: Application = coordinator** | Coordinator crash → participants stuck in doubt                       |
| **SPOF: Application log**           | Log lost → in-doubt transactions, orphaned locks, manual intervention |
| **Lowest common denominator**       | Cannot detect deadlocks across systems, doesn't work with SSI         |
| **Application code as bottleneck**  | Coordinator và participants communicate ONLY VIA application code     |
| **Không phải network protocol**     | Coordinator cannot contact participants directly                      |

```
"These problems are somewhat inherent in performing transactions across heterogeneous technologies"

Many cloud services CHOOSE NOT TO implement distributed transactions
   vì operational problems
```

---

## 6. Exactly-Once Semantics

### Vấn đề: Message Processing + DB Write

```
Consumer receives message từ broker → process → write to DB + acknowledge
Crash GIỮA CHỪNG → message được redelivered → DOUBLE PROCESSING

Với 2PC (XA): Atomically commit message acknowledgment + DB write
→ Ensures exactly-once processing

NHƯNG: Chỉ nếu ALL SYSTEMS support same atomic commit protocol
   Email được gửi như side effect: email server không support 2PC
   → Email có thể gửi 2+ lần
```

### Exactly-Once WITHOUT Distributed Transactions

```
Alternative: Chỉ cần transactions WITHIN THE DATABASE

Cách làm:
1. Mỗi message có UNIQUE ID
2. DB có table: processed_message_ids
3. Khi nhận message: BEGIN transaction
4. Check: message ID đã processed chưa?
   → Yes: acknowledge + drop (không process lại)
   → No: add to processed_ids, process message, commit
5. Sau commit: acknowledge message to broker
6. After successful acknowledgement: delete message ID (separate tx)

Nếu crash TRƯỚC commit → tx aborted → broker retries → retry thấy ID → drop
Nếu crash SAU commit TRƯỚC ack → retry → thấy ID → drop (idempotent)
→ EXACTLY-ONCE đạt được chỉ với single-DB transactions!

→ Database-internal distributed transactions vẫn hữu ích:
   Cho phép message IDs ở 1 shard, main data ở shard khác
   → Atomic cross-shard commit
```

---

## Tóm tắt phần này

```
DISTRIBUTED TRANSACTIONS:
   Problem: All nodes MUST either all commit or all abort (atomic commitment)

2PC (Two-Phase Commit):
   Phase 1: PREPARE (participants check, vote yes/no)
   Phase 2: COMMIT/ABORT (coordinator decides, retries forever)

   2 points of no return:
   - Participant vote YES → cannot abort unilaterally
   - Coordinator writes COMMIT POINT → irrevocable

   PROBLEMS:
   - Coordinator crash → participants "in doubt" → hold locks FOREVER
   - Orphaned transactions → manual admin intervention
   - Blocking protocol (unlike 3PC, but 3PC doesn't work in practice)

XA TRANSACTIONS (heterogeneous):
   Lowest common denominator
   Application = coordinator = SPOF
   Cannot detect cross-system deadlocks
   Doesn't work with SSI
   Many cloud services avoid XA

DB-INTERNAL DISTRIBUTED TRANSACTIONS:
   Same software → better protocols, coordinator replication
   CockroachDB, TiDB, FoundationDB, Spanner → work well

EXACTLY-ONCE WITHOUT DISTRIBUTED TRANSACTIONS:
   Idempotent message processing via unique message IDs
   Single-DB transactions sufficient!
   → Avoid XA complexity
```
