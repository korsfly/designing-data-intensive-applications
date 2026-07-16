# Summary & Key Concepts — Chapter 9

---

## 1. Tổng hợp toàn chương

Chương 9 là chương **"bi quan nhất"** trong sách — nó liệt kê có hệ thống tất cả những gì có thể sai trong distributed systems. Đây là **nền tảng bắt buộc** để hiểu tại sao các giải pháp ở Chapter 10 (Consensus) lại phức tạp như vậy.

### 3 Nguồn gốc Nondeterminism

```
1. UNRELIABLE NETWORKS:
   Packets: lost, delayed, reordered, duplicated
   Sender cannot distinguish: request lost / node down / response lost
   Timeouts: only option, but ambiguous

2. UNRELIABLE CLOCKS:
   Time-of-day: can jump backward (NTP sync, leap seconds)
   Clock drift: 200 PPM → 17 seconds/day without sync
   Process pauses: thread suspended arbitrarily (GC, VM, OS, disk, SIGSTOP)
   LWW with real-time clocks: SILENT DATA LOSS

3. PARTIAL FAILURES:
   Some nodes fail, others fine (characteristic of distributed systems)
   Node cannot know state of other nodes with certainty
   Cannot distinguish node crash from network failure via timeout
```

### Kết luận chính

```
"Suspicion, pessimism, and paranoia pay off in distributed systems"

Must build FAULT TOLERANCE into software (cannot avoid partial failures)
First step: DETECT faults → hard (timeouts can't distinguish crash vs slow)
Making system tolerate faults: even harder (no shared memory, no shared state)
Major decisions: require QUORUM (not single-node)

BUT: distributed systems CAN run forever without service interruption
     All faults and maintenance handled at NODE level
     → Only bad if bad config change rolled out to ALL nodes simultaneously

Single machine: often a reasonable choice if workload fits
   → Don't open Pandora's box unnecessarily
```

---

## 2. Bảng thuật ngữ (Glossary)

### Networks

| Thuật ngữ                        | Định nghĩa                                                                                 |
| -------------------------------- | ------------------------------------------------------------------------------------------ |
| **Shared-nothing system**        | Machines connected by network; each has own CPU/memory/disk                                |
| **Asynchronous packet network**  | No guarantee when/whether/in order packets arrive (internet, ethernet)                     |
| **Bounded delay**                | Network guarantee: packet arrives within time d or is lost (synchronous network)           |
| **Unbounded delay**              | Asynchronous network: no timing guarantee                                                  |
| **Queueing delay**               | Variable delay due to: switch queue, OS receive queue, VM queue, TCP backpressure          |
| **Network partition**            | Some nodes can communicate, others cannot                                                  |
| **Asymmetric fault**             | A→B works but B→A doesn't (or only in one direction)                                       |
| **Limping node**                 | Node responding but too slow to be useful                                                  |
| **Phi Accrual failure detector** | Adaptive failure detection (Akka, Cassandra): auto-adjust timeout based on observed delays |
| **Synchronous network**          | Circuit-switched: fixed bandwidth reserved, bounded delay (telephone)                      |
| **STONITH**                      | Shoot The Other Node In The Head: forcibly disconnect/shutdown suspected zombie            |

### Clocks

| Thuật ngữ                         | Định nghĩa                                                                                  |
| --------------------------------- | ------------------------------------------------------------------------------------------- |
| **Time-of-day clock**             | Wall clock: current date/time; can jump backward; NOT for elapsed time                      |
| **Monotonic clock**               | Always moves forward; good for timeouts/intervals; NOT for cross-node ordering              |
| **NTP** (Network Time Protocol)   | Synchronize clocks over network; limited by network delay (~35ms via internet)              |
| **Clock drift**                   | Clock diverges over time from true time; 200 PPM typical                                    |
| **Clock skew**                    | Difference in readings between two clocks at the same point in time                         |
| **Leap second**                   | Extra second added/removed from minute; causes crashes in many systems (disabled from 2035) |
| **Steal time**                    | CPU time spent in other VMs (hypervisor context switching)                                  |
| **Physical clock**                | Measures actual elapsed time (time-of-day, monotonic)                                       |
| **Logical clock**                 | Measures relative ordering only (Lamport clocks, vector clocks)                             |
| **TrueTime API**                  | Google Spanner: returns confidence interval [earliest, latest] for current time             |
| **Confidence interval**           | Range of possible values for current time given clock uncertainty                           |
| **PTP** (Precision Time Protocol) | Sub-microsecond clock sync (MiFID II finance regulation)                                    |
| **GPS receiver / Atomic clock**   | High-accuracy time sources deployed per datacenter (Spanner)                                |
| **Spanner wait**                  | Deliberately wait for confidence interval before committing → causal ordering               |

### Process Pauses

| Thuật ngữ                        | Định nghĩa                                                                   |
| -------------------------------- | ---------------------------------------------------------------------------- |
| **Process pause**                | Thread execution suspended arbitrarily at any code point                     |
| **GC pause**                     | Stop-the-world garbage collection; historically minutes, now usually ms      |
| **VM suspend/resume**            | VM saved to disk and resumed later (live migration between hosts)            |
| **Thrashing**                    | OS spends most time swapping pages, little actual work done                  |
| **SIGSTOP / SIGCONT**            | Unix signals: pause and resume process                                       |
| **Hard real-time system**        | Must respond within specified deadline; failure = system failure             |
| **RTOS**                         | Real-Time Operating System: guaranteed CPU allocation in specified intervals |
| **Soft real-time**               | Best effort for low latency; missing deadline not catastrophic               |
| **GC slewing**                   | NTP adjusts rate (speed) of monotonic clock; no jumps; max ±0.05%            |
| **ZGC, Shenandoah, G1**          | Low-pause Java garbage collectors                                            |
| **Automatic reference counting** | Swift memory management: no GC pauses                                        |
| **Off-heap allocation**          | Store data outside JVM heap to avoid GC pressure                             |

### Knowledge, Truth, Locks

| Thuật ngữ                     | Định nghĩa                                                                                          |
| ----------------------------- | --------------------------------------------------------------------------------------------------- |
| **Zombie**                    | Former leaseholder acting as if still valid (doesn't know it lost the lease)                        |
| **Fencing**                   | Mechanism to prevent zombie from causing damage                                                     |
| **Fencing token**             | Monotonically increasing number from lock service; included in writes; storage rejects stale tokens |
| **Sequencer**                 | Google Chubby's name for fencing token                                                              |
| **Epoch number**              | Kafka's name for fencing token                                                                      |
| **Ballot number**             | Paxos's name for fencing token                                                                      |
| **Term number**               | Raft's name for fencing token                                                                       |
| **Lease**                     | Lock with timeout; released automatically if holder doesn't renew                                   |
| **Quorum**                    | Minimum number of votes from several nodes for a decision                                           |
| **Byzantine fault**           | Node sending incorrect/malicious messages (not just crash)                                          |
| **Byzantine fault tolerance** | Tolerates up to 1/3 of nodes being malicious                                                        |
| **Conditional write**         | Write succeeds only if object hasn't been modified since last read (S3, Azure Blob, GCS)            |
| **LWW** (Last Write Wins)     | Keep value with highest timestamp; dangerous with real-time clocks                                  |

### System Models and Testing

| Thuật ngữ                                  | Định nghĩa                                                                     |
| ------------------------------------------ | ------------------------------------------------------------------------------ |
| **System model**                           | Formalized assumptions about node/network behavior                             |
| **Crash-stop**                             | Nodes only fail by stopping, never restart                                     |
| **Crash-recovery**                         | Nodes can crash and restart; stable storage survives crash                     |
| **Byzantine**                              | Nodes can do anything including lying                                          |
| **Synchronous model**                      | Bounded network delay, process pauses, clock error                             |
| **Asynchronous model**                     | No timing assumptions; no timeouts                                             |
| **Partial synchrony**                      | Usually synchronous; occasionally exceeds bounds                               |
| **Safety property**                        | "Nothing bad happens"; violation is permanent; cannot be undone                |
| **Liveness property**                      | "Something good eventually happens"; can be delayed                            |
| **Formal verification**                    | Mathematical proof of algorithm correctness (TLA+)                             |
| **TLA+**                                   | Temporal Logic of Actions: specification language for distributed algorithms   |
| **TLC model checker**                      | Exhaustively checks all reachable states for TLA+ specifications               |
| **Fault injection**                        | Deliberately trigger failures in test environment                              |
| **Jepsen**                                 | Framework for fault injection tests on distributed databases                   |
| **Deterministic Simulation Testing (DST)** | Mock network/I/O/clocks → control exact order of operations                    |
| **FoundationDB Flow**                      | Async library enabling DST; pioneer of DST approach                            |
| **TigerBeetle**                            | OLTP database with first-class DST support                                     |
| **Antithesis**                             | Custom hypervisor for machine-level deterministic testing                      |
| **MadSim**                                 | Rust library: deterministic implementations of async runtime APIs              |
| **State space explosion**                  | Too many reachable states to enumerate (problem with model checking real code) |

---

## 3. Mindmap khái niệm

```
                  THE TROUBLE WITH DISTRIBUTED SYSTEMS
                                │
          ┌─────────────────────┼────────────────────────┐
          │                     │                        │
   UNRELIABLE              UNRELIABLE              KNOWLEDGE,
   NETWORKS                CLOCKS                 TRUTH & LIES
          │                     │                        │
   ┌──────┴──────┐        ┌──────┴──────┐         ┌──────┴──────┐
   │             │        │             │         │             │
 8 outcomes   Timeout   Time-of-  Monotonic   Quorum      Fencing
 (can't       cannot    day clock  clock       voting      tokens
  tell apart) distinguish      │        │         │          │
     │        crash vs    Jump    Only for  Node cannot Sequencer
     │        slow/net   backwd  intervals  trust self  Epoch/Term/
  TCP:                  (NTP,   NOT for    → majority   Ballot
  only OS ack          leaps)  cross-node  rules         │
  no app ack           │       ordering   │          Zombie
     │              Drift     │         Lease/Lock   problem
  Queueing:         200PPM  Logical    Distrib.
  switch/OS/VM     NTP:35ms  clocks    Lock bugs
  TCP backpress    needed    (Ch.10)  (Fig.9-4,5)
     │
  Variable delays     Process Pauses
  not law of nature → 8 causes: GC,VM,
  static partition   suspend,SIGSTOP,
  = bounded delay     disk,paging,...
  BUT expensive
          │
   System Models          Testing
   & Safety/Liveness      │
          │          ┌────┴─────┐
   Crash-stop    Jepsen      DST
   Crash-recovery (fault   (deterministic
   Byzantine     injection) simulation)
          │         │          │
   Safety:    Found bugs  FoundationDB
   permanent  many DBs    TigerBeetle
   Liveness:             Antithesis
   eventual              MadSim

   Trade-off: sacrifice liveness for safety
```

---

## 4. Câu hỏi tự kiểm tra (Self-Check Questions)

1. Liệt kê **8 điều có thể xảy ra** khi gửi request trong asynchronous network. Tại sao sender KHÔNG THỂ phân biệt chúng?

2. Giải thích **TCP "reliability" KHÔNG có nghĩa là** gì trong context của distributed systems. Khi nào TCP ack có thể misleading?

3. Tại sao **không có giá trị "đúng" cho timeout**? Liệt kê nguy hiểm của timeout quá ngắn và quá dài.

4. Giải thích **nguồn gốc của variable network delays** (queueing ở 4 cấp độ). Tại sao đây không phải "law of nature"?

5. Phân biệt **time-of-day clock** và **monotonic clock**. Mỗi loại dùng cho mục đích gì? Cái nào an toàn cho đo elapsed time?

6. Tại sao **LWW với real-time clocks = silent data loss**? Giải thích với ví dụ từ Figure 9-3.

7. Giải thích **TrueTime API** của Spanner. Tại sao Spanner phải "wait" trước khi commit?

8. Liệt kê **8 nguyên nhân process pauses**. Tại sao thread không biết nó đã bị paused?

9. Giải thích **tình huống zombie leaseholder** (Figure 9-4 và 9-5). Tại sao STONITH không đủ?

10. Mô tả **fencing token mechanism** (Figure 9-6). Tại sao nó robust hơn STONITH?

11. Phân biệt **fencing tokens** và **optimistic concurrency control** (SSI từ Chapter 8).

12. Giải thích **Byzantine faults** vs crash-recovery faults. Khi nào cần Byzantine fault tolerance?

13. Mô tả **3 system models cho nodes** (crash-stop, crash-recovery, Byzantine). Model nào thực tế nhất cho server-side systems?

14. Phân biệt **safety** và **liveness** properties. Cho ví dụ cho mỗi loại trong context replication.

15. Giải thích **Deterministic Simulation Testing (DST)**. So sánh với fault injection (Jepsen). Tại sao DST có thể chạy nhanh hơn wall clock time?

16. Mô tả **3 strategies** để làm code deterministic cho DST (application-level, runtime-level, machine-level). Cho ví dụ cho mỗi loại.

17. Tại sao **determinism** là "simple but powerful idea" xuyên suốt toàn bộ sách?

18. Tại sao bài học cuối cùng của chương là **"keep things on single machine if possible"**?

---

## 5. Liên kết tới các chương khác

| Chapter 9 đề cập                        | Mở rộng ở                                  |
| --------------------------------------- | ------------------------------------------ |
| Fault tolerance, cascading failures     | **Chapter 2** — Reliability                |
| LWW conflict resolution                 | **Chapter 6** — Multi-Leader Replication   |
| Follower failure, leader failover risks | **Chapter 6** — Single-Leader Replication  |
| 2PC coordinator failure                 | **Chapter 8** — Distributed Transactions   |
| Logical clocks, ID generators           | **Chapter 10** — Consistency and Consensus |
| Consensus algorithms (Paxos, Raft)      | **Chapter 10** — Consensus                 |
| Fencing tokens in lock services         | **Chapter 10** — Distributed Locks         |
| State machine replication               | **Chapter 10** — Using Shared Logs         |
| Event sourcing (determinism)            | **Chapter 3** — Event Sourcing             |
| Workflow determinism                    | **Chapter 5** — Durable Execution          |
| Statement-based replication             | **Chapter 6** — Replication Logs           |
| Exactly-once semantics                  | **Chapter 12** — Stream Processing         |

---

## 6. Trích dẫn mở đầu chương

> _"They're funny things, Accidents. You never have them till you're having them."_
> — **A.A. Milne**, The House at Pooh Corner (1928)

→ Một cách nói duyên dáng về bản chất của **partial failures**: bạn không thể biết khi nào failure xảy ra cho đến khi nó đang xảy ra — và khi đó thường quá muộn để chuẩn bị. Đây là lý do tại sao phải **thiết kế cho failure từ đầu**, không phải sau khi failure đã xảy ra.

---

## 7. Tóm tắt "Bài học cốt lõi" của Chương 9

```
Distributed system engineer's mindset:
╔═══════════════════════════════════════════════════════════╗
║  "Suspicion, pessimism, and paranoia pay off."            ║
║                                                           ║
║  Assume:                                                  ║
║  - Networks WILL lose/delay packets                       ║
║  - Clocks WILL drift and jump                             ║
║  - Processes WILL pause unexpectedly                      ║
║  - Nodes WILL fail at the worst possible time             ║
║                                                           ║
║  Design for:                                              ║
║  - Timeouts (but set them carefully)                      ║
║  - Quorum decisions (not single-node authority)           ║
║  - Fencing tokens (prevent zombie damage)                 ║
║  - Testing with fault injection + DST                     ║
║                                                           ║
║  Prefer:                                                  ║
║  - Single machine if workload fits                        ║
║  - Logical clocks over real-time clocks for ordering      ║
║  - Deterministic code for testability                     ║
╚═══════════════════════════════════════════════════════════╝
```
