# DDIA 2nd Edition — Chapter 6: Replication

> **Sách:** Designing Data-Intensive Applications, 2nd Edition  
> **Tác giả:** Martin Kleppmann & Chris Riccomini  
> **Chương:** 6 — Replication

---

## Mục tiêu chương

**Replication** = giữ bản sao của cùng data trên nhiều máy kết nối qua network. Tuy khái niệm đơn giản, việc triển khai đúng đắn là **bài toán phức tạp** — đặc biệt khi data thay đổi liên tục.

> Epigraph mở đầu chương:
> _"The major difference between a thing that might go wrong and a thing that cannot possibly go wrong is that when a thing that cannot possibly go wrong goes wrong, it usually turns out to be impossible to get at or repair."_
> — Douglas Adams, Mostly Harmless (1992)

---

## 5 Lý do cần Replication

```
1. HIGH AVAILABILITY    — Tiếp tục hoạt động khi 1+ machine/zone/region down
2. DURABILITY           — Không mất data khi machine fail vĩnh viễn
3. DISCONNECTED OPERATION — Tiếp tục hoạt động dù network bị gián đoạn
4. LATENCY              — Đặt data gần user về địa lý
5. SCALABILITY          — Tăng read throughput bằng cách phân phối reads qua replicas
```

---

## Cấu trúc tài liệu này

| File                                | Nội dung                                                                         |
| ----------------------------------- | -------------------------------------------------------------------------------- |
| `01_Single_Leader_Replication.md`   | Leader-follower model, sync/async, setup follower, failover, replication logs    |
| `02_Replication_Lag_Problems.md`    | Eventual consistency, read-after-write, monotonic reads, consistent prefix reads |
| `03_Multi_Leader_Replication.md`    | Geo-distributed, topologies, sync engines, conflict resolution (LWW, CRDT, OT)   |
| `04_Leaderless_Replication.md`      | Dynamo-style, quorums (n/w/r), read repair, hinted handoff, anti-entropy         |
| `05_Concurrent_Writes_Detection.md` | Happens-before relation, version vectors, detecting/resolving concurrent writes  |
| `06_Summary_and_Key_Concepts.md`    | Tổng hợp toàn chương, glossary, mindmap, câu hỏi ôn tập                          |

---

## Big Picture — 3 Kiến trúc Replication

```
┌─────────────────────────────────────────────────────────┐
│              3 REPLICATION ARCHITECTURES                 │
│                                                           │
│  SINGLE-LEADER      MULTI-LEADER        LEADERLESS       │
│  (Active/Passive)   (Active/Active)     (Dynamo-style)   │
│                                                           │
│  Tất cả writes      Nhiều nodes         Bất kỳ replica   │
│  → 1 leader         accept writes       nào cũng         │
│                                         accept writes     │
│  Consistency:       Consistency:         Consistency:     │
│  Tốt nhất           Yếu hơn             Yếu nhất         │
│                                                           │
│  Fault tolerance:   Fault tolerance:    Fault tolerance:  │
│  Trung bình         Tốt hơn             Tốt nhất          │
└─────────────────────────────────────────────────────────┘
```

---

## Các chương liên quan

- **Chapter 2:** Fault tolerance, reliability — nền tảng
- **Chapter 5:** Data encoding (WAL, logical log đều cần encoding)
- **Chapter 7:** Sharding — kết hợp với replication
- **Chapter 8:** Transactions (ACID) — liên quan đến consistency guarantees
- **Chapter 9:** Network failures, unreliable clocks — vấn đề fundamental trong distributed systems
- **Chapter 10:** Consensus (leader election, linearizability)
- **Chapter 13:** Scalable conflict detection and resolution
