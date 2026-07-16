# Leaderless Replication (Dynamo-Style)

---

## 1. Giới thiệu

### Ý tưởng cốt lõi

```
Bỏ concept của LEADER hoàn toàn
BẤT KỲ replica nào cũng có thể TRỰC TIẾP accept writes từ clients

Lịch sử:
   Xuất hiện ở những replicated data systems ĐẦU TIÊN
   Bị lãng quên trong era thống trị của relational databases
   Trở lại sau khi Amazon dùng cho in-house Dynamo (2007)

   → Riak, Cassandra, ScyllaDB: open source, inspired by Dynamo
   → Còn gọi là "DYNAMO-STYLE"

KHÔNG nhầm với Amazon DynamoDB:
   DynamoDB = more recent cloud database từ Amazon
   DIFFERENT ARCHITECTURE: single-leader replication based on Multi-Paxos
```

### 2 Cách Implement Leaderless

```
1. CLIENT gửi write trực tiếp đến NHIỀU REPLICAS song song
2. COORDINATOR NODE làm thay (nhưng KHÔNG enforce ordering như leader)
```

---

## 2. Writing khi có Node Down

### Vấn đề với Single-Leader khi Node Down

```
Single-leader: node down → cần FAILOVER để tiếp tục accept writes
```

### Leaderless: Không có Failover

```
Ví dụ: 3 replicas, 1 node UNAVAILABLE

Client (user 1234) gửi write đến TẤT CẢ 3 replicas SONG SONG
2 replicas available: accept write
1 replica unavailable: miss write

Nếu 2/3 replicas acknowledge = sufficient → write CONSIDERED SUCCESSFUL
Client ignore việc 1 replica missed write

Khi unavailable node QUAY LẠI ONLINE:
   Clients bắt đầu đọc từ nó → CÓ THỂ nhận STALE VALUES

→ Để giải quyết: read requests cũng gửi đến NHIỀU NODES SONG SONG
   Nhận different responses → dùng VERSION NUMBER/TIMESTAMP để determine cái nào MỚI HƠN
```

---

## 3. Cơ chế Catch Up — 3 Mechanisms

### Read Repair

```
Client đọc từ NHIỀU NODES SONG SONG
Phát hiện replica nào có STALE response (lower version number)
→ Client write newer value NGƯỢC LẠI cho stale replica

Works well: values được đọc THƯỜNG XUYÊN
Limitation: values HIẾM KHI được đọc → có thể KHÔNG ĐƯỢC REPAIR
```

### Hinted Handoff

```
Replica X UNAVAILABLE → Replica Y (không phải replica "đúng" cho key đó)
   store write trên BEHALF của X (dưới dạng HINT)

Khi X quay lại → Y gửi hints đến X, rồi XÓA hints
→ Giúp bring replicas up-to-date ngay cả với values KHÔNG BAO GIỜ đọc

SLOPPY QUORUM (Riak, Dynamo) / CONSISTENCY LEVEL ANY (Cassandra, ScyllaDB):
   Large-scale network interruption → không thể form quorum thông thường
   → Cho phép BẤT KỲ reachable replica nào accept writes
   → KHÔNG guarantee subsequent reads sẽ thấy written value
   → Nhưng đôi khi BETTER than failing
```

### Anti-Entropy

```
BACKGROUND PROCESS định kỳ tìm differences trong data GIỮA REPLICAS
   → Copy bất kỳ missing data từ replica này sang replica kia

KHÁC VỚI replication log trong leader-based:
   - KHÔNG copy writes theo bất kỳ PARTICULAR ORDER
   - Có thể có SIGNIFICANT DELAY trước khi data được copied

→ Đảm bảo EVENTUALLY tất cả data replicated, dù values KHÔNG BAO GIỜ được đọc
```

---

## 4. Quorums for Reading and Writing

### Nguyên tắc Quorum

```
n replicas total
w nodes phải confirm write để considered successful
r nodes phải query cho mỗi read

QUORUM CONDITION: w + r > n

→ Ít nhất 1 trong r nodes mà ta đọc PHẢI có latest value
   (vì sets of w write nodes và r read nodes PHẢI OVERLAP ít nhất 1 node)

Ví dụ: n=3, w=2, r=2 → w+r=4 > 3=n ✓
   → Can tolerate 1 unavailable node

Ví dụ: n=5, w=3, r=3 → w+r=6 > 5=n ✓
   → Can tolerate 2 unavailable nodes (Figure 6-13)
```

### Cách chọn n, w, r

```
Thường: n là ODD NUMBER (3 hoặc 5)
        w = r = (n+1)/2 (rounded up) → MAJORITY quorum

VÍ DỤ KHÁC:
   Write-heavy workload: w=n, r=1
   → Reads rất NHANH (chỉ cần 1 response)
   → NHƯNG: 1 failed node làm ALL WRITES FAIL

n > number of nodes cần thiết cho 1 value:
   Dataset có thể SHARDED (Chapter 7)
   Mỗi value chỉ stored trên n nodes, không phải tất cả
```

### Reads và Writes LUÔN gửi đến TẤT CẢ n replicas

```
w và r xác định phải CHỜ BAO NHIÊU NODES respond
KHÔNG phải bao nhiêu nodes nhận request

Nếu ít hơn w hoặc r nodes available → error returned
```

---

## 5. Giới hạn của Quorum Consistency

### w + r > n KHÔNG đảm bảo perfect consistency

```
EDGE CASES where stale values có thể được trả về:

1. NODE CÓ NEW VALUE FAIL, restore từ OLD VALUE:
   → Số replicas có new value < w → quorum condition bị phá

2. REBALANCING IN PROGRESS (Chapter 7):
   Nodes có inconsistent views về replicas nào hold n copies
   → Read/write quorums có thể không overlap nữa

3. CONCURRENT READ + WRITE:
   Read có thể hoặc không thấy concurrently written value
   Possible: 1 read thấy new value, subsequent read thấy old value

4. WRITE SUCCEEDED ON SOME REPLICAS NHƯNG FAILED ON OTHERS:
   Tổng < w replicas acknowledged → reported as failed
   NHƯNG KHÔNG ROLLED BACK trên successful replicas
   → Subsequent reads: CÓ THỂ HOẶC KHÔNG thấy value từ failed write

5. REAL-TIME CLOCK TIMESTAMPS (Cassandra, ScyllaDB):
   Writes có thể bị silently dropped nếu node khác có FASTER CLOCK
   viết vào cùng key → LWW drop write từ node clock chậm hơn
   (thấy ở "Relying on Synchronized Clocks" — Chapter 9)

6. CONCURRENT WRITES:
   Có thể được processed theo DIFFERENT ORDER trên different replicas
   → Conflict → cần conflict resolution (như multi-leader)
```

> **Kết luận thực tế:** Quorum appearance to guarantee latest value, nhưng thực tế KHÔNG ĐƠN GIẢN. Dynamo-style databases tối ưu cho **eventual consistency** — w và r cho phép adjust PROBABILITY of stale values, KHÔNG phải absolute guarantee.

---

## 6. Monitoring Staleness

### Leader-Based: Dễ Monitor

```
Database expose REPLICATION LAG metrics
Leader và followers apply writes theo CÙNG THỨ TỰ, mỗi node có POSITION trong log
Lag = leader position - follower position
```

### Leaderless: Khó Monitor

```
KHÔNG có fixed order của writes being applied
→ Monitoring staleness PHỨC TẠP HƠN NHIỀU

Number of hints stored = 1 measure, NHƯNG khó interpret hữu ích

"Eventual consistency" là deliberately vague guarantee
Nhưng cho operability: cần QUANTIFY "eventual"
```

---

## 7. Single-Leader vs Leaderless Performance

### Leader-Based: Nhược điểm khi Read từ Leader

```
Read throughput LIMITED bởi leader capacity
Leader fail → phải chờ failover → temporarily increased response times
System rất nhạy với performance problems trên leader
```

### Leaderless: Ưu điểm về Resilience

```
Không có failover → 1 replica slow/unavailable = ÍT ẢNH HƯỞNG đến response times
Client dùng responses từ FASTER replicas

REQUEST HEDGING:
   Dùng fastest responses → SIGNIFICANTLY REDUCE TAIL LATENCY

Resilience đến từ: KHÔNG phân biệt giữa "normal case" và "failure case"
   Đặc biệt hữu ích với:
   - GRAY FAILURES: node không hoàn toàn down, chỉ unusual slow
   - OVERLOADED NODES: node offline lâu → recovery via hinted handoff tạo thêm load

Leader-based: phải decide có failover không (có thể gây thêm disruption)
Leaderless: question này NOT EVEN ARISE
```

### Leaderless: Nhược điểm

```
1. DETECTING unavailable replica để store hints:
   Khi unavailable replica quay lại → handoff process adds load
   System đã under strain → thêm load

2. LARGER QUORUMS = SLOW:
   Nhiều replicas hơn → bigger quorums → chờ nhiều responses hơn
   Cao hơn r/w → probability hit slow replica cao hơn → response time tăng
   Practical limit: quorums hiếm khi hơn 4/7 hoặc 5/9 nodes

3. NETWORK INTERRUPTION LARGE-SCALE:
   Disconnect từ nhiều replicas → không thể form quorum
   Sloppy quorum (fallback) có thể allow writes nhưng không guaranteed reads thấy chúng
```

### So sánh tổng quan

```
Single-Leader:
   ✓ Strong consistency guarantees
   ✓ Simple to understand
   ✗ All writes through leader → bottleneck
   ✗ Failover: downtime, risk of data loss, split brain

Multi-Leader:
   ✓ Greater resilience vs network interruptions
   ✓ Reads/writes chỉ cần 1 leader (co-located với client)
   ✗ Reads có thể arbitrarily out-of-date
   ✗ Conflict resolution complexity

Leaderless:
   ✓ Most resilient (no failover, request hedging)
   ✓ Good fault tolerance
   ✗ Reads có thể stale (weaker consistency than single-leader)
   ✗ Harder to monitor
   ✗ Quorum writes/reads → multi-node overhead
```

---

## 8. Multi-Region Operation với Leaderless

### Cassandra/ScyllaDB: Coordinator-based

```
Client → gửi đến LOCAL COORDINATOR NODE (cùng region)
Coordinator → forward write đến:
   TẤT CẢ replicas trong LOCAL REGION
   1 REPLICA trong MỖI REGION KHÁC (rồi replica đó forward đến các replicas còn lại)

Optimization: Tránh cross-region request NHIỀU LẦN

CONSISTENCY LEVELS cho phép chọn:
   - Quorum across TẤT CẢ regions (chờ lâu nhất)
   - Separate quorum trong MỖI region
   - Quorum chỉ trong LOCAL REGION (nhanh nhất, nhưng stale risk cao hơn)
```

### Riak: Region-Local

```
Tất cả communication client-database GIỮA LOCAL REGION
n = số replicas TRONG 1 REGION
Cross-region replication: async background, similar to multi-leader
```

---

## Tóm tắt phần này

```
LEADERLESS REPLICATION (Dynamo-Style):
   BẤT KỲ replica nào accept writes
   Không có failover
   Read từ NHIỀU NODES song song (detect stale values qua version numbers)

3 CATCH-UP MECHANISMS:
   Read Repair: client fix stale replicas khi đọc (works for frequently-read values)
   Hinted Handoff: replica khác hold hints, deliver khi target quay lại
   Anti-entropy: background process sync replicas (no ordering, may delay)

QUORUMS: n replicas, w write confirmations, r read nodes
   w + r > n → guarantee overlap → likely up-to-date reads
   Thực tế: edge cases vẫn có thể return stale values

   Không phải absolute guarantee → optimize probability, not certainty

PERFORMANCE:
   ✓ Resilient: no failover, request hedging → low tail latency
   ✓ Gray failure tolerant (không phân biệt normal/failure case)
   ✗ Larger quorums → slower; hints recovery → thêm load; monitoring khó

Multi-region:
   Cassandra/ScyllaDB: coordinator + cross-region replication
   Riak: region-local, async cross-region (similar to multi-leader)
```
