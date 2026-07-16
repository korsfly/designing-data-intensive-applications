# Linearizability

---

## 1. Định nghĩa: Strongest Consistency Guarantee

### Linearizability là gì?

```
LINEARIZABILITY: make replicated data appear as though there
   were ONLY ONE COPY, with all operations acting on it ATOMICALLY

Còn gọi: atomic consistency, strong consistency, immediate consistency,
          external consistency, strict serializability

Đặc điểm: as soon as 1 client successfully completes a write
   → ALL CLIENTS reading from database must see that written value

→ System behaves as if ONLY 1 COPY, even if physically replicated
→ Like a VARIABLE IN SINGLE-THREADED PROGRAM:
   After assigning x = 5, any subsequent read returns 5
```

### Recency Guarantee

```
LINEARIZABILITY ≠ SERIALIZABILITY:
   Serializability (Chapter 8): isolation property of transactions
      → Transactions behave as if executed in SOME SERIAL ORDER
      → Can be serial order DIFFERENT FROM ACTUAL ORDER

   Linearizability: recency guarantee for INDIVIDUAL OPERATIONS
      → Each operation appears instantaneous at a single point in time
      → Operations appear to execute in REAL-TIME ORDER
      → No time travel!

"Strict serializability" = Serializability + Linearizability
   → Most systems don't provide both simultaneously
```

### Register Operations

```
Single-object operations (read and write of a register):
   READ(x): return current value of x
   WRITE(x, v): set x to v, return ok

Linearizability of register:
   After a successful write of v, any subsequent read MUST return v
   (unless another write happens in between)

   "Subsequent" defined by REAL TIME:
   If write finishes at time t1 and read starts at time t2 > t1
   → Read MUST see v (or later value)
```

---

## 2. Linearizability vs. Nonlinearizability (Ví dụ)

### Figure 10-1: Nonlinearizable System

```
Timeline:
   Client B writes x = 0 → x = 1 (write in progress)
   Client A: read → 1 (sees new value)
   Client C: read → 0 (sees old value AFTER Client A's read!)

→ C reads AFTER A, but sees OLDER value than A
→ TIME TRAVEL: appeared to go backward in time
→ VIOLATES LINEARIZABILITY
```

### Figure 10-2: Linearizable System

```
Every read appears to be at a single point in linearized timeline:
   All reads BEFORE write completes: may return old or new value
   FIRST read to return new value: linearization point
   ALL SUBSEQUENT READS must return new value (or later)

Key property: Once any client reads the new value,
   ALL SUBSEQUENT reads from ANY client must also see new value
```

### Read-Write Register vs. CAS Register

```
Read-write register: linearizable reads/writes
CAS register (compare-and-set): ALSO linearizable
   → Atomically: read + conditional write in single operation
   → If current value = expected → set to new value, return ok
   → Otherwise → return error (no change)

CAS is MORE POWERFUL than read-write register for building higher-level constructs
```

---

## 3. Khi nào cần Linearizability?

### Use Cases

| Situation                             | Cần linearizability? | Lý do                                                                  |
| ------------------------------------- | -------------------- | ---------------------------------------------------------------------- |
| **Locking and leader election**       | ✓                    | Single leader: chỉ 1 node có lock; multi-node CAS must be linearizable |
| **Uniqueness constraints**            | ✓                    | Username registration: only 1 user gets given username                 |
| **Cross-channel timing dependencies** | ✓                    | Image upload + message via 2 channels: resizer must see uploaded file  |
| **Reporting and dashboards**          | Thường không         | Slight staleness OK                                                    |
| **Search indexes**                    | Thường không         | Lag of few seconds acceptable                                          |
| **Caching layers**                    | Thường không         | Cache doesn't need to match DB perfectly                               |

### Leader Election cần Linearizability

```
Single-leader replication: all nodes must AGREE on who is leader
   → If 2 nodes both believe they are leader: SPLIT BRAIN

Implementing leader election requires LINEARIZABLE LOCK/LEASE:
   At most 1 node can acquire the lock at any given time
   → Used by ZooKeeper, etcd, Consul for leader election
```

---

## 4. Triển khai Linearizability

### Single-node system

```
Single-node: TRIVIALLY linearizable (no replication)
   → All reads from same node, see same state
   → But: single point of failure, no fault tolerance
```

### Replication Algorithms vs. Linearizability

| Replication Type                         | Linearizable?  | Chi tiết                                                                     |
| ---------------------------------------- | -------------- | ---------------------------------------------------------------------------- |
| **Single-leader (reads from leader)**    | ✓ Potentially  | Reads from leader OK; but: failover can violate if new leader has stale data |
| **Single-leader (reads from followers)** | ✗ No           | Async replication: followers may be stale                                    |
| **Multi-leader**                         | ✗ No           | No single copy; concurrent writes on multiple leaders                        |
| **Leaderless (Dynamo-style)**            | ✗ Probably not | Even with quorum: "last write wins" with clock skew → not linearizable       |

```
SURPRISE: Leaderless replication + quorum ≠ linearizability!

Dynamo-style databases (Riak, Cassandra): trade linearizability for performance/availability
→ Provide STRONG EVENTUAL CONSISTENCY instead

Exceptions:
   Riak: strong consistency option (uses single-leader per shard)
   Cassandra: lightweight transactions (Paxos) for individual keys
```

---

## 5. Cost of Linearizability và CAP Theorem

### Network Partition Scenario

```
Multi-datacenter database (Figure 10-5):
   Network partition cuts link between datacenters

   Each datacenter still has local replicas
   Users in each datacenter: requests routed to local replicas

CHOICES:
   Option A: LINEARIZABLE
      → Any replica receiving write MUST forward to other datacenter
      → If link down: WAIT (block) or RETURN ERROR
      → Sacrifice availability

   Option B: AVAILABLE (accept writes locally)
      → Each datacenter continues operating independently
      → When link restored: RECONCILE CONFLICTS
      → Sacrifice linearizability (clients in diff DCs may see diff values)
```

### CAP Theorem

> **CAP Theorem** (Eric Brewer, 2000): A system can provide at most 2 of 3:
>
> - **C**onsistency (linearizability)
> - **A**vailability
> - **P**artition tolerance

```
REALITY: Network partitions WILL happen (Chapter 9)
   → Cannot choose "CP vs AP" in normal operation
   → Under partition: must choose between C and A

→ Better framing: "Consistent or Available WHEN PARTITIONED"
→ PAC theorem (Partitioned networks: pick A or C)

CAP theorem: popular but MISLEADING and LIMITED:
   Only considers one consistency model (linearizability)
   Only considers one kind of fault (network partition)
   Many other trade-offs exist (latency vs. consistency, etc.)

PACELC (alternative):
   If Partition: choose A (availability) or C (consistency)
   Else (normal operation): choose L (latency) or C (consistency)
```

### Latency Cost of Linearizability

```
EVEN WITHOUT network partitions: linearizability = SLOWER

Linearizable read: must contact quorum or leader to ensure latest value
   → Multiple network round-trips
   → Not possible to serve reads from cache alone

Nonlinearizable read: CAN be served from local replica/cache
   → Lower latency
   → May be stale

Spanner achieves linearizability across datacenters:
   Uses TrueTime (GPS/atomic clocks) to minimize wait times
   Accepts ~7ms commit wait → bounded clock uncertainty
```

### Nhiều Consistency Models hơn là Chỉ Linearizable/Not

```
Strong: LINEARIZABILITY (appears single copy, real-time order)
Strong: SERIALIZABILITY (transactions as if serial, some order)
Weaker: SNAPSHOT ISOLATION (each txn sees consistent snapshot)
Weaker: READ COMMITTED (no dirty reads)
Weak:   EVENTUAL CONSISTENCY (no guarantees except eventual convergence)
Weak:   MONOTONIC READS, READ-YOUR-WRITES (session guarantees)
```

---

## Tóm tắt phần này

```
LINEARIZABILITY:
   Strongest consistency: appears as single copy, atomically
   Recency guarantee: once write seen, all subsequent reads see it
   ≠ Serializability (different concepts, can combine → strict serializability)

IMPLEMENTATIONS:
   Single-leader (read from leader): potentially linearizable
   Multi-leader, leaderless: generally NOT linearizable
   Even leaderless quorum ≠ linearizability (clock skew, network)

COST:
   Under network partition: must choose linearizability OR availability (CAP)
   Even without partition: slower (must contact quorum/leader for reads)
   Spanner: linearizable across DCs using TrueTime (~7ms wait)

WHEN NEEDED:
   Leader election, distributed locks/leases
   Uniqueness constraints (usernames, seats)
   Cross-channel timing dependencies
```
