# DDIA 2nd Edition — Chapter 9: The Trouble with Distributed Systems

> **Sách:** Designing Data-Intensive Applications, 2nd Edition  
> **Tác giả:** Martin Kleppmann & Chris Riccomini  
> **Chương:** 9 — The Trouble with Distributed Systems

---

## Mục tiêu chương

Chương 9 là chương **"đen tối nhất"** trong sách — nó liệt kê một cách có hệ thống tất cả những thứ có thể sai trong distributed systems. Hiểu được những vấn đề này là **điều kiện tiên quyết** để thiết kế fault-tolerant distributed systems (Chapter 10).

> Epigraph:
> _"They're funny things, Accidents. You never have them till you're having them."_
> — A.A. Milne, The House at Pooh Corner (1928)

---

## Tư duy cốt lõi của chương

```
Single computer: deterministic, binary (works or doesn't work)
   Same operation always produces same result
   Hardware fault → total system failure (crash, BSOD)

Distributed system: FUNDAMENTALLY DIFFERENT
   PARTIAL FAILURES: some parts broken, others fine
   NONDETERMINISTIC: unpredictable timing, intermittent failures
   "Distributed systems are hard to work with" — understatement
```

---

## Cấu trúc tài liệu này

| File                              | Nội dung                                                                |
| --------------------------------- | ----------------------------------------------------------------------- |
| `01_Unreliable_Networks.md`       | Packet loss, TCP limitations, network faults, fault detection, timeouts |
| `02_Unreliable_Clocks.md`         | Time-of-day vs monotonic clocks, NTP, clock skew, LWW dangers, TrueTime |
| `03_Process_Pauses_and_GC.md`     | GC pauses, VM suspend/resume, SIGSTOP, response time guarantees, RTOS   |
| `04_Knowledge_Truth_Lies.md`      | The majority rules, distributed locks, fencing tokens, zombie processes |
| `05_System_Models_and_Testing.md` | System models, safety vs liveness, formal verification, Jepsen, DST     |
| `06_Summary_and_Key_Concepts.md`  | Tổng hợp toàn chương, glossary, mindmap, câu hỏi ôn tập                 |

---

## Big Picture — 3 Nguồn gốc của Nondeterminism

```
┌──────────────────────────────────────────────────────────┐
│        TROUBLE WITH DISTRIBUTED SYSTEMS                   │
│                                                            │
│  1. UNRELIABLE NETWORKS                                   │
│     Packets lost, delayed, reordered, duplicated         │
│     Timeouts: can't distinguish node crash vs network fail│
│                                                            │
│  2. UNRELIABLE CLOCKS                                     │
│     Time-of-day clocks: jump forward/backward (NTP sync) │
│     Monotonic clocks: only useful for elapsed time        │
│     Clock skew: LWW with real-time clocks = data loss     │
│     Process pauses: thread paused arbitrarily             │
│                                                            │
│  3. PARTIAL FAILURES                                      │
│     Some nodes fail, others fine                          │
│     Nodes can't know state of other nodes for sure        │
│     Quorums needed for collective decisions               │
└──────────────────────────────────────────────────────────┘
```

---

## Kết luận chính của chương

```
"Suspicion, pessimism, and paranoia pay off in distributed systems"

Không thể avoid partial failures → Must build TOLERANCE into software
Distributed system: can run forever without service interruption
   (all faults and maintenance handled at node level)
   → POWER of distributed systems

Most non-safety-critical systems: cheap+unreliable >> expensive+reliable
→ "Keep things on single machine if possible"
   (Chapter 2: Scalability is not the only reason for distributed systems)
```

---

## Các chương liên quan

- **Chapter 2:** Fault tolerance, reliability, cascading failures
- **Chapter 6:** Replication — follower failure, leader failover risks
- **Chapter 7:** Sharding — shard assignment coordination
- **Chapter 8:** Transactions — 2PC coordinator failure, distributed locks
- **Chapter 10:** Consensus — how to build reliable systems despite these problems
- **Chapter 12:** Stream processing — exactly-once semantics
