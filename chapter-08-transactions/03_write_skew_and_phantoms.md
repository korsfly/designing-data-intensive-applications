# Write Skew và Phantoms

---

## 1. Write Skew — Subtle Race Condition

### Ví dụ kinh điển: Doctor On-Call (Figure 8-5)

```
Business rule: Bệnh viện yêu cầu ÍT NHẤT 1 doctor on call tại mọi thời điểm

Database: doctors table với on_call column (boolean)
   Alice: on_call = true
   Bob: on_call = true

Transaction của Alice (đang shift Alice):
   SELECT COUNT(*) FROM doctors WHERE on_call = true;  → 2 (OK)
   UPDATE doctors SET on_call = false WHERE name = 'Alice';

Transaction của Bob (đồng thời, cùng shift):
   SELECT COUNT(*) FROM doctors WHERE on_call = true;  → 2 (OK)
   UPDATE doctors SET on_call = false WHERE name = 'Bob';

Kết quả: CẢ HAI Alice VÀ Bob không on call → VI PHẠM business rule!
```

### Tại sao Snapshot Isolation KHÔNG ngăn Write Skew?

```
Mỗi transaction đọc VALID SNAPSHOT (>1 doctor on call) → quyết định là đúng
NHƯNG: Cả 2 đồng thời quyết định "an toàn để rời khỏi"
       → kết quả combined là WRONG

→ Write skew KHÔNG phải dirty write (write different objects)
→ KHÔNG phải lost update (write DIFFERENT objects: Alice writes Alice, Bob writes Bob)
```

### Các ví dụ khác về Write Skew

| Use case                 | Write skew scenario                                                      |
| ------------------------ | ------------------------------------------------------------------------ |
| **Meeting room booking** | 2 users book cùng phòng cùng time slot simultaneously                    |
| **Multiplayer game**     | 2 players move to same position simultaneously                           |
| **Claiming username**    | 2 users claim same username simultaneously                               |
| **Double-spending**      | Spending points: check balance → spend (2 concurrent → negative balance) |

```
PATTERN chung của write skew:
   1. SELECT query → check condition (precondition)
   2. Dựa vào result → application code DECIDES to write
   3. WRITE thực hiện (thay đổi premise của condition)

   Với 2 concurrent transactions:
   Step 1 cả 2 thấy CÙNG STATE (OK)
   Step 3 cả 2 WRITE → combined result phá vỡ invariant
```

---

## 2. Phantoms — Write Skew trên Search Conditions

### Định nghĩa

```
PHANTOM:
   Transaction A thực hiện search query
   Transaction B write → ẢNH HƯỞNG đến kết quả của search query đó

   → Transaction A's view "appears" correct với snapshot
   → NHƯNG combined state is wrong

Write skew có thể xảy ra khi:
   Write AFFECTS OBJECTS mà READ QUERY CỦA TRANSACTION KHÁC đang xem
   → Dù những objects đó là KHÁC NHAU (hoặc KHÔNG TỒN TẠI khi đọc)
```

### Meeting Room Booking (Example 8-2)

```sql
BEGIN;
-- Check conflicts (concurrent bookings for same room + time)
SELECT COUNT(*) FROM bookings
WHERE room_id = 123
  AND end_time > '12:00' AND start_time < '13:00';
-- Application: if 0, safe to book
INSERT INTO bookings(room_id, start_time, end_time, user_id)
VALUES (123, '12:00', '13:00', 456);
COMMIT;
```

```
Vấn đề: 2 transactions đồng thời cùng check → cả 2 thấy COUNT = 0
         → Cả 2 INSERT → DOUBLE BOOKING!

Select và Insert về OBJECTS KHÁC NHAU (phantom):
   SELECT trả về 0 rows (không tìm thấy conflict)
   INSERT tạo row mới (không phải update row cũ)
   → KHÔNG PHẢI write-to-same-object conflict!

→ Materializing conflict (tạo row trước để lock) là workaround
```

### Materializing Conflicts — Workaround

```
Idea: Thêm rows vào DB đại diện cho "potential conflicts"
      để có thể LOCK chúng

Ví dụ meeting room:
   Tạo bảng với 1 row cho mỗi (room, time_slot) combination
   → SELECT ... FOR UPDATE trên row đó TRƯỚC khi check

   Nhược điểm:
   - Phải biết trước tất cả potential conflicts
   - Phức tạp, error-prone, không elegant

→ Last resort: nên dùng SERIALIZABLE ISOLATION thay thế
```

---

## 3. SELECT FOR UPDATE — Explicit Locking giải quyết Write Skew

### Áp dụng cho Doctor On-Call

```sql
BEGIN;
SELECT * FROM doctors
WHERE on_call = true AND shift_id = 1234
FOR UPDATE;  -- LOCK TẤT CẢ returned rows

-- Application kiểm tra: nếu chỉ còn 1 người on call → abort
UPDATE doctors SET on_call = false WHERE name = 'Alice' AND shift_id = 1234;
COMMIT;
```

```
FOR UPDATE lock mọi doctor returned by query
→ Concurrent transaction muốn đọc same doctors PHẢI CHỜ
→ Giải quyết write skew

NHƯNG: Phụ thuộc vào developer nhớ thêm FOR UPDATE
        → Dễ quên → Lỗi âm thầm
```

### Khi FOR UPDATE không đủ — Phantoms

```
Meeting room booking:
   SELECT ... FOR UPDATE không lock rows KHÔNG TỒN TẠI
   (không có existing booking để lock)

   → Không có object nào để attach lock
   → Cần PREDICATE LOCK hoặc INDEX-RANGE LOCK
   → Hoặc dùng SERIALIZABLE ISOLATION (tốt nhất)
```

---

## 4. Tổng hợp Anomalies và Giải pháp

| Anomaly         | Định nghĩa                                             | Giải pháp                           |
| --------------- | ------------------------------------------------------ | ----------------------------------- |
| **Dirty read**  | Đọc uncommitted data                                   | Read committed                      |
| **Dirty write** | Overwrite uncommitted data                             | Read committed                      |
| **Read skew**   | Đọc different parts tại different times                | Snapshot isolation                  |
| **Lost update** | Read-modify-write cycle bị overwrite                   | Atomic ops, FOR UPDATE, auto-detect |
| **Write skew**  | Đọc shared premise → write differently, combined wrong | Serializable isolation              |
| **Phantom**     | Write affects search condition của transaction khác    | Serializable isolation              |

```
Chỉ có SERIALIZABLE ISOLATION ngăn được TẤT CẢ anomalies
```

---

## Tóm tắt phần này

```
WRITE SKEW:
   Pattern: Check condition → decide to write → combined result wrong
   KHÔNG phải dirty write (different objects)
   KHÔNG phải lost update (write to different objects)
   Ví dụ: doctor on-call, meeting room, username claim, double-spending

PHANTOMS:
   Write affects search condition của transaction khác
   Đặc biệt khó: write tới rows KHÔNG TỒN TẠI khi đọc
   → Không có object để lock!

GIẢI PHÁP:
   SELECT FOR UPDATE: giải quyết write skew nếu objects exist
   Materializing conflicts: workaround cho phantoms (phức tạp)
   SERIALIZABLE ISOLATION: giải pháp đúng đắn nhất

Thực tế: Hầu hết developers KHÔNG biết đến write skew/phantoms
   → Dễ introduce bugs trong applications với complex concurrent access
```
