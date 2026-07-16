# Multi-Leader Replication

---

## 1. Giới thiệu và Động lực

### Hạn chế của Single-Leader

```
Single-leader: TẤT CẢ writes phải đi qua 1 leader
→ Nếu không thể kết nối leader (network interruption):
   KHÔNG THỂ write vào database
```

### Multi-Leader (Active/Active / Bidirectional Replication)

```
Cho phép NHIỀU HƠN 1 NODE accept writes
Mỗi leader đồng thời là follower của các leaders khác
Replication vẫn diễn ra tương tự: mỗi node process write
   → forward data change đến TẤT CẢ nodes khác

NOTE: Synchronous multi-leader ≈ single-leader (blocking khi network interrupt)
      → Chương này tập trung vào ASYNCHRONOUS multi-leader
```

### Khi nào dùng Multi-Leader? Hiếm trong single datacenter

```
Multi-leader TRONG 1 REGION: benefits hiếm khi outweigh added complexity

Các trường hợp HỢP LÝ:
   1. GEO-DISTRIBUTED operation (nhiều regions)
   2. OFFLINE operation (mobile apps, local-first)
   3. REAL-TIME COLLABORATION (Google Docs, Figma)
```

---

## 2. Geographically Distributed Operation

### Kiến trúc (Figure 6-6)

```
Mỗi region: 1 LEADER + followers trong region đó
Giữa regions: mỗi leader REPLICATES changes đến leaders ở regions khác

Within region: standard leader-follower replication
Between regions: multi-leader async replication
```

### So sánh Single-Leader vs Multi-Leader cho Multi-Region

| Khía cạnh                         | Single-Leader                                                                                 | Multi-Leader                                                                          |
| --------------------------------- | --------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------- |
| **Performance**                   | Mọi write → qua internet đến region có leader → LATENCY CAO                                   | Write xử lý tại LOCAL region → async replicate → inter-region delay ẩn với user       |
| **Tolerance of regional outages** | Failover promote follower ở region khác lên làm leader                                        | Mỗi region tiếp tục operate INDEPENDENTLY; catch up khi online lại                    |
| **Tolerance of network problems** | Rất sensitive: client ở region A write đến leader ở region B → phải chờ qua inter-region link | Async replication: tạm thời network interrupt → mỗi region vẫn process writes độc lập |
| **Consistency**                   | Có thể cung cấp STRONG consistency (serializable transactions)                                | Consistency YẾU HƠN NHIỀU: không thể guarantee bank balance không âm, username unique |

> **Lưu ý:** Multi-leader là FUNDAMENTAL LIMITATION của distributed systems — nếu cần enforce hard constraints, dùng single-leader.

### Databases hỗ trợ Multi-Leader

```
MySQL, Oracle, SQL Server, YugabyteDB (built-in)
Redis Enterprise, EDB Postgres Distributed, pglogical (external add-on)

CẢNH BÁO: Multi-leader trong nhiều DB là retrofitted feature
   → Subtle configuration pitfalls, surprising interactions
   → Autoincrement keys, triggers, integrity constraints → problematic
   → "Often considered dangerous territory that should be avoided if possible"
```

---

## 3. Multi-Leader Replication Topologies

### 3 Topologies (Figure 6-7)

#### Circular Topology

```
Node A → Node B → Node C → Node A (vòng tròn)
Mỗi node nhận writes từ 1 node, forward đến 1 node khác

VẤN ĐỀ: Nếu 1 node FAIL → INTERRUPT flow của toàn bộ vòng
         Các nodes khác không thể communicate cho đến khi node đó fixed
         Reconfiguration thường phải MANUAL
```

#### Star Topology

```
1 designated ROOT NODE forward writes đến TẤT CẢ nodes khác
(có thể generalize thành tree)

VẤN ĐỀ: Tương tự circular — root failure → toàn bộ bị interrupt
Có thể generalize thành TREE
```

#### All-to-All Topology (Phổ biến nhất)

```
Mỗi leader gửi writes đến MỌI leader khác

Fault tolerance TỐT HƠN:
   Messages có thể travel theo DIFFERENT PATHS → không có SPOF

VẤN ĐỀ RIÊNG: Network links tốc độ khác nhau
   → Replication messages có thể "OVERTAKE" nhau

Ví dụ (Figure 6-8):
   Client A INSERT row vào leader 1
   Client B UPDATE cùng row đó vào leader 3
   Leader 2 nhận UPDATE TRƯỚC INSERT → update row KHÔNG TỒN TẠI → LỖI

   Giải pháp: VERSION VECTORS (xem phần 5)
   NHƯNG: nhiều multi-leader systems không dùng good techniques
   → Test database kỹ trước khi dùng
```

### Ngăn chặn Infinite Replication Loops

```
Circular/Star topology: write có thể đi qua NHIỀU NODES

Mỗi node có UNIQUE IDENTIFIER
Trong replication log: mỗi write tagged với IDs của TẤT CẢ nodes đã đi qua
Node nhận write có tag = ID của chính nó → IGNORE (đã process rồi)
```

---

## 4. Sync Engines và Local-First Software

### Offline Operation = Multi-Leader

```
Calendar app trên nhiều devices:
   Đọc/ghi bất kỳ lúc nào, dù OFFLINE
   Changes synced với server và other devices khi có internet

→ Mỗi device = local DB replica acting as LEADER
→ Async multi-leader replication (sync) giữa các devices
   Replication lag: có thể HÀI hoặc NGÀY (tùy khi nào có internet)

→ Architecture này = multi-leader giữa regions, pushed to EXTREME
   (mỗi device = 1 "region", network = extremely unreliable)
```

### Real-Time Collaboration = Multi-Leader

```
Google Docs, Sheets, Figma, Linear,...
   User input → reflected in UI IMMEDIATELY (không chờ server)
   Edits từ 1 user → shown to collaborators với LOW LATENCY

→ Mỗi browser tab đang mở shared file = 1 REPLICA
→ Updates được async replicated đến devices của users khác

KHÔNG cần allow offline editing để multi-leader:
   Nhiều users edit cùng lúc mà không chờ server = đủ để multi-leader
```

### Sync Engine

> **Sync engine:** Software library hỗ trợ process capture changes, send to collaborators (nếu online) hoặc store locally (nếu offline), receive changes từ collaborators, merge vào local copy, update UI.

**Offline-first:** App cho phép user tiếp tục edit file khi offline (dùng sync engine)

**Local-first software:** Collaborative apps offline-first VÀ designed để tiếp tục hoạt động ngay cả khi developer shut down all online services. Dùng open standard sync protocol với nhiều service providers có thể.

```
Ví dụ: Git là local-first collaboration system
   (sync qua GitHub, GitLab, hoặc bất kỳ repo hosting nào)
```

### Ưu điểm của Sync Engines

| Ưu điểm                            | Giải thích                                                                              |
| ---------------------------------- | --------------------------------------------------------------------------------------- |
| **UI nhanh hơn**                   | Data locally → respond trong next frame (16ms ở 60Hz), không cần chờ network round-trip |
| **Offline support**                | "Being offline = having a very large network delay" — không cần separate offline mode   |
| **Programming model đơn giản hơn** | Reads/writes trên local data; operations almost never fail → more declarative style     |
| **Real-time edits từ others**      | Sync engine + reactive programming = efficient UI updates                               |

### Nhược điểm / Giới hạn

```
Works BEST khi TẤT CẢ data user cần được downloaded TRƯỚC
   → KHÔNG phù hợp nếu user có access đến VÔ SỐ data
   (OK: all files user created; NOT OK: entire ecommerce catalog)
```

### Sync Engine Implementations

| Loại                                  | Ví dụ                                    |
| ------------------------------------- | ---------------------------------------- |
| **Proprietary backend**               | Google Firestore, Realm, Ditto           |
| **Open source backend** (local-first) | PouchDB/CouchDB, Automerge, Yjs          |
| **Trong games**                       | "Netcode" — tương tự nhưng game-specific |

---

## 5. Dealing with Conflicting Writes

### Tại sao Conflict xảy ra?

```
Figure 6-9 ví dụ:
   User 1 thay đổi title wiki page: A → B (trên leader 1)
   User 2 thay đổi title wiki page: A → C (trên leader 2, đồng thời)
   Cả 2 changes: SUCCESSFULLY applied tại local leader

   Khi replicated async: CONFLICT DETECTED
   → Problem KHÔNG xảy ra trong single-leader database

Định nghĩa: 2 writes là CONCURRENT nếu neither was "aware" of the other
   → KHÔNG quan trọng chúng literally happen cùng lúc hay không
   → Quan trọng: 1 write có xảy ra trong state mà write kia ĐÃ có effect chưa?
```

---

## 6. Conflict Resolution Strategies

### Strategy 1: Conflict Avoidance (Tốt nhất nếu possible)

```
Ensure ALL WRITES cho 1 particular record → qua CÙNG 1 LEADER

Ví dụ: User chỉ edit own data → route requests từ user đó
   luôn đến cùng region → dùng leader trong region đó
   → Different users có different "home regions"
   → Từ góc nhìn 1 user = essentially single-leader

VẤN ĐỀ: Conflict avoidance BREAKS DOWN nếu:
   Cần CHANGE designated leader cho 1 record (region unavailable, user moved)
   → Risk: user write trong khi leader đang thay đổi → conflict

Autoincrement IDs: Leader 1 chỉ generate ODD numbers, Leader 2 generate EVEN
   → Không thể assign cùng ID cho different records
```

### Strategy 2: Last Write Wins (LWW)

```
Attach TIMESTAMP đến mỗi write
Luôn dùng value với TIMESTAMP CAO NHẤT (gần nhất)

Ví dụ: User 1 timestamp > User 2 timestamp
   → Cả 2 leaders quyết định title = B (User 1's write)
   → User 2's write bị DISCARD

Tên gọi misleading: LWW ≠ "write xảy ra sau"
THỰC CHẤT: khi có concurrent writes, 1 cái được RANDOMLY CHOSEN làm winner
            cái kia SILENTLY DISCARDED (dù đã successfully processed!)
```

**Khi nào LWW OK?**

```
Chỉ INSERT records với unique key, NEVER UPDATE → LWW không vấn đề
NHƯNG nếu UPDATE existing records → LWW = DATA LOSS

Vấn đề khác: Nếu dùng real-time clock (Unix timestamp):
   System rất nhạy với CLOCK SYNCHRONIZATION
   Node có clock đi TRƯỚC → writes của nó LUÔN WIN (kể cả writes của người khác rõ ràng đến SAU)
   Giải pháp: Dùng LOGICAL CLOCK (Chapter 9, "ID Generators and Logical Clocks")

Cassandra, ScyllaDB: dùng LWW với timestamps (mặc định)
```

### Strategy 3: Manual Conflict Resolution

```
Không muốn randomly discard writes → resolve MANUALLY

Tương tự Git merge conflicts:
   Database STORE TẤT CẢ concurrent values cho 1 record
   (gọi là SIBLINGS: vd cả B và C trong Figure 6-9)

Next time query record:
   Database RETURN TẤT CẢ sibling values
   Application code hoặc user RESOLVE:
     - Tự động trong app code (vd: concatenate B và C thành B/C)
     - Ask user to choose

   Sau khi resolve: WRITE BACK resolved value

VÍ DỤ: CouchDB dùng approach này
```

**4 Vấn đề của Manual Resolution:**

| Vấn đề                             | Chi tiết                                                                                                                         |
| ---------------------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| **API thay đổi**                   | Title từ string → set of strings (thường 1, đôi khi nhiều) → awkward trong code                                                  |
| **User burden**                    | Cần build UI cho conflict resolution + user bị confused                                                                          |
| **Shopping cart anomaly (Amazon)** | Merge bằng set union → deleted item REAPPEAR (Figure 6-10: device 1 remove Book, device 2 remove DVD, sau merge: cả 2 reappear!) |
| **Cascading conflicts**            | Nhiều nodes simultaneously resolve → conflict resolution có thể tạo ra CONFLICT MỚI (B/C vs C/B → B/C/C/B)                       |

---

## 7. Automatic Conflict Resolution (Best Approach)

### Nguyên tắc

```
Algorithm AUTOMATICALLY MERGE concurrent writes thành consistent state
TẤT CẢ replicas converge về CÙNG STATE
→ Gọi là STRONG EVENTUAL CONSISTENCY
   (kết hợp eventual consistency + convergence guarantee)
```

### Thuật toán theo loại Data

| Loại Data                                              | Thuật toán                                                                                     |
| ------------------------------------------------------ | ---------------------------------------------------------------------------------------------- |
| **Text** (title, body wiki page)                       | Detect characters inserted/deleted; merge = preserve ALL insertions/deletions                  |
| **Collection** (ordered list, unordered shopping cart) | Track insertions AND deletions → tránh Amazon shopping cart anomaly                            |
| **Counter** (number of likes, increment/decrement)     | Count increments/decrements per sibling, ADD TOGETHER → không double-count, không drop updates |
| **Key-value mapping**                                  | Apply conflict resolution per-key; different keys handled independently                        |

### CRDT vs OT — Hai Họ Thuật Toán

#### Operational Transformation (OT)

```
Dùng INDEX của characters (vị trí) để track insertions/deletions

Ví dụ (Figure 6-11): Start với "ice"
   Replica A: prepend 'n' → "nice" (insert 'n' tại index 0)
   Replica B: append '!' → "ice!" (insert '!' tại index 3)

   Exchange operations:
   'n' tại index 0: apply as-is → "nice"
   '!' tại index 3: nếu apply vào "nice" → "nic!e" (SAI!)

   → Phải TRANSFORM index: insertion của '!' → index 4
     (account cho insertion 'n' ở index 0 trước đó)
   → Kết quả: "nice!" (ĐÚNG)

Dùng bởi: Google Docs, ShareDB
```

#### CRDT (Conflict-Free Replicated Data Types)

```
Dùng UNIQUE IMMUTABLE ID cho mỗi character (thay vì index)

Ví dụ (Figure 6-11):
   Assign IDs: 1A=i, 2A=c, 3A=e
   Insert '!' sau ID 3A (character 'e')
   Insert 'n' trước character đầu tiên (preceding ID = nil)

   Concurrent insertions tại CÙNG VỊ TRÍ → ordered by character IDs
   → Replicas converge WITHOUT any transformation

Dùng bởi: Redis Enterprise, Riak, Azure Cosmos DB
          Automerge (CRDT cho JSON sync engine)
          Yjs (CRDT sync engine)
```

#### OT vs CRDT

```
Different design philosophies + performance characteristics
Cả 2 đều có thể handle text, lists, counters, key-value maps

OT: Most often for real-time collaborative text editing
CRDT: Found in distributed databases

CÓ THỂ combine advantages của cả 2 trong 1 algorithm
```

---

## 8. Types of Conflict

```
RÕAMBIG: 2 writes đồng thời modify CÙNG FIELD của cùng record
   → Rõ ràng là conflict

SUBTLE HƠN (ví dụ Meeting Room Booking):
   App insert new record cho mỗi booking
   Constraint: mỗi phòng chỉ được book bởi 1 nhóm tại 1 thời điểm

   → Conflict: 2 bookings được create cho cùng phòng cùng lúc
   → Dù app check availability trước → nếu 2 bookings gần nhau đủ
     cả 2 thấy phòng còn trống TRƯỚC KHI insert → CONFLICT

→ Không có quick fix; sẽ thấy thêm examples ở Chapter 8
   và scalable approaches ở Chapter 13
```

---

## Tóm tắt phần này

```
MULTI-LEADER REPLICATION:
   Nhiều nodes accept writes → replication giữa các leaders
   Async multi-leader (sync multi-leader ≈ single-leader)

USE CASES:
   Geo-distributed: 1 leader/region, better performance + tolerance
   Offline operation: mỗi device = 1 leader (mobile apps)
   Real-time collaboration: mỗi browser tab = 1 replica (Google Docs, Figma)

TOPOLOGIES:
   Circular/Star: SPOF risk
   All-to-all: Better fault tolerance, nhưng message overtaking issue → version vectors

SYNC ENGINE: Library cho offline-first/local-first apps
   Pros: UI nhanh, offline support, programming model đơn giản
   Cons: cần data đủ nhỏ để download toàn bộ

CONFLICT RESOLUTION (từ đơn giản → tốt):
   Conflict avoidance (best if possible, nhưng breaks down khi change leader)
   LWW (simple, nhưng random data loss, clock sync problems)
   Manual (flexible, nhưng user burden, cascading conflicts)
   Automatic: CRDT hoặc OT → Strong Eventual Consistency
```
