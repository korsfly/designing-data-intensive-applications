# ACID và Single/Multi-Object Transactions

---

## 1. ACID — Bốn Properties

> Transactions = abstraction layer cho phép application pretend một số concurrency problems và hardware/software faults không tồn tại.

### A — Atomicity (Tính nguyên tử)

```
Không phải "atomic" theo nghĩa của concurrency (Chapter 9)
→ Nghĩa là: CANNOT BE BROKEN DOWN INTO SMALLER PARTS

Nếu transaction thực hiện nhiều writes:
   ALL writes committed → SUCCESS
   NẾU có fault giữa chừng (network fail, crash, disk full,...):
   → ABORT: TẤT CẢ writes bị ROLL BACK (như chưa xảy ra)

Kết quả: Application có thể RETRY transaction với tự tin
         (không lo về state nửa vời)
```

### C — Consistency (Tính nhất quán)

```
C trong ACID là khái niệm DATABASE NHẤT LẪN LỘN và MÔNG LUNG nhất

Ý nghĩa thực: application-specific INVARIANTS về data phải luôn đúng
   Ví dụ: "Tổng tài sản của tất cả accounts luôn bằng nhau" (double-entry accounting)

NHƯNG: Database KHÔNG thể đảm bảo điều này!
   → Đây là responsibility của APPLICATION (phải write transactions đúng)
   → Database chỉ enforce: NOT NULL constraints, referential integrity,...

⚠️ C trong ACID thực ra là PROPERTY CỦA APPLICATION,
   không phải database. "The letter C doesn't really belong in ACID"

Historical note: "ACID" acronym tạo ra bởi Andreas Reuter và Theo Härder (1983)
để contrast với ACID's competitors. C được thêm vào để cho ra chữ đẹp.
```

### I — Isolation (Tính cô lập)

```
Concurrent transactions ISOLATED từ nhau:
   → Chúng PRETEND là chỉ transaction đó đang chạy trong DB
   → Khi commit: chúng thấy kết quả NHƯ THỂ chạy TUẦN TỰ (serially)
                 dù thực tế có thể đã chạy song song

Thực tế: Full isolation (serializability) ảnh hưởng performance
→ Hầu hết DBs dùng WEAKER ISOLATION LEVELS (xem Section 2 và 3)
```

### D — Durability (Tính bền vững)

```
Sau khi transaction committed:
   Dữ liệu sẽ KHÔNG BỊ MẤT ngay cả khi hardware failure hoặc DB crash

Single-node: Thường dùng WRITE-AHEAD LOG (WAL) + checkpoints
             → Writes flushed to non-volatile storage (fsync)

Multi-node: Durability = data được replicated thành công sang TẬP HỢP NODES ĐỦ LỚNC
           (thường majority quorum)

⚠️ "Perfect durability" KHÔNG TỒN TẠI:
   Nếu tất cả disks hỏng cùng lúc,
   nếu datacenter bị lũ lụt, động đất
   → Ngay cả WAL + replication cũng không đủ
```

### ACID vs BASE (NoSQL narrative)

```
BASE = Basically Available, Soft state, Eventually consistent
   (INTENTIONALLY VAGUE, "the opposite of ACID")
   → Nhiều NoSQL databases tự mô tả mình là "BASE"

THỰC TẾ: Ranh giới không rõ ràng:
   - Nhiều "NoSQL" databases hỗ trợ transactions tốt
   - Nhiều relational databases có weak isolation mặc định
```

---

## 2. Single-Object Writes — Atomicity và Isolation

### Vấn đề ngay cả với 1 object

```
Write đơn lẻ có thể bị interrupted giữa chừng:
   - Write 20KB JSON → crash sau khi 10KB → partially written record
   - Power failure → storage full write trước khi index updated
   - Node sees partially updated value while another writes

→ Ngay cả 1 object write cũng cần atomicity + isolation!
```

### Giải pháp phổ biến cho Single-Object

| Mechanism                                    | Mục đích                                     |
| -------------------------------------------- | -------------------------------------------- |
| **Write-ahead log (WAL)**                    | Atomicity (crash recovery)                   |
| **Concurrency control** (lock per object)    | Isolation                                    |
| **Atomic operations (compare-and-set, CAS)** | "Lightweight transactions" cho single object |
| **Increment operation**                      | Read-modify-write cycle atomic               |

```
Ví dụ CAS (Compare-and-Set):
   UPDATE counters SET value = value + 1 WHERE key = 'foo' AND value = 42;
   → Chỉ update nếu current value = 42 (kiểm tra race condition)
```

> **Quan trọng:** Những "lightweight transactions" này CHỈ áp dụng cho single object. "Transactions" theo nghĩa thực sự là **multi-object atomicity + isolation**.

---

## 3. Multi-Object Transactions

### Khi nào cần Multi-Object Transactions?

```
Ví dụ 1: Email Application
   Mailbox table: email_id, ...
   Unread_count table: mailbox_id, count

   Insert email (mới) → cũng phải INCREMENT unread_count
   → 2 objects phải update ATOMICALLY
   → Nếu insert thành công nhưng increment fail:
     Count không còn chính xác → BUG
```

```
Ví dụ 2: Relational Model — Referential Integrity
   INSERT record with foreign key constraint
   → Referenced row phải tồn tại
   → Nếu không có transaction:
     Có thể insert orphan records trong gap giữa 2 operations
```

```
Ví dụ 3: Denormalized Data
   Update 1 document/row → cũng phải update NHIỀU denormalized copies
   → Nếu partial update: inconsistent denormalized data
   (phổ biến trong document databases)
```

```
Ví dụ 4: Secondary Indexes
   Write record → cũng phải update secondary indexes
   → Nếu partial update: record appears in some views, not others
```

### Tại sao nhiều DB "không có transactions" vẫn hoạt động?

```
Workloads đơn giản (chỉ read/write SINGLE RECORD):
   → Có thể manage WITHOUT transactions

Nhưng: Khi access patterns phức tạp hơn:
   → Transactions hugely reduce error cases phải think about

Không có transactions: Processes crashing, network interruptions,
power outages, disk full, unexpected concurrency, v.v.
→ Data có thể inconsistent theo NHIỀU CÁCH KHÁC NHAU
→ Rất khó reason về system behavior
```

---

## 4. Handling Errors và Aborts

### Transaction Abort → Retry

```
Transaction có thể bị aborted (bởi DB, do violations, errors,...)
→ Application thường RETRY aborted transaction

NHƯNG: Retry không đơn giản:
```

| Vấn đề khi Retry         | Chi tiết                                                                     |
| ------------------------ | ---------------------------------------------------------------------------- |
| **Network failure**      | Transaction có thể đã committed, nhưng network lost ack → Retry = DUPLICATE! |
| **Overload**             | Error do overload → Retry làm xấu thêm; cần exponential backoff              |
| **Non-transient errors** | Constraint violation → Retry sẽ fail lại → Application phải handle           |
| **Side effects**         | Email được gửi bên trong transaction → Retry gửi 2 lần                       |

### Savepoints

```
Cho phép PARTIAL ROLLBACK (chỉ một phần transaction):

Transaction:
   INSERT ... OK
   UPDATE ... OK
   SAVEPOINT my_savepoint;   ← đánh dấu điểm này
   INSERT ... FAIL
   ROLLBACK TO SAVEPOINT my_savepoint;  ← chỉ undo từ savepoint trở đi
   INSERT (lần khác)...
   COMMIT;  ← commit toàn bộ

Không phải tất cả DBs support savepoints.
```

---

## Tóm tắt phần này

```
ACID:
   A: All-or-nothing (atomicity khi write nhiều objects/crash recovery)
   C: Application invariants (application's responsibility, không phải DB)
   I: Concurrent transactions isolated from each other
   D: Committed data not lost even on hardware failure

SINGLE-OBJECT: WAL + lock per object + atomic ops (CAS, increment)
MULTI-OBJECT: ACID transactions handle nhiều objects atomically
   Cần cho: relational integrity, denormalized data, secondary indexes

HANDLING ERRORS:
   Retry aborted transaction: hầu hết ORMS làm automatically
   NHƯNG: phải care về: network fail (duplicate), overload, side effects
   Savepoints: partial rollback trong transaction
```
