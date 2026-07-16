# Summary & Key Concepts — Chapter 10

---

## 1. Tổng hợp toàn chương

Chương 10 là câu trả lời cho chương 9: **làm sao xây dựng hệ thống đáng tin cậy dù mọi thứ xung quanh đều không đáng tin cậy?**

### Luồng kiến thức

```
PROBLEM (Chapter 9):
   Unreliable networks, clocks, process pauses
   → Partial failures, nondeterminism
        │
        ▼
STRONGEST GUARANTEE: LINEARIZABILITY
   Appears as single copy, atomically
   Cost: slow + unavailable during network partition (CAP)
   When needed: leader election, uniqueness, cross-channel timing
        │
        ▼
WEAKER BUT USEFUL: CAUSAL ORDERING
   Lamport Clocks: causal consistency, total order
   HLC: causal + close to physical time
   Total Order Broadcast ≡ Consensus ≡ Shared Log
        │
        ▼
CONSENSUS ALGORITHMS: Paxos, Raft, Zab, VR
   Two rounds voting: elect leader (epoch) + propose entry (quorum overlap)
   "Single-leader replication done right"
   Fault-tolerant: tolerates f failures with 2f+1 nodes
        │
        ▼
COORDINATION SERVICES: ZooKeeper, etcd, Consul
   Locks/leases + fencing + failure detection + notifications
   Applications: config, leader election, work allocation, service discovery
```

---

## 2. Bảng Consistency Models

| Model                      | Strength           | Guarantee                        | Examples                       |
| -------------------------- | ------------------ | -------------------------------- | ------------------------------ |
| **Linearizability**        | Strongest          | Single copy, real-time order     | Spanner, etcd (default reads)  |
| **Serializability**        | Strong             | Transactions serial              | PostgreSQL (SSI), SQL Server   |
| **Strict Serializability** | Strongest combined | Both linearizable + serializable | Spanner, FoundationDB          |
| **Snapshot Isolation**     | Moderate           | Consistent snapshot at start     | Most SQL DBs                   |
| **Read Committed**         | Weak               | No dirty reads/writes            | Default PostgreSQL, MySQL      |
| **Causal Consistency**     | Weak-moderate      | Respects happens-before          | Many eventually consistent DBs |
| **Eventual Consistency**   | Weakest            | Eventually converge              | Dynamo, Cassandra (default)    |

---

## 3. Bảng Equivalences

```
All problems EQUIVALENT to consensus (solution to one → all):

┌─────────────────────────────────────────────────────────┐
│                   CONSENSUS                              │
│              (agree on single value)                     │
│                                                          │
│  ≡ CAS REGISTER (linearizable compare-and-set)          │
│  ≡ SHARED LOG (total order broadcast)                    │
│  ≡ ATOMIC COMMIT (fault-tolerant 2PC)                   │
│  ≡ LOCKS / LEASES (distributed mutual exclusion)         │
│  ≡ UNIQUENESS CONSTRAINTS (first writer wins)            │
│  ≡ FETCH-AND-ADD (linearizable counter increment)        │
└─────────────────────────────────────────────────────────┘
```

---

## 4. Bảng thuật ngữ (Glossary)

### Linearizability

| Thuật ngữ                   | Định nghĩa                                                                  |
| --------------------------- | --------------------------------------------------------------------------- |
| **Linearizability**         | Strongest consistency: appears single copy, all ops atomic, real-time order |
| **Atomic consistency**      | Another name for linearizability                                            |
| **Strict serializability**  | Serializability + Linearizability combined                                  |
| **Recency guarantee**       | Once write confirmed, all subsequent reads see it                           |
| **CAS (Compare-and-Set)**   | Atomic: read + conditional write; only one concurrent CAS wins              |
| **CAP theorem**             | Under partition: choose consistency (C) or availability (A)                 |
| **PACELC**                  | Partition: A or C; Else: L (latency) or C (consistency)                     |
| **Unclean leader election** | Allow stale replica to become leader (sacrifices linearizability)           |

### Ordering and Causality

| Thuật ngữ                      | Định nghĩa                                                                          |
| ------------------------------ | ----------------------------------------------------------------------------------- |
| **Causal order**               | Partial order based on happens-before relationships                                 |
| **Causal consistency**         | Weakest model preventing causality violations                                       |
| **Total order**                | Every pair of events comparable (one before the other)                              |
| **Lamport clock**              | (counter, node_id): causal consistency, total order; doesn't distinguish concurrent |
| **Hybrid Logical Clock (HLC)** | Logical + physical time: causally consistent, close to real time                    |
| **Total Order Broadcast**      | Reliable + totally ordered delivery to all nodes                                    |
| **Atomic broadcast**           | Another name for total order broadcast                                              |
| **FLP Impossibility**          | Cannot guarantee consensus termination in async system with even 1 crash            |
| **Partial synchrony**          | Usually synchronous, occasionally exceeds bounds                                    |
| **Snowflake ID**               | 64-bit: timestamp + datacenter + machine + sequence; NOT causally consistent        |
| **ULID / KSUID**               | Sortable, URL-safe distributed IDs with timestamp prefix                            |
| **Consensus number**           | Fetch-and-add: 2; CAS/shared log: ∞                                                 |

### Consensus Algorithms

| Thuật ngữ                        | Định nghĩa                                                               |
| -------------------------------- | ------------------------------------------------------------------------ |
| **Consensus**                    | Agreement: uniform agreement + integrity + validity + termination        |
| **Single-value consensus**       | Nodes agree on one value; once decided, never change                     |
| **Shared log**                   | Ordered sequence of entries all nodes agree on; ≡ consensus              |
| **State machine replication**    | Same deterministic ops in same order on each replica → same state        |
| **Paxos**                        | Classic consensus; complex; Multi-Paxos = shared log                     |
| **Multi-Paxos**                  | Practical extension of Paxos providing shared log                        |
| **Raft**                         | Understandable consensus algorithm; etcd, CockroachDB, TiDB              |
| **Zab**                          | ZooKeeper Atomic Broadcast; primary-backup model                         |
| **Viewstamped Replication (VR)** | Early consensus algorithm (1988); predecessor to Paxos                   |
| **Egalitarian Paxos (EPaxos)**   | Leaderless Paxos variant; robust against poor network connections        |
| **Epoch number**                 | Monotonically increasing; unique leader per epoch; higher epoch prevails |
| **Term number**                  | Raft's name for epoch number                                             |
| **Ballot number**                | Paxos's name for epoch number                                            |
| **View number**                  | VR's name for epoch number                                               |
| **Pre-vote phase**               | Raft extension: ask peers before starting real election                  |
| **Log matching invariant**       | Same (index, term) → same content at all prior entries (Raft)            |
| **Quorum overlap**               | Leader election quorum ∩ entry proposal quorum ≠ ∅                       |
| **Reconfiguration**              | Adding/removing voting nodes from consensus cluster                      |
| **Observer** (ZooKeeper)         | Receives log but doesn't vote; stale reads but high availability         |

### Coordination Services

| Thuật ngữ                | Định nghĩa                                                            |
| ------------------------ | --------------------------------------------------------------------- |
| **Coordination service** | ZooKeeper, etcd, Consul: coordinate distributed system nodes          |
| **Google Chubby**        | Google's lock service; model for coordination services                |
| **Ephemeral node**       | ZooKeeper: disappears when session ends; failure detection            |
| **zxid**                 | ZooKeeper's transaction ID; monotonically increasing fencing token    |
| **Watch**                | ZooKeeper: one-time notification when key changes                     |
| **Session**              | Long-lived connection between client and coordination service         |
| **Heartbeat**            | Periodic signal: "I'm still alive"; failure detection mechanism       |
| **Lease timeout**        | If no heartbeat within this duration → session considered dead        |
| **Apache Curator**       | Java library providing higher-level tools on top of ZooKeeper         |
| **Apache BookKeeper**    | Replicate fast-changing internal state of a service                   |
| **KRaft**                | Kafka's built-in Raft-based consensus (replaces ZooKeeper dependency) |
| **Service registry**     | Services register endpoints on startup; clients discover via registry |
| **TTL (Time-to-Live)**   | Cache expiry; service discovery cache refresh mechanism               |

---

## 5. Mindmap khái niệm

```
                    CONSISTENCY AND CONSENSUS
                             │
         ┌───────────────────┼──────────────────────┐
         │                   │                      │
  LINEARIZABILITY      ORDERING &             CONSENSUS
         │               CAUSALITY            ALGORITHMS
   ┌─────┴─────┐             │                      │
   │           │      ┌──────┴──────┐         ┌─────┴─────┐
Single   Replicated  Causal  Lamport  HLC    Paxos   Raft   Zab
 node:  algorithms:  Cons.   Clock         Multi    Pre-  Primary
trivial  Only SL    Partial   total  Close  Paxos   Vote  backup
         readable   order    order  phys.  shared   ext.    │
         may work!   │         │    time    log      │     VR
   │                 │         │      │              │
  CAS             FLP:       TOB ≡ Consensus     Epoch
  Unique         impossible  = Shared Log       numbers
 constraints    async+crash  ≡ CAS             Two votes
  Leader         Partial     ≡ Atomic          (election
  election      synchrony:  Commit             + entry)
                practical   ≡ Locks           Quorum
                            ≡ Uniqueness      overlap
   CAP Theorem
   Under partition:
   C or A
   PACELC: also
   L or C normally
         │
   COORDINATION
   SERVICES
         │
   ┌─────┴──────┐
ZooKeeper etcd Consul
     │
 4 features:
 Locks/Lease
 Fencing tokens
 Failure detect
 Watch/Notify
     │
 Applications:
 Config mgmt
 Leader election
 Work allocation
 Service discovery
 (overkill for this)
```

---

## 6. Câu hỏi tự kiểm tra (Self-Check Questions)

1. Định nghĩa **Linearizability**. Tại sao nó giống "variable in single-threaded program"? Phân biệt với Serializability.

2. Tại sao **single-leader replication reads from followers** KHÔNG linearizable? Khi nào single-leader reads from leader là linearizable?

3. Tại sao **leaderless quorum + LWW** vẫn KHÔNG linearizable? Liên kết với Chapter 9 (clock skew).

4. Giải thích **CAP theorem** và tại sao nó bị coi là "misleading and limited". Giải thích **PACELC** như một framework tốt hơn.

5. Khi nào cần **linearizability**? Liệt kê 4 use cases cụ thể.

6. Phân biệt **causal order (partial)** và **total order**. Tại sao causal consistency = weakest model preventing causality violations?

7. Giải thích **Lamport clock** algorithm. Tại sao không thể dùng Lamport timestamps để phân biệt concurrent vs causal relationships?

8. **HLC (Hybrid Logical Clock)** giải quyết vấn đề gì của Lamport clocks? Tại sao được dùng trong CockroachDB, TiDB?

9. Tại sao **Snowflake IDs, UUIDs** KHÔNG causally consistent? Khi nào cần linearizable ID generator?

10. Giải thích **Total Order Broadcast** (2 properties). Tại sao nó equivalent với consensus?

11. Liệt kê **6 problems equivalent to consensus**. Tại sao tất cả đều tương đương nhau?

12. Phân biệt **Consensus** (Raft, Paxos) và **2PC** (Chapter 8). Tại sao 2PC bị "stuck" nhưng consensus không?

13. Giải thích **two rounds of voting** trong consensus. Tại sao quorums PHẢI OVERLAP?

14. Tại sao **epoch numbers** ngăn split brain? Mô tả tình huống cụ thể.

15. Giải thích **Raft leader election** (RequestVote, log up-to-date check). Tại sao cần check log up-to-date?

16. Tại sao **pre-vote phase** cần thiết trong Raft? Vấn đề gì nó giải quyết?

17. So sánh **Paxos vs Raft vs EPaxos** (philosophy, use cases, network robustness).

18. Liệt kê **4 features** của coordination services. Cho ví dụ cụ thể về cách ZooKeeper/etcd implement mỗi feature.

19. Tại sao **service discovery** thường không cần coordination service (linearizability overkill)?

20. Tại sao coordination service dùng **small fixed cluster (3-5 nodes)** thay vì chạy consensus trên tất cả nodes?

---

## 7. Liên kết tới các chương khác

| Chapter 10 đề cập                   | Liên quan đến                                                      |
| ----------------------------------- | ------------------------------------------------------------------ |
| Linearizability vs. replication lag | **Chapter 6** — Replication, eventual consistency                  |
| Leader election, failover           | **Chapter 6** — Single-leader replication                          |
| 2PC vs. fault-tolerant consensus    | **Chapter 8** — Distributed Transactions                           |
| Fencing tokens (from Chapter 9)     | **Chapter 9** — Knowledge, Truth, Lies                             |
| Unreliable clocks (HLC motivation)  | **Chapter 9** — Unreliable Clocks                                  |
| State machine replication           | **Chapter 3** — Event Sourcing; **Chapter 12** — Stream Processing |
| Shared logs for stream processing   | **Chapter 12** — Stream Processing                                 |
| ZooKeeper for shard coordination    | **Chapter 7** — Sharding                                           |
| Atomic commit = fault-tolerant 2PC  | **Chapter 8** — Distributed Transactions                           |
| Consensus for serial transactions   | **Chapter 8** — Serial Execution                                   |

---

## 8. Trích dẫn mở đầu chương

> _"Is it better to be alive and wrong or right and dead?"_
> — **Jay Kreps**, conflating two famous statements (2013)

→ Captures perfectly the **availability vs. consistency trade-off** (CAP theorem). Is it better to:

- Be **alive and wrong** (available but serving stale data)?
- Or **right and dead** (consistent but unavailable during partition)?

Chapter 10 shows that with **consensus algorithms**, we can be mostly alive AND right — tolerating failures gracefully while maintaining strong consistency.

---

## 9. Bảng Consensus Algorithms So sánh

|                        | Paxos              | Multi-Paxos       | Raft                    | Zab           | VR            | EPaxos     |
| ---------------------- | ------------------ | ----------------- | ----------------------- | ------------- | ------------- | ---------- |
| **Type**               | Single-value       | Shared log        | Shared log              | Shared log    | Shared log    | Leaderless |
| **Leader**             | Optional (phase 1) | Yes (stable)      | Yes                     | Yes (primary) | Yes           | No         |
| **Understandability**  | Hard               | Hard              | Easy                    | Moderate      | Moderate      | Complex    |
| **Network robustness** | Good               | Good              | Pre-vote helps          | Good          | Good          | Best       |
| **Used by**            | Chubby (Google)    | DynamoDB, Spanner | etcd, CockroachDB, TiDB | ZooKeeper     | Farsite, PBFT | Research   |
| **Epoch/Term name**    | Ballot             | Ballot            | Term                    | Epoch         | View          | Ballot     |
