# Serializability: Serial Execution, 2PL, SSI

---

## 1. Serializability — Định nghĩa

```
SERIALIZABILITY: Isolation level MẠNH NHẤT

Đảm bảo: Kể cả transactions chạy song song (parallel/concurrent),
   RESULT giống như thể chúng chạy TUẦN TỰ (serially),
   mặc dù thứ tự không xác định

→ Ngăn TẤT CẢ anomalies:
   dirty reads, dirty writes, read skew, phantom reads, lost updates, write skew

3 cách implement:
   1. Literally execute serially (serial execution)
   2. Two-Phase Locking (2PL) — pessimistic
   3. Serializable Snapshot Isolation (SSI) — optimistic
```

---

## 2. Serial Execution — Đơn giản nhất

### Ý tưởng

```
Chỉ 1 TRANSACTION mỗi lần, trên 1 CPU CORE, theo thứ tự
→ KHÔNG CÓ concurrency → KHÔNG CÓ concurrency problems
→ Dễ implement, dễ reason about

Tại sao hiện đại này là viable?
   RAM giảm giá → Toàn bộ dataset FIT IN MEMORY
   → Không cần disk I/O (bottleneck cũ)
   → Transactions rất nhanh

Ví dụ: VoltDB, Datomic, Redis (single-threaded)
```

### Stored Procedures — Key Enabler

```
Interactive multi-statement transaction (Figure 8-8):
   Application: Begin tx
   DB: SELECT ...
   App: process result (may wait for user input!)
   App: UPDATE ...
   DB: ok
   App: Commit

→ SLOW: nhiều network round-trips giữa app và DB
   Nếu chờ USER INPUT trong transaction = giữ lock rất lâu = TERRIBLE

STORED PROCEDURE (Figure 8-9):
   Application submit toàn bộ TRANSACTION CODE lên DB (1 round-trip)
   DB execute locally, không cần round-trips
   → FAST enough cho serial execution

VoltDB: Java/Groovy stored procedures
Datomic: Java/Clojure
Redis: Lua scripts
MongoDB: JavaScript
```

### Pros và Cons của Stored Procedures

| Pros                                              | Cons                                                       |
| ------------------------------------------------- | ---------------------------------------------------------- |
| Transactions rất fast                             | DB-specific language (PL/SQL, PL/pgSQL,...)                |
| Phù hợp với serial execution                      | Khó debug, monitor, version control                        |
| Useful khi GraphQL/proxy không support validation | DB là performance-sensitive target → bad procs gây lớn hơn |
| State machine replication (VoltDB)                | Security risk trong multitenant systems                    |

```
VoltDB: Stored procedures phải DETERMINISTIC
   (khi replicated: cùng stored proc trên mỗi replica)
   → Dùng deterministic APIs cho time, random,...
   → Gọi là "STATE MACHINE REPLICATION" (Chapter 10)
```

### Sharding với Serial Execution

```
Single-threaded → throughput limited bởi 1 CPU core

SOLUTION: SHARD data (Chapter 7)
   Mỗi shard có 1 transaction processing thread
   → Throughput scale linearly với số CPU cores

   NHƯNG: Transaction cần access NHIỀU SHARDS
   → CROSS-SHARD COORDINATION cần thiết
   → Stored proc phải run in lockstep across all shards
   → Cross-shard transactions MUCH SLOWER:
     VoltDB: ~1,000 cross-shard writes/sec
     (orders of magnitude below single-shard throughput)
     → Cannot be increased by adding more machines
```

### Khi Serial Execution phù hợp?

```
Điều kiện:
   ✓ Transactions nhỏ và nhanh (stored procedures)
   ✓ Active dataset fit in memory
   ✓ Write throughput thấp hoặc có thể shard
   ✓ Cross-shard transactions hiếm hoặc không cần
```

---

## 3. Two-Phase Locking (2PL)

### Nguyên tắc cơ bản

```
Mạnh hơn "no dirty writes" locks (chỉ lock khi write):
   2PL: READERS ALSO BLOCK WRITERS, WRITERS ALSO BLOCK READERS

Rules:
   - Muốn READ object → phải acquire lock SHARED MODE
   - Muốn WRITE object → phải acquire lock EXCLUSIVE MODE
   - Nếu có exclusive lock → tất cả shared + exclusive phải wait
   - Nếu có shared lock → exclusive phải wait (shared OK)
   - Đọc rồi write: upgrade shared → exclusive

"Two-phase" = growing phase (acquire locks) + shrinking phase (release ALL locks khi end)
   → KHÔNG release lock nào rồi acquire lock mới
```

```
CONTRASTED với snapshot isolation:
   SI: "writers don't block readers, readers don't block writers"
   2PL: "writers DO block readers, readers DO block writers"
```

### Deadlocks

```
Transaction A giữ lock X, chờ lock Y
Transaction B giữ lock Y, chờ lock X
→ DEADLOCK: không ai tiến được

Database AUTO-DETECT deadlocks → ABORT 1 transaction
→ Application phải RETRY aborted transaction

2PL làm deadlocks MORE COMMON (nhiều locks hơn)
→ Thêm wasted effort khi retry
```

### Performance của 2PL

```
DOWNSIDE: Transaction throughput và response time SIGNIFICANTLY WORSE
   1. Overhead acquire/release locks
   2. Reduced concurrency (any potential race condition → 1 must wait)

   Ví dụ: Transaction reads entire table (backup, analytical query):
   → Phải take shared lock trên ENTIRE TABLE
   → Tất cả writes vào table phải WAIT cho đến khi read-only tx commit
   → DB effectively UNAVAILABLE for writes trong thời gian dài

   → "Unstable latencies, very slow at high percentiles nếu có contention"
   → 1 slow/heavy transaction có thể làm system "grind to a halt"
```

### Predicate Locks (cho Phantoms)

```
Lock không phải 1 object cụ thể mà là TẤT CẢ objects khớp condition:

SELECT * FROM bookings
WHERE room_id = 123 AND end_time > '12:00' AND start_time < '13:00';

→ Predicate lock: lock TẤT CẢ bookings cho room 123 trong time window đó
   Kể cả bookings CHƯA TỒN TẠI nhưng có thể được INSERT sau!
   → Ngăn phantoms
```

### Index-Range Locking (Hiệu quả hơn Predicate Locks)

```
Predicate locks: checking for matching locks = TIME-CONSUMING nếu nhiều locks

SIMPLIFICATION: Khóa một SET OBJECTS LỚN HƠN (nhưng safe)
   Thay vì lock "room 123, noon to 1pm":
   Option A: Lock TẤT CẢ bookings của room 123 (bất kể time)
   Option B: Lock TẤT CẢ rooms trong time slot noon to 1pm
   → Cả 2 đều safe (any write matching original predicate matches approximation)

Implement: Attach lock tới INDEX ENTRY
   Room_id index: attach shared lock tới entry room 123
   Time index: attach shared lock tới range noon to 1pm

   → Other transaction INSERT conflict booking:
     Phải update same part of index → gặp shared lock → WAIT

Nếu không có suitable index:
   → Fall back: shared lock on ENTIRE TABLE (safe nhưng not good for performance)
```

### 2PL ≠ 2PC (Quan trọng!)

```
2PL = Two-Phase Locking: concurrency control cho serializability
2PC = Two-Phase Commit: atomic commit trong distributed database

→ Hai khái niệm HOÀN TOÀN KHÁC NHAU, chỉ tương đồng về tên
```

---

## 4. Serializable Snapshot Isolation (SSI)

### Vấn đề với 2PL và Serial Execution

```
2PL: Serializable nhưng performance KÉM (lock contention)
Serial: Serializable nhưng limited throughput (single CPU core)

→ Are serializable isolation and good performance fundamentally at odds?
```

### SSI — Giải pháp Mới (2008)

```
SSI: Full serializability với SMALL PERFORMANCE PENALTY
   Dùng trong: PostgreSQL (serializable), CockroachDB, FoundationDB,
               SQL Server In-Memory OLTP, BadgerDB
```

### Pessimistic vs Optimistic

```
2PL = PESSIMISTIC: "Nếu có khả năng sai → WAIT cho đến khi safe"
   (Mutexes trong multithreaded programming)

SSI = OPTIMISTIC: "Proceed dù có potential danger,
   HY VỌNG everything will be fine"
   → Khi commit: CHECK nếu isolation violated
   → Nếu có conflict: ABORT + retry

Optimistic phù hợp khi: LOW CONTENTION (ít conflicts)
                          Enough spare capacity
```

### SSI dựa trên Snapshot Isolation

```
Tất cả reads trong transaction: consistent snapshot
SSI thêm: Algorithm detect SERIALIZATION CONFLICTS

Concept: "Decision based on outdated premise"
   Transaction đọc data → dùng kết quả để DECIDE action
   → Action dựa trên PREMISE về state lúc đọc
   → Nếu premise thay đổi trước khi commit → ABORT

2 loại serialization conflicts SSI detect:
   1. Stale MVCC reads (uncommitted write occurred before read)
   2. Writes affecting prior reads (write occurs after read)
```

### Detecting Stale MVCC Reads (Figure 8-10)

```
Txn 43: SELECT doctors WHERE on_call = true AND shift = 1234
   → Thấy: Alice on_call=true, Bob on_call=true
   → Txn 42 đang update Alice: on_call=false (uncommitted) → IGNORED bởi snapshot

   Sau đó: Txn 42 COMMITS

Txn 43 muốn commit:
   DB check: Bất kỳ ignored writes nào đã committed chưa?
   → Yes (txn 42 committed Alice's on_call=false)
   → PREMISE của txn 43 NO LONGER TRUE → ABORT txn 43

Tại sao chờ đến lúc commit? Tại sao không abort ngay khi detect stale read?
   → Txn 43 có thể là read-only → không cần abort (no write skew risk)
   → Txn 42 có thể vẫn chưa committed → read may not be stale
   → Avoiding unnecessary aborts → preserves snapshot isolation benefits
```

### Detecting Writes Affecting Prior Reads (Figure 8-11)

```
Txn 42 và 43 đồng thời tìm doctors on_call trong shift 1234

DB dùng index entry "shift_id = 1234" để RECORD:
   "Txn 42 và 43 đã đọc data liên quan đến shift_id = 1234"

Khi txn 43 WRITE (doctor off call):
   → DB check index: Có transaction nào khác đọc affected data?
   → Yes (txn 42)
   → Txn 42 được NOTIFIED: "Prior read may be outdated"

Kết quả:
   Txn 42 commits first → success
   → Txn 43 muốn commit: thấy conflicting write từ txn 42 đã commit
   → Txn 43 ABORTED
```

### Performance của SSI

```
So với 2PL:
   ✓ NO BLOCKING: transaction không cần chờ locks của transaction khác
   ✓ Readers không block writers, writers không block readers
   ✓ Read-only queries: run trên consistent snapshot không cần locks
   → Query latency MORE PREDICTABLE, LESS VARIABLE

So với Serial Execution:
   ✓ KHÔNG limited bởi 1 CPU core (FoundationDB distribute conflict detection)
   ✓ Transactions có thể read/write data trên MULTIPLE SHARDS + serializable

Trade-off:
   Cần check for serialization violations → overhead
   Debate: đáng không? Một số: "not worth it". Khác: "performance now so good"

   Rate of aborts ảnh hưởng performance:
   → Long read-write transactions: likely conflicts → abort
   → SSI requires FAIRLY SHORT read-write transactions (long read-only OK)
   → Less sensitive to slow txns than 2PL or serial execution
```

---

## Tóm tắt phần này

```
3 APPROACHES TO SERIALIZABILITY:

1. SERIAL EXECUTION:
   Chỉ 1 txn/lần, stored procedures, in-memory data
   ✓ Simple, fast nếu điều kiện phù hợp
   ✗ Single CPU core, cross-shard coordination overhead

2. TWO-PHASE LOCKING (2PL):
   Writers block readers AND vice versa
   Predicate locks / Index-range locks cho phantoms
   ✓ Widely used (MySQL/InnoDB serializable, SQL Server serializable)
   ✗ Performance poor (lock contention, deadlocks, unstable latency)

3. SERIALIZABLE SNAPSHOT ISOLATION (SSI):
   Optimistic: proceed → check at commit → abort if conflict
   Detect: stale MVCC reads + writes affecting prior reads
   ✓ Better performance (no blocking), scales qua shards
   ✓ PostgreSQL, CockroachDB, FoundationDB
   ✗ Abort rate ảnh hưởng performance, read-write txns must be short

RULE OF THUMB:
   Low contention → SSI thường best
   High contention → 2PL may be better (fewer aborts)
   Simple workload + in-memory + low throughput → Serial execution
```
