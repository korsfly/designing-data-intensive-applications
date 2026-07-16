# Detecting Concurrent Writes và Version Vectors

---

## 1. Vấn đề cốt lõi: Event Ordering

```
Leaderless databases (cũng: multi-leader) cho phép concurrent writes đến cùng key
→ Conflicts cần được resolved

Events có thể ARRIVE theo DIFFERENT ORDER tại different nodes:
   (vì variable network delays và partial failures)

Ví dụ (Figure 6-14): 2 clients A và B, đồng thời write vào key X, 3 nodes:
   Node 1: nhận write A, KHÔNG nhận write B (transient outage)
   Node 2: nhận A trước, rồi B
   Node 3: nhận B trước, rồi A

→ Nếu mỗi node simply OVERWRITE value khi nhận request:
   Node 1: X = A
   Node 2: X = B (overwrite A)
   Node 3: X = A (overwrite B)
   → PERMANENTLY INCONSISTENT! Nodes không đồng ý về final value

→ Cần mechanism để DETECT và RESOLVE concurrent writes
```

---

## 2. Happens-Before Relation và Concurrency

### Định nghĩa chính xác

> **Operation A happens before Operation B** nếu B KNOWS ABOUT A, DEPENDS ON A, hoặc BUILDS UPON A theo bất kỳ cách nào.

> **Concurrent:** Hai operations A và B là concurrent nếu NEITHER happens before the other.

```
3 khả năng giữa 2 operations A và B:
   1. A happened before B (B knows about / depends on A)
   2. B happened before A (A knows about / depends on B)
   3. A và B concurrent (neither knows about the other)

Algorithm phân biệt 3 trường hợp này = KEY to conflict detection
```

### Concurrency ≠ Same Physical Time

```
KHÔNG QUAN TRỌNG liệu chúng literally xảy ra cùng lúc về vật lý

Hai operations là CONCURRENT nếu:
   Cả 2 UNAWARE OF EACH OTHER
   (bất kể physical time chúng xảy ra)

Ví dụ: Network slow/interrupted → 2 operations xảy ra CÁCH NHAU
   nhưng network problems prevented chúng BIẾT VỀ NHAU
   → VẪN CONCURRENT theo định nghĩa

Connection với Special Theory of Relativity:
   Information cannot travel faster than light
   2 events cách nhau đủ xa về không gian/thời gian KHÔNG THỂ ảnh hưởng nhau
   → Trong CS: network delay plays same role as speed of light
```

---

## 3. Algorithm: Capturing Happens-Before (Single Replica)

### Cơ chế (Figure 6-15 — Shopping Cart Example)

```
DATABASE:
   Maintain VERSION NUMBER cho mỗi key
   Increment version number mỗi khi key được written
   Store new version number cùng với value written

CLIENT READ:
   Server trả về TẤT CẢ SIBLINGS (values chưa bị overwrite)
   VÀ latest version number
   Client PHẢI READ trước khi WRITE

CLIENT WRITE:
   Phải include VERSION NUMBER từ prior read
   Phải MERGE TẤT CẢ siblings nhận được trong prior read
   (ví dụ: dùng CRDT)

SERVER nhận write với particular version number:
   OVERWRITE TẤT CẢ values với version ≤ version number đó
   (đã được merged vào new value)
   KEEP TẤT CẢ values với version number CAO HƠN
   (concurrent với incoming write → chưa được merged)
```

### Ví dụ Step-by-Step (Figure 6-15): Shopping Cart

```
Initial state: Cart empty, version 0

Step 1: Client 1 add "milk"
   → Server: store [milk] với version 1
   → Server trả về: [milk], version 1 cho client 1

Step 2: Client 2 add "eggs" (KHÔNG biết về milk)
   → Server: store [milk] và [eggs] như 2 siblings (concurrent)
     assigns version 2
   → Server trả về: [milk, eggs], version 2 cho client 2

Step 3: Client 1 add "flour" (vẫn nghĩ cart = [milk])
   → Client 1 gửi [milk, flour] với version 1
   → Server: version 1 supersedes [milk] (merge) → overwrite
              NHƯNG [eggs] = version 2 = concurrent với write này → KEEP
   → Assign version 3 cho [milk, flour]
   → Trả về: [[milk, flour], [eggs]], version 3

Step 4: Client 2 muốn add "ham" (có [milk] và [eggs] từ step 2)
   → Client 2 merge: [eggs, milk, ham] với version 2
   → Server: version 2 supersedes [eggs] → overwrite
              [milk, flour] = version 3 = concurrent → KEEP
   → Assign version 4 cho [eggs, milk, ham]
   → Trả về: [[milk, flour], [eggs, milk, ham]], version 4

Step 5: Client 1 add "bacon" (có [milk, flour] và [eggs] từ step 3)
   → Client 1 merge: [milk, flour, eggs, bacon] với version 3
   → Server: version 3 supersedes [milk, flour] → overwrite
              ([eggs] đã được overwrite ở step 4)
              [eggs, milk, ham] = version 4 = concurrent → KEEP
   → Final: [[milk, flour, eggs, bacon], [eggs, milk, ham]]
```

```
KẾT QUẢ: Clients không bao giờ fully up-to-date với server
   (luôn có operation đang diễn ra concurrent)
   NHƯNG: Old versions eventually overwritten
          Không có writes bị LOST
```

### Write mà KHÔNG include Version Number

```
→ Concurrent với TẤT CẢ other writes
→ KHÔNG overwrite gì cả
→ Sẽ được trả về như 1 trong các siblings trong subsequent reads
```

---

## 4. Version Vectors (Multiple Replicas)

### Vấn đề với Single Version Number khi có Multiple Replicas

```
Single version number ĐỦ với 1 replica
NHƯNG với NHIỀU REPLICAS accepting writes CONCURRENTLY:
   1 version number per key KHÔNG ĐỦ để capture dependencies

→ Cần VERSION NUMBER PER REPLICA cũng như PER KEY
```

### Version Vector

> **Version Vector:** Collection của version numbers từ TẤT CẢ replicas.

```
Mỗi replica INCREMENT OWN VERSION NUMBER khi process write
Mỗi replica cũng TRACK VERSION NUMBERS từ CÁC REPLICAS KHÁC

→ Xác định: values nào cần OVERWRITE, values nào là SIBLINGS (cần keep)

Database → Client: gửi version vector khi read values
Client → Database: gửi version vector back khi write values

Version vector cho phép:
   An toàn khi read từ 1 replica, write lại sang replica KHÁC
   Có thể tạo ra siblings, NHƯNG data không bị mất
   miễn là siblings được merged ĐÚNG CÁCH

Riak: encode version vector dưới dạng string gọi là "CAUSAL CONTEXT"
Riak 2.0: dùng DOTTED VERSION VECTOR (variant tinh tế hơn)
```

### Version Vector vs Vector Clock

```
Thường bị dùng thay thế nhau, nhưng:
   Vector Clock: dùng để order events (timestamp events)
   Version Vector: dùng để compare state of replicas
   → KHÁC NHAU mặc dù tương tự

Khi so sánh state of replicas → VERSION VECTOR là right data structure
```

---

## Tóm tắt phần này

```
HAPPENS-BEFORE:
   A happens before B: B knows about / builds upon A
   Concurrent: neither knows about the other
   3 cases: A→B, B→A, A||B (concurrent)
   Physical time KHÔNG quan trọng, chỉ causal knowledge quan trọng

ALGORITHM (Single Replica):
   Server maintains version number per key
   Client phải READ trước khi WRITE
   Write includes version number → server biết cái gì concurrent
   Server keeps higher-version siblings, overwrites same/lower
   Clients merge siblings (dùng CRDT)

VERSION VECTORS (Multiple Replicas):
   1 version number per (replica, key) combination
   Collection = version vector (causal context trong Riak)
   Sent DB→client on read; sent back client→DB on write
   Allows read from replica A, write back to replica B safely

RELATED:
   CRDT/OT: automatic merge algorithms cho siblings
   LWW: simple but data loss; version vectors: no data loss
   Version vector ≠ vector clock (different purposes)
```
