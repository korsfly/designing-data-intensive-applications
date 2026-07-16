# Operations: Rebalancing và Request Routing

---

## 1. Automatic vs Manual Rebalancing

### Câu hỏi quan trọng

```
Khi nào shard splitting/rebalancing xảy ra?
   → Tự động (automatic)?
   → Hay do admin cấu hình thủ công (manual)?
```

### Fully Automatic Rebalancing

```
System TỰ QUYẾT ĐỊNH:
   - Khi nào split shards
   - Move shards từ node nào đến node nào
   - Không cần human interaction

ƯU ĐIỂM:
   Less operational work cho normal maintenance
   Có thể AUTOSCALE: tự thêm/bớt shards theo workload
   DynamoDB: có thể add/remove shards trong vài phút

NHƯỢC ĐIỂM: UNPREDICTABLE
   Rebalancing = EXPENSIVE operation:
      Reroute requests
      Move LARGE AMOUNT OF DATA giữa nodes
   → Nếu không cẩn thận:
      Overload network/nodes
      Harm performance của requests khác

   Nếu system near MAX WRITE THROUGHPUT:
      Shard-splitting process có thể KHÔNG THEO KỊP rate of incoming writes

   NGUY HIỂM KẾT HỢP với automatic failure detection:
      Node overloaded → respond slow → other nodes assume it's DEAD
      → Automatically rebalance: move load AWAY FROM IT
      → Thêm load lên other nodes VÀ network → SitUATION WORSE
      → Risk: CASCADING FAILURE
        (other nodes cũng bị overload → bị nghi là dead → lại rebalance tiếp)
```

### Manual Rebalancing

```
Admin explicitly cấu hình shard assignment

CHẬM HƠN automatic, NHƯNG:
   Help PREVENT operational surprises
   Useful cho PROACTIVE REBALANCING:
      Biết trước traffic surge (Cyber Monday, World Cup ticket sales,...)
      → Manually rebalance TRƯỚC khi event xảy ra
```

### Middle Ground (Best of both worlds)

```
Couchbase, Riak:
   System GENERATE suggested shard assignment automatically
   NHƯNG: Admin phải COMMIT trước khi có hiệu lực
   → Human oversight + automation convenience
```

---

## 2. Request Routing — Vấn đề cốt lõi

### Câu hỏi

```
Khi muốn read/write 1 particular key → cần connect đến NODE NÀO?

Điều này gọi là REQUEST ROUTING, tương tự SERVICE DISCOVERY (Chapter 5)
NHƯNG KHÁC BIỆT QUAN TRỌNG:
   Services: mỗi instance STATELESS → LB có thể send request đến bất kỳ instance nào
   Sharded DB: request cho key K chỉ có thể handle bởi NODE là replica cho SHARD chứa K
   → Routing PHẢI BIẾT: key → shard → node mapping
```

### 3 Approaches (Figure 7-7)

#### Approach 1: Contact Any Node (Round-Robin LB)

```
Client → Round-Robin Load Balancer → BẤT KỲ NODE nào

Nếu node đó sở hữu shard → HANDLE directly
Nếu không → NODE FORWARD request đến ĐÚNG node → nhận reply → pass lại client

Ví dụ: Cassandra, ScyllaDB, Riak (gossip-based), CockroachDB, TiDB
```

#### Approach 2: Routing Tier

```
Tất cả requests → ROUTING TIER trước
Routing tier: KHÔNG handle requests (no DB logic)
   Chỉ là SHARD-AWARE LOAD BALANCER
   → Determine đúng node → forward

Ví dụ: MongoDB (mongos daemons), Redis Cluster proxy, twemproxy
```

#### Approach 3: Shard-Aware Client

```
Client BIẾT về sharding và assignment
→ Connect TRỰC TIẾP đến đúng node, không qua intermediary

Ví dụ: Hbase clients (biết ZooKeeper), Cassandra client with token-aware routing
```

---

## 3. Coordination Services: ZooKeeper, etcd, Raft

### Vấn đề chung của cả 3 approaches

```
BẤT KỂ ai thực hiện routing (node, routing tier, hay client):
   Cần biết: KEY → SHARD → NODE mapping

Khi assignment thay đổi (shard moved, node added/removed):
   Routing component phải được UPDATE

Ai quyết định shard lives ở node nào?
   → Cần COORDINATOR → NHƯNG nếu coordinator down thì sao?
   → Cần FAULT-TOLERANT coordinator: dùng CONSENSUS ALGORITHM (Chapter 10)
```

### ZooKeeper/etcd Approach (Figure 7-8)

```
Databases dùng SEPARATE COORDINATION SERVICE:
   ZooKeeper (HBase, SolrCloud)
   etcd (Kubernetes)
   (cả 2: consensus protocols để provide fault tolerance + split brain protection)

HOW IT WORKS:
   1. Mỗi node REGISTERS itself trong ZooKeeper/etcd
   2. ZooKeeper/etcd maintains AUTHORITATIVE mapping: shards → nodes
   3. Routing tier/shard-aware clients SUBSCRIBE to this information
   4. Khi shard changes ownership/node added/removed:
      → ZooKeeper/etcd NOTIFIES routing tier
      → Routing tier UPDATE routing information

ADDRESSES:
   - DNS cho initial discovery của ZooKeeper/etcd nodes
   - ZooKeeper/etcd cho dynamic shard-to-node assignment
```

### Raft Built-in (Kafka, YugabyteDB, TiDB, ScyllaDB)

```
Dùng BUILT-IN implementation của Raft consensus protocol
   → Perform coordination function WITHOUT separate ZooKeeper
   → Kafka KRaft mode (Kafka Raft) eliminates ZooKeeper dependency

MongoDB:
   Own config server implementation
   + mongos daemons làm routing tier
```

### Gossip Protocol Approach (Riak)

```
KHÁC BIỆT: Riak dùng GOSSIP PROTOCOL giữa các nodes
   → Disseminate any changes trong cluster state

WEAKER CONSISTENCY than consensus protocol:
   POSSIBLE: Split brain → different parts of cluster có DIFFERENT NODE ASSIGNMENTS
             cho cùng shard

Tại sao Riak chấp nhận điều này?
   Riak = leaderless database (Chapter 6) → weak consistency guarantees anyway
   → Weak routing consistency KHÔNG tệ hơn consistency của data operations
```

### Cutover Period Challenge

```
Khi shard đang được MOVED từ node A sang node B:
   New node B đã take over
   NHƯNG: Requests đến OLD NODE A vẫn còn in flight

Cần mechanism để handle "in flight" requests:
   Forwarding từ A đến B trong transition period
   HOẶC: Client retry sau khi nhận error từ A
```

---

## 4. Request Routing trong Analytical DBs

```
Phần trên tập trung vào OLTP sharded databases
(finding shard for individual key)

ANALYTICAL DATABASES cũng dùng sharding NHƯNG:
   Query KHÔNG chỉ cần 1 shard
   → Query cần AGGREGATE VÀ JOIN data từ NHIỀU SHARDS SONG SONG
   → PARALLEL QUERY EXECUTION

Chi tiết về parallel query execution: Chapter 11
```

---

## Tóm tắt phần này

```
REBALANCING:
   Automatic: convenient nhưng unpredictable, risk cascading failure
   Manual: slower, prevents surprises, good for planned events
   Middle ground (Couchbase, Riak): auto-suggest + human commit

3 REQUEST ROUTING APPROACHES:
   1. Any node + forwarding (Cassandra, Riak)
   2. Routing tier (MongoDB mongos, Redis Cluster proxy)
   3. Shard-aware client (HBase, Cassandra token-aware)

COORDINATION:
   ZooKeeper/etcd: Separate coordination service với consensus (HBase, SolrCloud, K8s)
   Raft built-in: Kafka KRaft, YugabyteDB, TiDB, ScyllaDB
   MongoDB: own config server + mongos
   Riak: gossip protocol (weaker, but OK for leaderless DB)

CHALLENGES:
   Who decides shard assignment? → Fault-tolerant coordinator
   Routing components must track changes → Subscribe to coordinator
   Cutover period: in-flight requests during shard move
```
