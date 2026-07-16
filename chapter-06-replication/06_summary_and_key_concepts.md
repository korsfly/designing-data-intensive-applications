# Summary & Key Concepts — Chapter 6

---

## 1. Tổng hợp toàn chương

Chương 6 khám phá **Replication** — cách giữ cùng một data trên nhiều máy. Dù khái niệm đơn giản, việc xử lý khi data thay đổi liên tục và khi có lỗi xảy ra là **cực kỳ phức tạp**.

### 5 Mục tiêu của Replication

```
1. HIGH AVAILABILITY    — Tiếp tục hoạt động khi machine/zone/region fail
2. DURABILITY           — Không mất data khi machine fail vĩnh viễn
3. DISCONNECTED OPERATION — Tiếp tục hoạt động dù bị interrupt
4. LATENCY              — Đặt data gần user
5. SCALABILITY          — Tăng read throughput qua nhiều replicas
```

### 3 Kiến trúc chính + trade-offs

```
SINGLE-LEADER:
   ✓ Strong consistency, simple, dễ hiểu
   ✗ Writes bottleneck qua leader; failover risks; data loss risk

MULTI-LEADER:
   ✓ Geo-distributed performance, disconnected operation, real-time collab
   ✗ Weak consistency, conflict resolution complexity, "dangerous territory"

LEADERLESS (Dynamo-style):
   ✓ Most resilient (no failover, request hedging), gray failure tolerant
   ✗ Weakest consistency, harder to monitor, quorum overhead
```

---

## 2. Consistency Models nổi bật trong chương

| Model                                   | Đảm bảo gì                                                        |
| --------------------------------------- | ----------------------------------------------------------------- |
| **Eventual Consistency**                | Replicas eventually converge nếu dừng writes                      |
| **Read-After-Write (Read-Your-Writes)** | User luôn thấy data mà chính họ submit                            |
| **Monotonic Reads**                     | User không thấy data "đi ngược thời gian"                         |
| **Consistent Prefix Reads**             | Data appear theo thứ tự có nghĩa về causal                        |
| **Strong Eventual Consistency**         | Eventual consistency + convergence guarantee (CRDT)               |
| **Linearizability**                     | Strongest: mọi operation appear xảy ra tại 1 instant (Chapter 10) |

---

## 3. Bảng thuật ngữ (Glossary)

### Replication Basics

| Thuật ngữ                                        | Định nghĩa                                                   |
| ------------------------------------------------ | ------------------------------------------------------------ |
| **Replication**                                  | Giữ bản copy của cùng data trên nhiều machines               |
| **Leader / Primary / Source**                    | Node duy nhất accept writes (single-leader)                  |
| **Follower / Replica / Secondary / Hot standby** | Nhận stream of changes từ leader, server reads               |
| **Replication log / Change stream**              | Stream of data change events từ leader đến followers         |
| **Synchronous replication**                      | Leader chờ follower confirm trước khi báo success cho client |
| **Asynchronous replication**                     | Leader không chờ follower, gửi tiếp                          |
| **Semi-synchronous**                             | 1 follower sync, còn lại async                               |
| **Log Sequence Number (LSN)**                    | PostgreSQL's position trong replication log                  |
| **GTID**                                         | Global Transaction ID trong MySQL                            |
| **WAL shipping**                                 | Gửi Write-Ahead Log từ leader đến followers                  |
| **Logical replication**                          | Row-based log, decoupled từ storage engine internals         |
| **Change Data Capture (CDC)**                    | Capture row-level changes, gửi đến external systems          |
| **Statement-based replication**                  | Gửi SQL statements (vấn đề với nondeterminism)               |
| **Tiered storage**                               | Hot data → SSD, cold data → object store                     |
| **Zero-Disk Architecture (ZDA)**                 | ALL data persisted to object store; nodes stateless          |
| **Fencing**                                      | Kỹ thuật ngăn split brain, shutdown old leaders              |
| **Failover**                                     | Promote follower lên làm new leader khi leader fail          |
| **Split brain**                                  | 2 nodes đều believe they are leader → data corruption        |

### Replication Lag & Consistency

| Thuật ngữ                        | Định nghĩa                                                            |
| -------------------------------- | --------------------------------------------------------------------- |
| **Replication lag**              | Delay từ khi write xảy ra trên leader đến khi reflected trên follower |
| **Eventual consistency**         | Nếu dừng writes, followers eventually catch up (vague guarantee)      |
| **Read-after-write consistency** | User luôn thấy data mình đã submit                                    |
| **Monotonic reads**              | User không thấy time go backward giữa các reads                       |
| **Consistent prefix reads**      | Observer thấy writes theo đúng causal order                           |
| **Stale read**                   | Đọc data cũ từ replica chưa caught up                                 |
| **Availability zone / Zone**     | Mỗi datacenter riêng biệt trong cùng region                           |
| **Region**                       | 1+ datacenters trong cùng geographic location                         |
| **NewSQL**                       | Scalable databases với strong consistency + ACID transactions         |

### Multi-Leader

| Thuật ngữ                                        | Định nghĩa                                                      |
| ------------------------------------------------ | --------------------------------------------------------------- |
| **Multi-leader / Active-active / Bidirectional** | Nhiều nodes accept writes, replicate đến nhau                   |
| **Circular topology**                            | A→B→C→A: 1 node fail interrupt toàn bộ                          |
| **Star topology**                                | Root forward đến tất cả, có thể generalize thành tree           |
| **All-to-all topology**                          | Mỗi leader gửi đến mọi leader khác; best fault tolerance        |
| **Sync engine**                                  | Library hỗ trợ offline-first / real-time collaboration          |
| **Offline-first**                                | App tiếp tục hoạt động khi offline                              |
| **Local-first software**                         | Offline-first + không phụ thuộc vào specific online service     |
| **Netcode**                                      | Sync engine equivalent trong game development                   |
| **Concurrent writes**                            | 2 writes khi neither was aware of the other                     |
| **Sibling**                                      | Concurrent written values cùng 1 key (cần merge)                |
| **Conflict avoidance**                           | Route all writes cho 1 record qua cùng 1 leader                 |
| **Last Write Wins (LWW)**                        | Highest timestamp wins (random data loss với concurrent writes) |
| **Manual conflict resolution**                   | Store siblings, application/user resolve                        |
| **Strong eventual consistency**                  | Eventual consistency + convergence guarantee                    |
| **CRDT** (Conflict-free Replicated Data Types)   | Automatic merge qua unique character IDs                        |
| **OT** (Operational Transformation)              | Automatic merge qua index transformation                        |

### Leaderless

| Thuật ngữ                  | Định nghĩa                                                              |
| -------------------------- | ----------------------------------------------------------------------- |
| **Leaderless replication** | Bất kỳ replica nào accept writes, không có failover                     |
| **Dynamo-style**           | Leaderless inspired by Amazon's Dynamo (Riak, Cassandra, ScyllaDB)      |
| **Read repair**            | Client phát hiện stale replica, write newer value lại                   |
| **Hinted handoff**         | Replica khác hold writes cho unavailable replica                        |
| **Sloppy quorum**          | Allow writes đến bất kỳ reachable replica khi không đủ quorum           |
| **Anti-entropy**           | Background process sync missing data giữa replicas                      |
| **Quorum**                 | Tập hợp w (write) hoặc r (read) nodes đủ để complete operation          |
| **Quorum condition**       | w + r > n → guarantee at least 1 overlap                                |
| **Request hedging**        | Dùng fastest r responses → giảm tail latency                            |
| **Gray failure**           | Node không hoàn toàn down, chỉ unusually slow                           |
| **Coordinator node**       | Node nhận client request, forward đến replicas (không enforce ordering) |

### Concurrent Write Detection

| Thuật ngữ                 | Định nghĩa                                                                                   |
| ------------------------- | -------------------------------------------------------------------------------------------- |
| **Happens-before**        | A happens before B nếu B knows about/depends on A                                            |
| **Concurrent**            | Neither operation happens before the other                                                   |
| **Version number**        | Per-key counter; increment on each write                                                     |
| **Version vector**        | Collection của version numbers từ tất cả replicas                                            |
| **Dotted version vector** | Variant dùng trong Riak 2.0                                                                  |
| **Causal context**        | Riak's encoding của version vector                                                           |
| **Vector clock**          | Similar to version vector nhưng khác mục đích (ordering events, không so sánh replica state) |

---

## 4. Mindmap khái niệm

```
                          REPLICATION
                               │
          ┌────────────────────┼─────────────────────┐
          │                    │                     │
   SINGLE-LEADER         MULTI-LEADER           LEADERLESS
          │                    │                (Dynamo-style)
   ┌──────┴──────┐      ┌──────┴──────┐             │
   │             │      │             │         ┌────┴────┐
 Sync/     Failover  Geo-     Conflict       Quorum   Catch-up
 Async   (risks:    Distr.   Resolution        │    mechanisms
   │    split brain Offline  │                 │    │    │
   │    data loss)  Collab  LWW CRDT OT      w+r   Read  Anti-
 Repl.     │         │        │          │    >n  Repair entropy
  logs   Fencing  Sync     Sibling   Conflict │         Hinted
   │     WAL-G    Engine    merge     avoidance│        Handoff
   │               │        │              Edge     │
  WAL    Logical   Local-  String    cases:     Request
 Ship.   Replic.   first   Text     clock,     Hedging
  CDC     ZDA      Autom   Counter   rebalance
                   erge    K-V map
          │
    Replication Lag
    Problems
    ┌──────┴──────┐
    │      │      │
   Read- Mono- Consist.
  After-  tonic  Prefix
  Write  Reads   Reads
    │             │
  NewSQL       Causal
  (strong)    Dependency
              Tracking
                  │
           HAPPENS-BEFORE
                  │
           VERSION VECTORS
           (Multi-replica)
```

---

## 5. Câu hỏi tự kiểm tra (Self-Check Questions)

1. Liệt kê **5 lý do** cần Replication. Tại sao chúng đôi khi conflict với nhau?

2. So sánh **Synchronous vs Asynchronous replication** về durability và availability. Tại sao **semi-synchronous** phổ biến trong thực tế?

3. Mô tả **quy trình 4 bước** setup new follower mà không cần downtime. LSN và GTID dùng để làm gì?

4. Giải thích **Zero-Disk Architecture (ZDA)**. Nó tận dụng lợi thế nào của object storage?

5. Liệt kê **4 rủi ro của automatic failover** trong single-leader replication (split brain, data loss, GitHub incident, sai timeout).

6. So sánh **3 implementation của replication log**: Statement-based, WAL shipping, Logical (row-based). Cái nào tốt nhất cho rolling upgrades và CDC?

7. Giải thích **3 anomaly của replication lag** (read-after-write, monotonic reads, consistent prefix reads). Cung cấp ví dụ cụ thể và giải pháp cho từng loại.

8. Phân biệt **Availability Zone** và **Region**. Multi-zone vs Multi-region deploy khác nhau điểm gì?

9. Giải thích **4 trade-offs của Multi-Leader vs Single-Leader** trong geo-distributed setup (performance, regional outage tolerance, network problem tolerance, consistency).

10. Mô tả **3 multi-leader topologies** (circular, star, all-to-all). Vấn đề của circular/star là gì? Vấn đề đặc biệt của all-to-all là gì?

11. Tại sao **Sync Engine** cho phép UI nhanh hơn và offline operation? Giới hạn của nó là gì?

12. So sánh **LWW, Manual resolution, và Automatic resolution (CRDT/OT)** cho conflict resolution. Khi nào nên dùng mỗi loại?

13. Phân biệt **CRDT** và **OT**. Cho ví dụ cụ thể từ Figure 6-11 (merge "nice" và "ice!" thành "nice!").

14. Giải thích **Leaderless replication**: Không có failover nghĩa là gì? Làm sao detect stale values khi đọc?

15. Mô tả **3 mechanisms** để catch up sau khi node quay lại (read repair, hinted handoff, anti-entropy).

16. Giải thích **quorum condition** (w + r > n). Cho ví dụ với n=5, w=3, r=3. Tại sao đây vẫn KHÔNG phải absolute guarantee?

17. Giải thích **request hedging** trong leaderless. Tại sao nó giảm được tail latency?

18. Định nghĩa **"happens-before"** và **"concurrent"**. Tại sao physical time KHÔNG quan trọng khi define concurrency?

19. Mô tả **algorithm capture happens-before** (single replica). Tại sao client phải read trước khi write? Điều gì xảy ra khi write không include version number?

20. Phân biệt **Version Vector** và **Vector Clock**. Tại sao Version Vector là right data structure để compare state of replicas?

---

## 6. Liên kết tới các chương khác

| Chapter 6 đề cập                             | Mở rộng ở                                            |
| -------------------------------------------- | ---------------------------------------------------- |
| ACID Transactions (strong consistency)       | **Chapter 8** — Transactions                         |
| Linearizability, consensus (leader election) | **Chapter 10** — Consistency and Consensus           |
| Distributed Locks (fencing)                  | **Chapter 9** — The Trouble with Distributed Systems |
| Unreliable clocks (LWW với real-time clock)  | **Chapter 9** — "Unreliable Clocks"                  |
| Sharding + replication combined              | **Chapter 7** — Sharding                             |
| Change Data Capture (logical replication)    | **Chapter 12** — Stream Processing                   |
| Conflict detection và resolution (scalable)  | **Chapter 13**                                       |
| WAL, crash recovery, durability              | **Chapter 8** — Transactions                         |

---

## 7. Trích dẫn mở đầu chương (Epigraph)

> _"The major difference between a thing that might go wrong and a thing that cannot possibly go wrong is that when a thing that cannot possibly go wrong goes wrong, it usually turns out to be impossible to get at or repair."_
> — **Douglas Adams**, Mostly Harmless (1992)

→ Perfectly captures the essence của replication challenges: Khi bạn think hệ thống "cannot possibly go wrong" — split brain, data loss, cascading failures — đó thường là lúc khó fix nhất. Chương 6 cung cấp framework để **think about what can go wrong** và cách design systems tốt hơn.
