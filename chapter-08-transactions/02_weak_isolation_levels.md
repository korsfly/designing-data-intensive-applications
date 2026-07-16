# Weak Isolation Levels

---

## 1. Tại sao Weak Isolation?

```
Full isolation (Serializability):
   → Đảm bảo hoàn toàn nhưng PERFORMANCE COST CAO

→ Hầu hết databases dùng WEAKER ISOLATION LEVELS
   = Strong hơn "no isolation" nhưng kém hơn serializable

Hệ quả: Nhiều subtle bugs trong production do weak isolation
   Flexcoin (2014): Race condition → $620,000 Bitcoin stolen
   Poloniex (2014): Race condition → 12.3% more Bitcoin withdrawn than allowed
   Post Office Horizon scandal: Potentially related concurrency bugs
```

---

## 2. Read Committed (Isolation Level phổ biến nhất)

### 2 Guarantees cơ bản

```
1. NO DIRTY READS:
   Chỉ thấy data đã được COMMITTED
   (không thấy data của uncommitted transactions)

2. NO DIRTY WRITES:
   Chỉ overwrite data đã được COMMITTED
   (không overwrite data của uncommitted transactions)
```

### Dirty Reads — Vấn đề

```
User 1: UPDATE accounts SET balance = 600 WHERE id = 1;
         (chưa commit, balance hiện là 600 in-progress)
User 2: SELECT balance FROM accounts WHERE id = 1;
         → Nếu thấy 600 = DIRTY READ (uncommitted data)

Nếu User 1 sau đó ROLLBACK:
   User 2 đã đọc "phantom" data không bao giờ commit!
   → Rất confusing
```

### Dirty Writes — Vấn đề (Figure 8-3)

```
Hai users cùng mua 1 item (last-write-wins no transaction):

User 1: UPDATE items SET buyer = 'user1' WHERE id = X;   (ok)
User 2: UPDATE items SET buyer = 'user2' WHERE id = X;   (overwrite user1's write)
User 1: UPDATE invoices SET customer = 'user1' WHERE item = X;  (user1 pays)
User 2: UPDATE invoices SET customer = 'user2' WHERE item = X;  (user2 pays)

Kết quả WEIRD: item.buyer = user2, invoice.customer = user2
   NHƯNG nếu order different: item.buyer = user2, invoice.customer = user1
   → Bob mua xe nhưng thực ra Alice trả tiền!
```

### Implement No Dirty Reads

```
Multi-version approach (most common):
   Transaction đang update → DB giữ CẢ HAI versions:
   - Old committed value (available for reads)
   - New uncommitted value (chỉ thấy bởi transaction đang write)

   → Other transactions thấy OLD VALUE cho đến khi new value committed
   → KHÔNG cần lock reads → Better performance
```

### Implement No Dirty Writes

```
Lock per object (row-level lock):
   Transaction muốn write object → phải acquire lock TRƯỚC
   → Phải CHỜ đến khi transaction cũ commit hoặc abort
   → Sau đó mới được write và giữ lock cho đến khi commit
```

---

## 3. Read Committed vs Snapshot Isolation

### Read Skew (Nonrepeatable Read) — Vấn đề không giải quyết bởi Read Committed

```
Alice có 2 accounts, mỗi tài khoản $500, tổng $1000

Alice reads account 1: $500 (committed)
--- Bank đang xử lý transfer $100 từ account 1 sang 2 ---
Alice reads account 2: $600 (committed sau transfer)

Alice thấy TỔNG = $1100 (không đúng!)
   → Bà thấy một phần state TRƯỚC transfer, một phần SAU transfer
   → Gọi là READ SKEW (nonrepeatable read)

Read Committed KHÔNG ngăn điều này vì:
   Cả 2 reads đều thấy COMMITTED data
   Chỉ là TIMESTAMPS của 2 reads KHÁC NHAU
```

### Khi nào Read Skew không OK?

```
Transient anomaly → OK với read committed cho một số use cases

Không OK:
1. BACKUPS: Entire DB được dump, quá trình mất vài giờ
   → Backup chứa data từ nhiều thời điểm KHÁC NHAU → bất nhất

2. ANALYTICAL QUERIES:
   Query scan LARGE PORTION of database
   → Nếu data đang thay đổi trong lúc scan → nonsensical results

3. INTEGRITY CHECKS:
   Check xem DB có inconsistencies không
   → Read skew có thể làm appear có inconsistency khi không có
```

---

## 4. Snapshot Isolation

### Nguyên tắc cơ bản

```
MỖI TRANSACTION đọc từ CONSISTENT SNAPSHOT của database:
   Chỉ thấy data đã committed VÀO LÚC TRANSACTION BẮT ĐẦU
   → Kể cả có writes committed SAU ĐÓ, transaction KHÔNG thấy

→ LONG-RUNNING READ-ONLY TRANSACTIONS: thấy consistent state
   dù DB đang có concurrent updates

→ Thích hợp cho: backups, analytical queries, integrity checks
```

### Ai dùng Snapshot Isolation?

```
PostgreSQL, Oracle, SQL Server, MySQL InnoDB (dưới tên REPEATABLE READ)
CockroachDB, TiDB, FoundationDB, Spanner
```

### Implement: Multi-Version Concurrency Control (MVCC)

```
MVCC: Mỗi write KHÔNG overwrite old value
      Thay vào đó: TẠO VERSION MỚI

Mỗi row trong DB có TẬP HỢP versions:
   row_id | created_by_txn | deleted_by_txn | value
   -------+----------------+----------------+------
   1      | txn 1          | null           | 500   (current)
   1      | txn 1          | txn 5          | 500   (old, deleted by txn 5)
   1      | txn 5          | null           | 400   (new, written by txn 5)

MVCC KHÔNG thực sự xóa: chỉ mark "deleted_by" với txn ID
   → GC process định kỳ DELETE versions không cần nữa
   (PostgreSQL VACUUM process)
```

### Visibility Rules

```
Mỗi transaction có UNIQUE MONOTONICALLY INCREASING transaction ID

Khi transaction bắt đầu: snapshot được tạo
   → Transaction chỉ thấy: (1) Data committed TRƯỚC khi snapshot tạo
                           (2) Writes của CHÍNH TRANSACTION ĐÓ

Transaction KHÔNG thấy: (1) Uncommitted transactions khi snapshot tạo
                         (2) Writes của transactions committed SAU snapshot
                         (3) Writes của aborted transactions

→ Để determine visibility: compare transaction IDs
```

### Snapshot Isolation và Indexes

```
Vấn đề: Index trỏ đến object với NHIỀU versions
         → Nếu show all versions, index scan lấy nhiều hơn cần

Options:
   1. Separate index entry per version + filter theo visibility rules
      (append-only, copy-on-write B-trees)
   2. Nếu index entry trỏ đến nhiều versions cùng page:
      → Filter versions later (like PostgreSQL)
   3. Dùng LSM-trees cho versioned data (RocksDB, LevelDB)

PostgreSQL MVCC disadvantage: Bloat từ nhiều versions của cùng row
   (Vacuuming được biết là "the part of PostgreSQL we hate most")
```

---

## 5. Lost Updates (Concurrent Read-Modify-Write Cycles)

### Vấn đề

```
2 transactions đồng thời read → modify → write cùng object:
   Transaction 1: READ counter (42) → +1 → WRITE 43
   Transaction 2: READ counter (42) → +1 → WRITE 43  ← mất increment của T1!

Kết quả: Counter = 43, dù được increment 2 lần → LOST UPDATE
```

### Ví dụ thực tế

```
- Counter increment (like/view counts)
- Account balance update (withdraw money)
- Edit wiki page (read → modify text → write back)
- Multiplayer games (2 players move at same position simultaneously)
```

### Giải pháp 1: Atomic Write Operations

```
UPDATE counters SET value = value + 1 WHERE key = 'foo';

Database thực hiện nội bộ:
   Read → +1 → Write ATOMIC (với exclusive lock trên đối tượng)
→ NO read-modify-write cycle exposed to application

Hỗ trợ bởi: Hầu hết relational DBs
MongoDB: Atomic operations cho complex types (e.g., push to list)
Redis: Atomic operations cho complex data structures
```

### Giải pháp 2: Explicit Locking (SELECT FOR UPDATE)

```sql
BEGIN;
SELECT * FROM figures WHERE name = 'robot' AND game_id = 222 FOR UPDATE;
-- Application computes new position
UPDATE figures SET position = 'c4' WHERE id = 1234;
COMMIT;
```

```
FOR UPDATE: lock tất cả rows returned bởi query
   → Other transactions phải WAIT đến khi lock released

Dễ dùng nhưng: Dễ FORGET → application bugs
```

### Giải pháp 3: Automatically Detect Lost Updates

```
Database tự detect lost updates → ABORT TRANSACTION khi phát hiện
   → Application retry

PostgreSQL repeatable read (snapshot isolation):
   AUTOMATICALLY DETECT lost updates

MySQL/InnoDB, Oracle, SQL Server snapshot isolation:
   KHÔNG automatically detect → cần locks thủ công

→ Snapshot isolation đôi khi KHÔNG đảm bảo detect lost updates!
   (tùy implementation)
```

### Giải pháp 4: Compare-and-Set (CAS)

```sql
-- Chỉ update nếu value chưa thay đổi kể từ khi đọc
UPDATE wiki_pages SET content = 'new content'
WHERE id = 1234 AND content = 'old content';

-- Kiểm tra rows affected = 0 → retry
```

```
⚠️ Không an toàn nếu DB snapshot isolation:
   WHERE clause đọc từ OLD SNAPSHOT → có thể TRUE nhưng đã changed
   → Phải kiểm tra DB cụ thể
```

### Giải pháp 5: Conflict Detection trong Replicated DBs

```
Multi-leader / leaderless: Concurrent writes → CONFLICTS
→ Dùng LWW (Last Write Wins) = lost update là mặc định!
→ Cần: application-level conflict resolution (CRDT) hoặc
        tracking happened-before relationships (Chapter 6)

→ Cần cẩn thận khi dùng replicated databases!
```

---

## Tóm tắt phần này

```
READ COMMITTED (hầu hết DBs dùng mặc định):
   ✓ No dirty reads (uncommitted data không visible)
   ✓ No dirty writes (chỉ overwrite committed data)
   ✗ KHÔNG ngăn read skew (nonrepeatable reads)

SNAPSHOT ISOLATION / MVCC:
   ✓ Consistent snapshot khi bắt đầu transaction
   ✓ Ngăn read skew: tốt cho backups, analytics, integrity checks
   ✓ Writers don't block readers, readers don't block writers
   ✗ KHÔNG đảm bảo ngăn lost updates (tùy DB)
   Implementation: MVCC (multiple versions per row, visibility rules)

LOST UPDATES (concurrent read-modify-write):
   Giải pháp: Atomic ops | Explicit lock (FOR UPDATE) |
              Auto-detect | CAS | Application-level conflict resolution

   DBs tự động detect: PostgreSQL ✓; MySQL, Oracle: ✗ (cần locks)
```
