# DDIA 2nd Edition — Chapter 10: Consistency and Consensus

> **Sách:** Designing Data-Intensive Applications, 2nd Edition  
> **Tác giả:** Martin Kleppmann & Chris Riccomini  
> **Chương:** 10 — Consistency and Consensus

---

## Mục tiêu chương

Chương 9 trình bày **tất cả vấn đề** của distributed systems. Chương 10 trình bày **giải pháp**: làm sao xây dựng hệ thống đáng tin cậy dù bản thân network, clocks, và processes đều không đáng tin cậy.

> Epigraph:
> _"Is it better to be alive and wrong or right and dead?"_
> — Jay Kreps, conflating two Famous statements (2013)

---

## Cấu trúc tài liệu này

| File                             | Nội dung                                                                      |
| -------------------------------- | ----------------------------------------------------------------------------- |
| `01_Linearizability.md`          | Linearizability definition, recency guarantee, cost, CAP theorem              |
| `02_Ordering_and_Causality.md`   | Causality, Lamport clocks, hybrid logical clocks, total order broadcast       |
| `03_Linearizable_IDs_and_CAS.md` | ID generators, compare-and-set, uniqueness constraints                        |
| `04_Consensus_Algorithms.md`     | Paxos, Raft, Zab, epoch numbers, two rounds of voting, pros/cons              |
| `05_Coordination_Services.md`    | ZooKeeper, etcd, Consul: locks, fencing, failure detection, service discovery |
| `06_Summary_and_Key_Concepts.md` | Tổng hợp toàn chương, glossary, mindmap, câu hỏi ôn tập                       |

---

## Big Picture

```
┌──────────────────────────────────────────────────────────┐
│            CONSISTENCY AND CONSENSUS                      │
│                                                            │
│  LINEARIZABILITY: strongest consistency guarantee         │
│  "appears as single copy, ops happen atomically"         │
│  Cost: slow (network round-trips); CAP theorem            │
│                                                            │
│  ORDERING: causality → Lamport/HLC clocks                 │
│  Total Order Broadcast ≡ Consensus ≡ Shared Log           │
│                                                            │
│  CONSENSUS ALGORITHMS: Paxos, Raft, Zab, VR              │
│  "single-leader replication done right"                   │
│  Two rounds of voting: elect leader + append to log       │
│                                                            │
│  COORDINATION SERVICES: ZooKeeper, etcd, Consul           │
│  Locks/leases + fencing + failure detection + discovery   │
└──────────────────────────────────────────────────────────┘
```

---

## Các chương liên quan

- **Chapter 6:** Replication lag, eventual consistency (contrast with linearizability)
- **Chapter 7:** Sharding — coordination services for shard assignment
- **Chapter 8:** Transactions — atomic commit, 2PC (contrast with consensus)
- **Chapter 9:** Problems (unreliable networks/clocks/pauses) — solved by consensus
- **Chapter 12:** Stream processing — total order broadcast, state machine replication
