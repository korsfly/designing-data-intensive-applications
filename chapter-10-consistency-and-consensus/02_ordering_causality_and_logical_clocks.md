# Ordering, Causality, và Logical Clocks

---

## 1. Tại sao Ordering quan trọng?

```
ORDER in which things happen = FUNDAMENTAL TO DISTRIBUTED SYSTEMS:
   Chapter 6: Replication lag → read-after-write violation
   Chapter 8: SSI detects serialization conflicts
   Chapter 9: Fencing tokens (monotonically increasing)
   Chapter 3: Event sourcing (ordered log of events)

CAUSALITY: events that causally depend on each other must happen in order
   "A CAUSES B" → A must happen before B in any execution

   Example: "You can only answer a question that has been asked"
   → Question must come BEFORE answer in any meaningful order
```

---

## 2. Causality và Ordering

### Causal Order (Partial Order)

```
Happens-before relationship (Chapter 6):
   A HAPPENS BEFORE B: B knows about / depends on / builds on A

CONCURRENT: neither A happens before B, nor B before A

CAUSAL ORDER = PARTIAL ORDER:
   Not all events are ordered
   Two concurrent events: NEITHER happened before the other

TOTAL ORDER: all events comparable (every pair A,B: A before B or B before A)
   Examples: natural numbers (1,2,3,...), linearizable operations
```

### Causally Consistent vs. Linearizable

```
CAUSAL CONSISTENCY:
   Weakest consistency model that does not allow anomalies
   that arise from reordering causally related operations

   Preserves causal order
   Allows concurrent operations to be in any order
   (concurrent events don't have causal dependency → any order OK)

LINEARIZABILITY implies CAUSAL CONSISTENCY:
   Linearizable: total order, respects real time
   → Also respects causality (if A happens before B in real time → A before B in order)
   → But: linearizability STRONGER (also orders concurrent ops)

MANY WEAKER CONSISTENCY MODELS maintain causal consistency:
   Read-your-writes (within session), monotonic reads, consistent prefix
   → All causal guarantees
   → But less overhead than linearizability
```

---

## 3. Lamport Clocks

### Vấn đề: Determine Causal Order without Linearizability

```
Without total order (no linearizability):
   How to determine which of two concurrent writes happened first?

Lamport clocks: simple mechanism for CAUSAL ORDERING

Based on Leslie Lamport (1978): fundamental insight of distributed systems
```

### Algorithm

```
Each node has:
   NODE ID: unique (e.g., IP, node number)
   COUNTER: incrementing integer, starts at 0

For each operation:
   Counter = max(own_counter, received_counter) + 1
   Timestamp = (counter, node_id)

Operations:
   Local event: counter++
   Send message: include current (counter, node_id) in message
   Receive message: counter = max(local_counter, received_counter) + 1
```

### Ordering với Lamport Timestamps

```
Compare (counter_a, node_id_a) vs (counter_b, node_id_b):
   1. Compare counters: LARGER counter = LATER
   2. If counters EQUAL: compare node IDs (tiebreaker)

→ TOTAL ORDER of all operations across all nodes!

KEY PROPERTY (Lamport's theorem):
   If A happens-before B → timestamp(A) < timestamp(B)

CONVERSE IS NOT TRUE:
   timestamp(A) < timestamp(B) does NOT imply A happens-before B
   (may be concurrent operations with adjacent timestamps)
```

### Giới hạn của Lamport Clocks

```
Problem: cannot use Lamport timestamp to determine if 2 operations are concurrent

   timestamp(A) < timestamp(B): may be A→B OR A||B (concurrent)
   → Cannot distinguish causality from concurrency

Also: cannot determine "which username was registered first" in real-time
   → Lamport order may not match real-time order exactly
   (violates linearizability)

→ Need TOTAL ORDER BROADCAST for stronger guarantees
```

---

## 4. Hybrid Logical Clocks (HLC)

### Motivation

```
Lamport clocks: purely based on message exchange
   → May diverge greatly from real time
   → Hard to reason about "approximately when" an event happened

Physical clocks (time-of-day): close to real time
   → But: can jump backward, unreliable for ordering

HYBRID LOGICAL CLOCK (HLC): best of both:
   Tracks both logical (causal) order AND physical time
   Never goes backward (unlike time-of-day)
   Close to physical time (unlike pure Lamport)
```

### HLC Algorithm (simplified)

```
State per node: (physical_time, logical_counter)

On each event:
   pt = max(own_physical_time, received_physical_time, current_physical_clock)

   if pt > own_physical_time:
       logical_counter = 0
   elif pt == received_physical_time:
       logical_counter = max(own_logical, received_logical) + 1
   else:
       logical_counter++

   own_physical_time = pt

HLC timestamp = (pt, logical_counter, node_id)
```

### HLC Properties

```
HLC timestamps are:
   ✓ Causally consistent (if A→B then HLC(A) < HLC(B))
   ✓ Close to physical time (within bounded drift)
   ✓ Never go backward
   ✓ Totally ordered (like Lamport, for any two events)

Used by: CockroachDB, YugabyteDB, TiDB
   → Can serve as transaction IDs
   → Close to physical time → useful for time-travel queries (snapshot at time T)
```

---

## 5. ID Generators và Logical Clocks

### Single-Node Autoincrement

```
SIMPLEST: Single-node, autoincrement primary key
   ✓ Linearizable (single node)
   ✓ Totally ordered, causally consistent
   ✗ SPOF: single node is bottleneck and failure point

Used by: PostgreSQL sequences, MySQL autoincrement
```

### Distributed ID Generation Schemes

| Scheme                         | Properties                                                              |
| ------------------------------ | ----------------------------------------------------------------------- |
| **Twitter Snowflake** (64-bit) | Timestamp (41b) + datacenter ID (5b) + machine ID (5b) + sequence (12b) |
| **UUID**                       | 128-bit random: very low collision probability; NOT ordered             |
| **MongoDB ObjectID**           | 12-byte: timestamp + random + incrementing counter                      |
| **ULID / KSUID**               | Timestamp + random: sortable, URL-safe                                  |

```
PROBLEM with timestamp-based IDs (Snowflake, etc.):
   If clocks NOT perfectly synchronized:
   Node with FASTER clock generates HIGHER IDs → appears to go "first"
   Even if causally LATER than operations on other nodes

   → Does NOT respect causal order!
   → Not linearizable (timestamps can be wrong, out of sync)

→ For strict causal ordering: need Lamport/HLC clocks or consensus
```

### Linearizable ID Generator

```
LINEARIZABLE ID generator requires consensus:
   Single-node CAS: atomically read counter, increment, return old value
   → Linearizable fetch-and-add operation

   Distributed: use consensus algorithm (Paxos, Raft)
   → Each "slot" in shared log = one ID
   → First node to append gets that ID

Used by: etcd leases, ZooKeeper zxid
```

---

## 6. Total Order Broadcast

### Định nghĩa

```
TOTAL ORDER BROADCAST (atomic broadcast):
   Protocol for exchanging messages between nodes with 2 properties:

   1. RELIABLE DELIVERY:
      No messages are lost
      If message delivered to one node → delivered to ALL nodes

   2. TOTALLY ORDERED DELIVERY:
      Messages delivered to EVERY NODE in SAME ORDER

(If node crashes → wait until recovered, then deliver)
```

### Equivalent Abstractions

```
TOTAL ORDER BROADCAST ≡ CONSENSUS ≡ SHARED LOG

They are all EQUIVALENT problems:
   Solution to one → solution to any other

SHARED LOG = sequence of entries all nodes agree on
   = total order broadcast output (each delivered message = one entry)
   = consensus on "what is entry N"

Single-value consensus → Total order broadcast:
   Run many instances (one per "slot" in the log)
   When value chosen for slot N → append to log
   → Slot N decided = consensus achieved for that position
```

### Total Order Broadcast và Linearizability

```
Total order broadcast: NOT same as linearizability

TOB is ASYNCHRONOUS:
   Messages delivered in fixed order (agreed upon)
   But: no guarantee on timing (maybe delayed)
   → Cannot guarantee "read your writes" without additional mechanism

Can BUILD linearizable storage using TOB:
   Linearizable write: append to log → wait for delivery → confirm
   Linearizable read:
     Option 1: append read request to log too → wait for position → read then
     Option 2: read from up-to-date replica (if such guarantee exists)

CONVERSELY: can build TOB from linearizable storage:
   Use linearizable register to store sequence number
   Every node atomically increments + uses as message position
   → Messages sent to all nodes with sequence number → deliver in order
```

---

## Tóm tắt phần này

```
CAUSALITY:
   Partial order (not all events ordered)
   Causal consistency: weakest model preventing causality violations
   Linearizability → causal consistency (stronger than needed for causality)

LAMPORT CLOCKS:
   (counter, node_id): total order, causal consistency
   A→B implies timestamp(A) < timestamp(B)
   CONVERSE NOT TRUE: timestamps don't distinguish concurrent from causal

HYBRID LOGICAL CLOCKS (HLC):
   Logical + physical time: causally consistent + close to real time
   Used: CockroachDB, YugabyteDB, TiDB

ID GENERATORS:
   Distributed timestamp-based (Snowflake, etc.): NOT causally consistent
   Linearizable: requires consensus (single CAS or shared log position)

TOTAL ORDER BROADCAST:
   Reliable delivery + totally ordered delivery
   Equivalent to consensus and shared log
   ≡ Foundation of: state machine replication, event sourcing
```
