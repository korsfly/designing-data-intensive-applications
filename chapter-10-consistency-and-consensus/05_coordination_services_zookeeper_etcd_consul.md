# Coordination Services: ZooKeeper, etcd, Consul

---

## 1. Coordination Services là gì?

### Định nghĩa

```
COORDINATION SERVICES: specialized distributed systems designed to
   COORDINATE AMONG NODES of another distributed system

Ví dụ thực tế:
   Kubernetes → relies on etcd
   Spark, Flink (high availability mode) → rely on ZooKeeper
   HashiCorp products → Consul

KHÔNG phải general-purpose database:
   NOT designed for high write volumes
   NOT for large datasets (typically fit in memory)

THIẾT KẾ CHO: small amounts of coordination data
   Replicated across multiple nodes via fault-tolerant consensus
```

### Nguồn gốc

```
Coordination services modeled after Google's CHUBBY LOCK SERVICE
   (described in paper by Burrows, 2006)

Combine consensus algorithm with several additional features:
   Locks and leases
   Fencing token support
   Failure detection
   Change notifications
```

---

## 2. Bốn Tính năng Chính

### Feature 1: Locks và Leases

```
CONSENSUS systems implement ATOMIC, FAULT-TOLERANT CAS:

Locks/Leases in ZooKeeper/etcd:
   Multiple nodes try to acquire same lease → ONLY ONE succeeds

Implement: create an ephemeral node (ZooKeeper) or use lease API (etcd)
   If node crashes → lease automatically released (after timeout)

Use cases:
   Single-leader databases: only 1 node holds leader lock
   Task assignment: only 1 worker processes each task
   Exclusive resource access: only 1 client writes to partition

IMPORTANT: Holds lease → can obtain fencing token → write to storage safely
   (prevents zombie damage — Chapter 9)
```

### Feature 2: Fencing Token Support

```
Each LOG ENTRY in consensus log: monotonically increasing ID

ZooKeeper: zxid (transaction ID), cversion (node version)
etcd: revision number + lease ID
Consul: modify index

These IDs serve as FENCING TOKENS:
   Leaseholder includes token in every write to storage
   Storage service rejects writes with lower-or-equal token
   → Zombie former leaseholder fenced off

Combined: lease (from consensus) + fencing token (from log position)
   → Complete solution to zombie problem
```

### Feature 3: Failure Detection

```
HEARTBEAT mechanism:
   Clients maintain long-lived SESSION with coordination service
   Periodically exchange heartbeats to signal "still alive"

Connection temporarily interrupted: session remains valid
   (Leases remain active while session alive)

No heartbeat for longer than LEASE TIMEOUT:
   Coordination service assumes client is DEAD
   → Releases the lease
   → Other nodes can acquire it

ZooKeeper: "ephemeral nodes" = nodes that disappear when session ends
   Useful for "who is currently alive" tracking
```

### Feature 4: Change Notifications (Watch/Subscribe)

```
Client registers WATCH on a key:
   Coordination service sends NOTIFICATION whenever key changes

Use cases:
   New node joins cluster: writes its endpoint → others notified
   Node fails: session times out → ephemeral nodes disappear → others notified
   Config change: updated in coordination service → services notified immediately

ALTERNATIVE to polling:
   Without watches: clients poll repeatedly ("is leader still X?")
   With watches: immediate notification → more efficient, lower latency

ZooKeeper watches: one-time (must re-register after firing)
etcd watches: continuous stream of events
Consul watches: HTTP long-polling, HTTP streaming
```

---

## 3. Các Ứng dụng Phổ biến

### Application 1: Configuration Management

```
Store application configuration as key-value pairs:
   Timeouts, thread pool sizes, feature flags, connection strings,...

Process startup: load latest settings from coordination service
Subscribe to changes: receive notification when config changes

WHY use coordination service for config?
   Fault-tolerant (replicated): config doesn't disappear
   Consistent: all nodes see same config
   Change notification: no polling needed

ALTERNATIVE: polling file or URL periodically
   Simpler if you don't already run a coordination service
   → Avoid specialized service if not needed elsewhere
```

### Application 2: Leader Election và Work Allocation

```
LEADER ELECTION:
   Several instances of a service → ONE must be chosen as primary
   If leader fails → another takes over

Use: atomic CAS + ephemeral nodes
   First node to create ephemeral "/leader" node wins
   Others watch it; when it disappears → start new election

WORK ALLOCATION:
   Sharded resource (database, message streams, actor system,...):
   Decide which shard → which node
   When new nodes join → rebalance some shards
   When nodes fail → reassign failed node's shards

   Coordination service tracks: shard assignments, node health
   → Application reads assignments, implements rebalancing logic
```

### Application 3: Service Discovery

```
SERVICE DISCOVERY: finding which IP:port to connect to for a service

Each service instance registers when it starts:
   "My endpoint is 10.1.2.3:8080, I handle shard 7"

Clients query coordination service to find endpoint:
   Watch for changes → update routing table

WHY coordination service is OVERKILL for service discovery:
   Service discovery generally doesn't require linearizability
   More important: HIGHLY AVAILABLE and FAST
   If service discovery down → everything stops

BETTER: Cache service discovery info
   Clients unable to connect: bypass cache, retry with latest, update cache
   Refresh cache periodically (TTL-based)

   DNS-based: multiple layers of caching → good performance/availability

ZooKeeper OBSERVERS:
   Receive log, maintain data copy
   DON'T participate in consensus voting
   Reads may be stale (not linearizable)
   But: remain available if network interrupted + increase read throughput
```

### Application 4: Distributed Locks in Practice

```
"Only one thing should happen" pattern:
   Only one node processes specific task → no duplicate work
   Only one node writes to specific partition → no data corruption
   Only one instance is active (active-passive HA)

Implementation with coordination service:
   Try to acquire lock (CAS on key or create ephemeral node)
   If successful: do the work
   When done: release lock
   If crash: lease times out → lock automatically released

Real-world: Apache Curator (Java library)
   Higher-level tools on top of ZooKeeper client API
   Implements common patterns: leader election, distributed locking,...

   BUT: still complex! Better than implementing from scratch,
   still requires careful design.
```

---

## 4. ZooKeeper vs. etcd vs. Consul

| Feature           | ZooKeeper                         | etcd                            | Consul                      |
| ----------------- | --------------------------------- | ------------------------------- | --------------------------- |
| **Consensus**     | Zab                               | Raft                            | Raft                        |
| **Data model**    | Hierarchical znodes               | Key-value (prefix support)      | Key-value, services, checks |
| **Watch/Notify**  | One-time watches                  | Continuous stream               | HTTP long-poll/streaming    |
| **Reads**         | Sync or stale                     | Linearizable (default) or stale | Various                     |
| **Service mesh**  | No                                | No                              | Yes (with Envoy)            |
| **Health checks** | Session-based                     | TTL, k/v-based                  | Rich (HTTP, TCP, script)    |
| **Users**         | Hadoop, Kafka (old), HBase, Spark | Kubernetes, etcd-based dbs      | HashiCorp stack             |
| **Language**      | Java                              | Go                              | Go                          |

```
ZooKeeper: mature, battle-tested, but complex, Java-based
   → Many Hadoop-ecosystem tools use it
   → Kafka moving away from ZooKeeper (KRaft mode)

etcd: simpler API, Go-based, Kubernetes's choice
   → Clean API, good documentation
   → Linearizable reads by default (at cost of latency)

Consul: includes service mesh (built-in), richer health checks
   → Good for heterogeneous service infrastructure
   → HashiCorp ecosystem (Terraform, Vault, Nomad)
```

---

## 5. Scalability của Coordination Services

### Không scale để lưu nhiều data

```
Coordination services: NOT designed to hold data that changes thousands/sec
   → Use conventional database for that

Coordination data: SLOW-CHANGING
   "Node 10.1.1.23 is leader for shard 7"
   → Changes on timescale of MINUTES or HOURS (not ms!)

For fast-changing internal state of a service:
   → Apache BookKeeper: replicate fast-changing state
```

### Fixed small cluster for coordination

```
Storage system with thousands of shards:
   Running consensus over thousands of nodes: TERRIBLE (too slow)

BETTER: "Outsource" consensus to SMALL coordination service
   3 or 5 nodes for coordination service
   Thousands of storage nodes just consult coordination service

Benefits:
   Consensus overhead: only on 3-5 nodes (fast quorum)
   Storage nodes: NO consensus overhead (just read from coord service)
   Scales to arbitrary number of storage/processing nodes
```

---

## Tóm tắt phần này

```
COORDINATION SERVICES (ZooKeeper, etcd, Consul):
   NOT general-purpose databases
   Small coordination data in memory, replicated via consensus
   Modeled after Google's Chubby

4 FEATURES:
   1. Locks/Leases: atomic CAS → only one acquires
   2. Fencing tokens: monotonically increasing log IDs (zxid, revision)
   3. Failure detection: heartbeats + ephemeral nodes/lease TTL
   4. Change notifications: watches/streaming (no polling needed)

APPLICATIONS:
   Config management: store config, watch for changes
   Leader election: ephemeral node pattern
   Work allocation: shard assignment, rebalancing
   Service discovery: BUT overkill (use caching + TTL instead)

ZooKeeper (Zab) vs etcd (Raft) vs Consul (Raft + service mesh)

SCALABILITY:
   Small fixed cluster (3-5 nodes) for coordination
   Larger cluster for actual data/processing
   → Consensus overhead only on small coordination cluster
```
