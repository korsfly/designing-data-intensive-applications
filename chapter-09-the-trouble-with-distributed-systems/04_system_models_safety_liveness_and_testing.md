# System Models, Safety/Liveness, và Testing

---

## 1. System Models — Formalizing Assumptions

### Tại sao cần System Models?

```
Distributed algorithms CANNOT solve problems in TOTAL GENERALITY
   (phải assume CÁC ĐẶC ĐIỂM nhất định về môi trường)

System model: FORMALIZE assumptions về behavior của nodes và network

→ Algorithm can be PROVED correct within certain system model
→ Reliable behavior achievable EVEN IF underlying model provides few guarantees
```

### 3 System Models cho Nodes

| Model              | Định nghĩa                                                                                                                                 | Ví dụ                     |
| ------------------ | ------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------- |
| **Crash-stop**     | Nodes chỉ fail theo 1 cách: STOPPING (crash). Sau khi crash: NEVER COMES BACK                                                              | Simplest but unrealistic  |
| **Crash-recovery** | Nodes có thể crash tại BẤT KỲ THỜI ĐIỂM nào và RESTART SAU ĐÓ. State stored in stable storage (durable across crash); in-memory state LOST | Most common in practice   |
| **Byzantine**      | Nodes có thể làm BẤT CỨ ĐIỀU GÌ (kể cả crash, send wrong messages, lie, corrupt data)                                                      | Security-critical systems |

```
Most server-side distributed systems: CRASH-RECOVERY MODEL
   Assume non-byzantine (honest) nodes
   → Much simpler algorithms sufficient
```

### 2 System Models cho Timing/Network

| Model            | Định nghĩa                                                                                                 |
| ---------------- | ---------------------------------------------------------------------------------------------------------- |
| **Synchronous**  | Bounded network delay, bounded process pauses, bounded clock error. NOT a realistic model for most systems |
| **Asynchronous** | No timing assumptions allowed. NO TIMEOUTS. Very restrictive but algorithms extremely robust               |

```
PARTIAL SYNCHRONY (most practical algorithms assume this):
   System usually behaves synchronously
   BUT: occasionally exceeds bounds on network delay, pauses, clock error
   → Must handle cases when timing assumptions violated
```

---

## 2. Safety và Liveness — Hai Loại Properties

### Định nghĩa

| Property                                                     | Định nghĩa                                                                                          |
| ------------------------------------------------------------ | --------------------------------------------------------------------------------------------------- |
| **Safety** (informal: "nothing bad happens")                 | If safety property violated at some point: CANNOT BE UNDONE. Violation is a definite moment in time |
| **Liveness** (informal: "something good eventually happens") | May not hold at some point but hope it will hold eventually                                         |

### Ví dụ

```
SAFETY properties:
   "At any given time, there is at most 1 leader" (single-leader replication)
   "No two transactions read the same object and write to it" (serializability)
   "If node crashes, all committed data is preserved" (durability)
   → If violated → cannot "un-violate" it

LIVENESS properties:
   "Every request eventually receives a response" (availability)
   "If node fails, eventually SOME other node takes over as leader" (failover)
   → "Eventually" is vague but OK for liveness
```

### Tại sao phân biệt quan trọng?

```
Algorithm thường phải trade off liveness for safety:
   Must tolerate failures → may have to give up on liveness guarantees

Ví dụ: Quorum-based voting
   SAFETY: Never return inconsistent result (majority must agree)
   LIVENESS: May not be able to process requests if majority of nodes fail
   → Sacrifice liveness (availability) for safety (correctness)

CAP theorem (Chapter 10): extreme case of this trade-off
```

---

## 3. Formal Verification — Model Checking

### TLA+ và Temporal Logic

```
Hệ thống phức tạp:
   Concurrency, message passing, network delays, crashes
   → Complex interactions → subtle bugs

FORMAL VERIFICATION = mathematical proof of correctness:
   TLA+ (Temporal Logic of Actions): specification language
   AWS, Microsoft, Azure, Intel: use TLA+ for critical systems

   Examples: Paxos, Raft, Amazon S3, Azure Cosmos DB specified in TLA+

TLC MODEL CHECKER: exhaustively checks all reachable states
   → Finds bugs that testing might miss
   → But ONLY for simplified models (state space explosion problem)

→ POWERFUL but not always practical for real production code
```

---

## 4. Fault Injection Testing — Jepsen

### Tại sao cần Fault Injection?

```
Unit tests và integration tests: happy path
Distributed system bugs: ONLY appear during failures
   (network partition, clock skew, node crashes)

→ Must deliberately trigger failures in test environment
```

### Jepsen Framework

```
JEPSEN: framework for running fault injection tests on distributed systems
   Written by Kyle Kingsbury

Các tools Jepsen cung cấp:
   Integrations for various operating systems
   Many prebuilt fault injectors

Jepsen HAS BEEN remarkably effective at finding critical bugs:
   Many widely-used systems: Cassandra, Elasticsearch, Redis, MongoDB, etc.
   Found serious consistency and data loss bugs in production systems
   → Major wake-up call for distributed systems community
```

### Cách hoạt động Fault Injection

```
1. Start distributed system on several nodes
2. Run client workload (read/write operations)
3. INJECT FAULTS (network partitions, clock skews, node crashes)
4. Verify: invariants still hold? (no data loss, no inconsistency)
5. Check: system recovers correctly after fault healed?

Fault types:
   Network partition (block traffic between nodes)
   Clock skew injection
   Node crash and restart
   Slow network (add artificial delays)
   Disk I/O errors
```

---

## 5. Deterministic Simulation Testing (DST)

### Motivation

```
Fault injection tests: CUMBERSOME to write (myriad tools required)
Model checking: LIMITED (state space explosion for real code)

→ Need approach that:
   Tests ACTUAL CODE (not a model)
   Explores MANY situations automatically
   Is REPRODUCIBLE when failure found

→ DETERMINISTIC SIMULATION TESTING (DST)
```

### Cách hoạt động DST

```
Simulator runs through LARGE NUMBER of randomized executions:
   Network communication: MOCKED
   I/O: MOCKED
   Clock timing: MOCKED
   → Simulator controls EXACT ORDER of everything (including failures)

→ Can explore FAR MORE situations than:
   Handwritten tests
   Fault injection (which doesn't have fine-grained control)

REPLAYABILITY: If test fails → can be RERUN (simulator knows exact order)
   (contrast: fault injection = nondeterministic, harder to reproduce)
```

### 3 Strategies để Make Code Deterministic for DST

#### 1. Application-Level

```
Systems built from ground up for determinism:

FOUNDATIONDB: Pioneer in DST space
   Built using Flow (asynchronous communication library)
   → Provides insertion point for deterministic network simulation

TIGERBEETLE (OLTP database): First-class DST support
   State modeled as STATE MACHINE
   All mutations: single event loop
   Combined with mock deterministic primitives (clocks)
   → Can run deterministically
```

#### 2. Runtime-Level

```
Languages with async runtimes: insertion point for determinism

FROSTDB: patches Go's runtime to execute goroutines SEQUENTIALLY

RUST MADSIM library:
   Deterministic implementations of Tokio's async runtime API
   Amazon S3 library, Kafka's Rust library, others
   → Apps swap in deterministic libraries WITHOUT changing code
```

#### 3. Machine-Level

```
Entire machine made deterministic:
   Delicate process: ALL normally nondeterministic calls → deterministic responses
   Clocks, network, storage → all must be accounted for

ANTITHESIS: builds custom HYPERVISOR
   Replaces nondeterministic operations with deterministic ones
   → Run entire distributed system in containers within hypervisor
   → Completely deterministic distributed system

Additional benefit: Antithesis BRANCHES test executions
   When less common behavior discovered → explore multiple subexecutions
   → Explore many more code paths

DST with mocked clocks: tests can run FASTER than wall clock time
   (simulate network latency and timeouts without actually waiting)
```

---

## 6. The Power of Determinism

```
Nondeterminism = CORE of all distributed systems challenges:
   Concurrency, network delay, process pauses, clock jumps, crashes
   → All happen in UNPREDICTABLE WAYS

Conversely: IF SYSTEM CAN BE MADE DETERMINISTIC → hugely simplifies things

Determinism appears throughout the book:
   EVENT SOURCING (Chapter 3): deterministically REPLAY log of events
                                → reconstruct derived materialized views
   WORKFLOW ENGINES (Chapter 5): workflow definitions must be DETERMINISTIC
                                  → durable execution semantics
   STATE MACHINE REPLICATION (Chapter 10): same deterministic transactions
                                           on each replica
   STATEMENT-BASED REPLICATION (Chapter 6): same SQL on each replica
   SERIAL TRANSACTION EXECUTION WITH STORED PROCEDURES (Chapter 8)

Making code fully deterministic requires CARE:
   Even after removing concurrency, I/O, network, clocks, random:
   Nondeterminism may remain!

   Examples:
   - Hash table iteration order: NONDETERMINISTIC in some languages
   - Resource limits (memory allocation failure, stack overflow): NONDETERMINISTIC
   → Must handle these carefully in deterministic simulation
```

---

## Tóm tắt phần này

```
SYSTEM MODELS:
   Nodes: Crash-stop | Crash-recovery (most practical) | Byzantine
   Timing: Synchronous | Asynchronous | Partial synchrony (most practical)

SAFETY vs LIVENESS:
   Safety: "nothing bad happens" → cannot un-violate; hard constraint
   Liveness: "something good eventually happens" → vague, eventual
   Trade-off: sacrifice liveness (availability) for safety (correctness)

FORMAL VERIFICATION (TLA+):
   Mathematical proof of correctness
   AWS, Microsoft: use for critical systems
   Practical for specifications, not always for actual code

FAULT INJECTION (Jepsen):
   Deliberately trigger failures in test environment
   Jepsen: remarkably effective, found critical bugs in many systems

DETERMINISTIC SIMULATION TESTING (DST):
   Mock network/I/O/clocks → control exact order of events
   Reproducible, faster than wall clock time
   Strategies: Application-level (FoundationDB, TigerBeetle),
               Runtime-level (FrostDB, MadSim),
               Machine-level (Antithesis hypervisor)

DETERMINISM: simple but powerful idea, appears throughout book
   Event sourcing, workflow engines, state machine replication, stored procs
   Must handle remaining nondeterminism: hash tables, resource limits
```
