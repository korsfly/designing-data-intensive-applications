# Single-Leader Replication

---

## 1. Mô hình cơ bản

> Còn gọi: **leader-based**, **primary-backup**, **active/passive** replication.  
> (Tránh dùng: "master-slave" — thuật ngữ này bị coi là offensive)

### 3 Nguyên tắc hoạt động

```
1. LEADER (Primary/Source):
   Khi client muốn WRITE → gửi request đến leader
   Leader viết data vào local storage TRƯỚC

2. FOLLOWERS (Read replicas, Secondaries, Hot standbys):
   Leader gửi data change đến TẤT CẢ followers (replication log / change stream)
   Mỗi follower apply writes theo CÙNG THỨ TỰ như leader

3. READS:
   Client có thể read từ LEADER hoặc BẤT KỲ follower nào
   NHƯNG: writes chỉ accepted bởi LEADER
```

### Hệ thống sử dụng Single-Leader Replication

```
Relational DB: PostgreSQL, MySQL, Oracle Data Guard, SQL Server Always On
Document DB: MongoDB, DynamoDB
Message brokers: Kafka
Block devices: DRBD
Consensus algorithms (tự elect leader mới nếu cũ fail):
   Raft → CockroachDB, TiDB, etcd, RabbitMQ quorum queues
```

### Nếu database bị sharded

```
Mỗi shard có 1 leader riêng
Different shards có thể có leaders trên different nodes
Mỗi shard vẫn chỉ có ĐÚNG 1 leader node
```

---

## 2. Replication: Synchronous vs Asynchronous

### Synchronous Replication

```
Leader CHỜ follower confirm đã nhận write
TRƯỚC KHI báo success cho client

Ưu điểm: Follower GUARANTEED có up-to-date data nhất quán với leader
Nhược điểm: Nếu follower không respond (crash, network fault)
             → Write KHÔNG THỂ process
             → Leader phải BLOCK TẤT CẢ writes cho đến khi follower available lại
```

### Asynchronous Replication

```
Leader gửi message đến follower NHƯNG không chờ response

Ưu điểm: Leader tiếp tục process writes dù TẤT CẢ followers đã fall behind
Nhược điểm: Nếu leader fail và KHÔNG thể recover
             → Bất kỳ write nào chưa replicated sang follower = BỊ MẤT
             → Write không guaranteed durable dù đã được confirm với client!
```

> **Asynchronous replication được dùng phổ biến**, đặc biệt khi có nhiều followers hoặc geographically distributed. Tuy nhiên phải hiểu rõ trade-off về durability.

### Semi-Synchronous (Thực tế phổ biến)

```
Thực tế: Nếu DB offer "synchronous replication" → thường có nghĩa:
   1 follower là SYNCHRONOUS
   Các follower khác là ASYNCHRONOUS

Nếu synchronous follower slow/unavailable:
   → 1 trong số asynchronous followers được PROMOTE lên synchronous

→ Luôn có up-to-date copy trên ÍT NHẤT 2 nodes: leader + 1 synchronous follower

QUORUM VARIANT: majority of replicas (vd: 3/5) updated synchronously
   → Sẽ thảo luận thêm ở phần Leaderless Replication
```

---

## 3. Setting Up New Followers

### Vấn đề: Không thể chỉ Copy Files

```
Database luôn thay đổi → Standard file copy thấy DIFFERENT PARTS
của database tại DIFFERENT POINTS IN TIME → kết quả KHÔNG có nghĩa

Lock database? → ĐI NGƯỢC lại mục tiêu high availability
```

### Quy trình Setup Follower (Không cần Downtime)

```
1. SNAPSHOT: Take consistent snapshot của leader database
   (không lock toàn bộ nếu có thể)
   Hầu hết DB có feature này (cũng cần cho backups)
   Một số cần tools như Percona XtraBackup (MySQL)

2. COPY: Copy snapshot sang new follower node

3. CATCH UP: Follower kết nối leader, request TẤT CẢ data changes
   từ lúc snapshot được lấy
   Cần snapshot được associate với EXACT POSITION trong replication log:
   - PostgreSQL: log sequence number (LSN)
   - MySQL: binlog coordinates HOẶC global transaction identifiers (GTIDs)

4. LIVE: Follower đã "caught up", tiếp tục nhận data changes theo real-time
```

### Lưu trữ trên Object Storage

```
Có thể ARCHIVE replication log + periodic snapshots vào object store

Setting up new follower = download files từ object store
Tools: WAL-G (PostgreSQL, MySQL, SQL Server), Litestream (SQLite)
```

---

## 4. Databases Backed by Object Storage

### Lợi ích

```
Object stores (S3, GCS, Azure Blob) cho database storage có nhiều ưu điểm:
   1. RẺ hơn cloud storage khác → store less-accessed data ở đây
   2. Multi-zone/multi-region replication với HIGH DURABILITY built-in
   3. Conditional write feature (compare-and-set) → implement transactions + leader election
   4. Storing data từ nhiều DBs cùng chỗ → simplify data integration
      (đặc biệt với open formats như Parquet, Iceberg)
```

### Trade-offs

```
NHƯỢC ĐIỂM:
   Much HIGHER read/write latency so với local disk
   Per-API-call fee → phải BATCH reads/writes → tăng latency thêm
   Objects thường IMMUTABLE → random writes vào object lớn = rất tốn kém
   Không có standard filesystem interface
      (FUSE giúp mount như filesystem, nhưng thiếu POSIX features)
```

### Các giải pháp kiến trúc

| Kiến trúc                         | Mô tả                                                                                             |
| --------------------------------- | ------------------------------------------------------------------------------------------------- |
| **Tiered storage**                | Less-accessed data → object store; hot data → SSD/NVMe/memory                                     |
| **Object storage + separate WAL** | Object storage cho primary storage, low-latency storage (EBS, Neon's Safekeepers) cho WAL         |
| **Zero-Disk Architecture (ZDA)**  | Persist ALL data vào object storage; disk/memory chỉ cho caching; nodes KHÔNG có persistent state |

```
ZDA examples: WarpStream, Confluent Freight, Bufstream, Redpanda Serverless
   (Kafka-compatible, built trên ZDA)
Nearly every modern cloud data warehouse cũng dùng architecture này
```

---

## 5. Handling Node Outages

### Follower Failure: Catch-up Recovery

```
Mỗi follower giữ LOG of data changes nhận từ leader trên local disk

Follower crash/restart hoặc network interruption:
   → Follower biết LAST TRANSACTION đã process từ log
   → Reconnect leader, request TẤT CẢ changes từ sau điểm đó
   → "Catch up" rồi tiếp tục nhận changes bình thường

Challenge về performance:
   High write throughput hoặc follower offline lâu → NHIỀU writes để catch up
   High load trên CẢ recovering follower VÀ leader (phải gửi backlog)

Leader dilemma về log retention:
   Nếu follower unavailable lâu → leader phải CHỌN:
   RETAIN log cho đến khi follower recover (risk: hết disk space trên leader)
   HOẶC DELETE log → follower phải restore từ backup khi quay lại
```

### Leader Failure: Failover

```
Một trong các followers cần được PROMOTED làm new leader
Clients cần reconfigure để gửi writes đến new leader
Followers khác cần consume data changes từ new leader

Failover có thể MANUAL (admin quyết định) hoặc AUTOMATIC

AUTOMATIC FAILOVER gồm 3 bước:
   1. DETECT: Leader failed? Dùng TIMEOUT (vd: 30s không respond = dead)
              (Nếu leader được take down có kế hoạch: trigger safe handoff trước)

   2. CHOOSE NEW LEADER: Election (majority of remaining replicas)
                          HOẶC appointed bởi controller node
                          Best candidate: replica có MOST UP-TO-DATE data
                          (minimize data loss)
                          Đây là CONSENSUS PROBLEM → Chapter 10

   3. RECONFIGURE: Clients → send writes đến new leader
                   Old leader quay lại → phải recognize new leader và trở thành follower
```

### Các rủi ro của Failover

| Rủi ro                                  | Chi tiết                                                                                                                                                                                |
| --------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Data loss với async replication**     | New leader có thể chưa nhận TẤT CẢ writes từ old leader → unreplicated writes bị DISCARD                                                                                                |
| **Primary key reuse (GitHub incident)** | Out-of-date MySQL follower promoted → reused primary keys đã dùng → inconsistency với Redis → private data disclosed cho wrong users                                                    |
| **Split brain**                         | 2 nodes đều believe they are leader → cả 2 accept writes → data LOST/CORRUPTED. Safety mechanism: shutdown 1 node nếu phát hiện 2 leaders — nhưng nếu không cẩn thận: CẢ 2 bị shutdown! |
| **Timeout sai**                         | Quá ngắn: unnecessary failovers khi load spike hoặc network glitch. Quá dài: slow recovery khi leader thực sự fail                                                                      |

> **Fencing:** Kỹ thuật ngăn split brain bằng cách limit/shutdown old leaders. Chi tiết ở "Distributed Locks and Leases" (Chapter 9).

> **Kết luận thực tế:** Nhiều operations teams PREFER manual failovers dù software support automatic failover, vì các vấn đề này chưa có giải pháp đơn giản.

---

## 6. Implementation of Replication Logs

### Statement-Based Replication

```
Leader log EVERY WRITE REQUEST (SQL statement), gửi đến followers
Mỗi follower parse và execute statement như thể nhận từ client

VẤN ĐỀ:
   - Nondeterministic functions: NOW(), RAND() → khác nhau trên mỗi replica
   - Autoincrement columns, UPDATE WHERE <condition>:
     phải execute ĐÚNG THỨ TỰ (limit với concurrent transactions)
   - Side effects: triggers, stored procedures, UDF → khác nhau trên mỗi replica

WORKAROUND: Leader replace nondeterministic function calls với
   FIXED RETURN VALUE khi log statement

Dùng bởi: MySQL < 5.1, VoltDB (yêu cầu transactions deterministic)

Tương đồng với: Event Sourcing model (Chapter 3), State Machine Replication
```

### Write-Ahead Log (WAL) Shipping

```
WAL (đã thấy trong Chapter 4) chứa TẤT CẢ thông tin để
restore indexes và heap về consistent state sau crash

→ Dùng CÙNG LOG để build replica trên another node:
   Leader ghi log lên disk + gửi qua network đến followers
   Followers process log → build copy GIỐNG HỆT files trên leader

Dùng bởi: PostgreSQL, Oracle

NHƯỢC ĐIỂM:
   Log mô tả data ở VERY LOW LEVEL (bytes changed in which disk blocks)
   → TIGHTLY COUPLED với storage engine
   → Không thể run DIFFERENT VERSIONS của DB software trên leader và followers
   → WAL shipping thường KHÔNG cho phép version mismatch
      → Upgrades cần DOWNTIME
```

### Logical (Row-Based) Log Replication

```
Dùng DIFFERENT LOG FORMAT cho replication và storage engine
→ DECOUPLE replication log khỏi storage engine internals

Logical log = sequence of records mô tả writes theo GRANULARITY OF ROW:
   INSERT: log new values của TẤT CẢ columns
   DELETE: log đủ info để identify row (primary key, hoặc all columns nếu không có PK)
   UPDATE: log row identifier + new values của columns đã thay đổi
   TRANSACTION COMMIT: log record đặc biệt

MySQL: binlog (separate logical replication log, bên cạnh WAL)
PostgreSQL: logical replication qua decoding physical WAL thành row events

NHƯỢC ĐIỂM:
   Logical log là BACKWARD COMPATIBLE hơn WAL shipping
   → Leader và follower CÓ THỂ run DIFFERENT DATABASE VERSIONS
   → Cho phép upgrade với MINIMAL DOWNTIME

   Dễ PARSE hơn cho external applications:
   → Gửi database contents đến data warehouse, custom indexes, caches
   → Technique này gọi là CHANGE DATA CAPTURE (CDC)
   → Chi tiết Chapter 12
```

---

## Tóm tắt phần này

```
SINGLE-LEADER REPLICATION:
   Tất cả writes → 1 leader, followers nhận stream of changes
   Reads: từ bất kỳ replica (followers có thể STALE)

SYNC vs ASYNC:
   Sync: Durability tốt hơn, nhưng blocking nếu follower down
   Async: Phổ biến hơn, nhưng writes có thể MẤT khi leader fail
   Semi-sync (thực tế): 1 sync follower, còn lại async

SETUP NEW FOLLOWER: Snapshot → Copy → Catch up (không cần downtime)

OBJECT STORAGE: Rẻ, high durability, nhưng latency cao hơn. ZDA = nodes stateless

FAILOVER RISKS: Data loss, split brain, GitHub primary-key incident

3 REPLICATION LOG METHODS:
   Statement-based: compact nhưng nondeterminism vấn đề
   WAL shipping: low-level, tightly coupled, khó upgrade
   Logical (row-based): decoupled, backward compatible, dễ CDC
```
