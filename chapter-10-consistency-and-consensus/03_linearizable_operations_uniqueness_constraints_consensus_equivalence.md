# Linearizable Operations, Uniqueness Constraints, và Consensus Equivalences

---

## 1. Linearizable Compare-and-Set (CAS)

### Định nghĩa

```
CAS operation (atomically):
   READ current value of register
   IF current value = expected value:
      WRITE new value, return SUCCESS
   ELSE:
      Return FAILURE (no change)

Example:
   CAS(x, old_value=5, new_value=10):
   → If x == 5: set x = 10, return ok
   → Otherwise: return error, x unchanged
```

### Tại sao cần Linearizable CAS?

```
Concurrent CAS from multiple clients:
   Client A: CAS(x, 5, 10) — if x is 5, change to 10
   Client B: CAS(x, 5, 20) — if x is 5, change to 20

   Only ONE can succeed (x cannot be both 10 and 20)
   → Requires consensus: which client wins?

   Linearizable CAS ensures:
   EXACTLY ONE of the two operations succeeds
   Result is deterministic and consistent across all nodes
```

### CAS applications

```
1. Leader election: node tries CAS on "current_leader" key
   → Only first to CAS wins; others fail → know someone else is leader

2. Distributed locking: try to CAS lock from "unlocked" to "locked"
   → Only one succeeds; others must retry or fail

3. Implementing fetch-and-add: CAS(counter, current, current+1)
   → Retry until CAS succeeds → atomically increment
   (Less efficient than native fetch-and-add under contention)
```

---

## 2. Uniqueness Constraints

### Problem

```
Many databases enforce uniqueness constraints:
   - Username must be unique (only one user per username)
   - Email address must be unique
   - Booking: only one reservation per seat

Single-node: CHECK then INSERT within transaction
   → Works if only one node

Distributed system: race condition
   - Two clients simultaneously try to register same username
   - Both check: username available
   - Both INSERT: duplicate username!

→ Without linearizability: CONSTRAINT VIOLATION POSSIBLE
```

### Solutions requiring Linearizability

```
Option 1: LINEARIZABLE CAS per username
   Try CAS(username, null, user_id)
   Only first to succeed gets the username
   → Requires consensus for each username registration

Option 2: TOTAL ORDER BROADCAST (equivalent to CAS)
   Append "register username X" to shared log
   First log entry for X wins; subsequent ones fail
   → Same as consensus on "who registered X first"

Option 3: SHARD by username
   All operations on username X go to same node
   → Single-node can enforce constraint within that shard
   → But: cross-shard uniqueness still needs consensus
```

### Multi-Object Uniqueness

```
Sometimes uniqueness spans multiple objects:
   Two different tables must agree (e.g., calendar bookings)

→ Requires DISTRIBUTED TRANSACTION or CONSENSUS
   across multiple shards/nodes

→ This is exactly why ATOMIC COMMITMENT (Chapter 8 2PC) is needed
   and why 2PC ≡ Consensus
```

---

## 3. Consensus Equivalences: A Unified View

### 6 Problems Equivalent to Consensus

```
The following problems are ALL EQUIVALENT:
   If you have a solution to any one → can build solutions to all others

1. SINGLE-VALUE CONSENSUS
   Nodes agree on single value; once decided, never change

2. LINEARIZABLE CAS REGISTER
   Atomic compare-and-set; only one winner among concurrent requests

3. SHARED LOG (Total Order Broadcast)
   All nodes agree on same ordered sequence of entries
   (= atomic broadcast with reliable + totally ordered delivery)

4. ATOMIC TRANSACTION COMMIT (2PC equivalent if fault-tolerant)
   All nodes either all commit or all abort
   (2PC without SPOF coordinator = consensus)

5. LINEARIZABLE FETCH-AND-ADD (consensus number = ∞)
   Atomically increment counter and return old value
   (Note: plain fetch-and-add only solves consensus for 2 nodes)

6. UNIQUENESS CONSTRAINTS / LOCKS / LEASES
   Under concurrent access, exactly one request wins
```

### Why all Equivalent?

```
KEY INSIGHT:
   All these problems require: "among concurrent requests, exactly one wins"
   → Requires agreement among nodes on WHICH ONE WINS
   → That agreement = CONSENSUS

Single-node: TRIVIALLY solvable (single dictator decides)
Distributed: requires consensus algorithm

Shared log implements all of them:
   Single-value: decide value that appears first in log
   CAS: append "attempt CAS" to log; first entry for key wins
   Fetch-and-add: counter = sum of all increment entries in log
   Uniqueness: first log entry for given key wins
   Atomic commit: consensus on "commit" or "abort" for transaction
```

### Consensus ≢ 2PC (Important Distinction!)

```
CONSENSUS (Raft, Paxos, Zab): fault-tolerant
   Any node can start election
   Only QUORUM needed to respond
   Can make progress as long as MAJORITY available

2PC (Chapter 8): NOT truly fault-tolerant
   ONLY coordinator requests votes
   Requires YES from ALL participants (not just majority)
   If coordinator crashes → in-doubt state (stuck)

→ 2PC with replicated coordinator (using consensus) = fault-tolerant 2PC
   → Some systems do this (CockroachDB, TiDB)
```

---

## 4. Cross-Channel Timing Dependencies

### Problem

```
Example (Figure 10-4): Image resizing service
   1. Client uploads photo to storage service
   2. Client sends message "photo uploaded" to message queue
   3. Message queue delivers to image resizer
   4. Image resizer reads photo from storage → processes → stores thumbnail

Race condition (without linearizability):
   Step 3 (message delivered) may happen BEFORE storage service
   has replicated the photo to all nodes
   → Image resizer reads from replica that hasn't received photo yet
   → Gets file-not-found error!

The ordering is:
   Photo upload → Message queue delivery → Image resizer reads
   But with async replication:
   Message delivery may APPEAR to happen before photo fully committed
```

### Why Linearizability Helps

```
Linearizable storage:
   Once write confirmed (upload complete):
   → ALL subsequent reads from ANY replica see the photo
   → Image resizer ALWAYS finds the photo

Without linearizability:
   Must add explicit coordination:
   - Polling/retry (wait for photo to appear)
   - Or pass storage version/timestamp in message
     (image resizer reads from replica with version ≥ that timestamp)
```

---

## Tóm tắt phần này

```
LINEARIZABLE CAS:
   Read + conditional write atomically
   Concurrent CAS: only ONE succeeds (consensus problem!)
   Applications: leader election, distributed locking, atomic increment

UNIQUENESS CONSTRAINTS:
   Distributed uniqueness requires consensus or single-node sharding
   Equivalent to CAS on "who registered X first"

CONSENSUS EQUIVALENCES:
   Single-value consensus ≡ CAS ≡ Shared Log ≡ Atomic Commit
   ≡ Fetch-and-add ≡ Distributed Locks/Leases
   All equivalent: solution to one → solution to all
   All require: "among concurrent requests, exactly one wins"

   Single-node: trivial (dictator decides)
   Distributed: requires consensus algorithm

CONSENSUS ≠ 2PC:
   2PC: coordinator SPOF, needs ALL participants
   Consensus: any node can start election, only quorum needed

CROSS-CHANNEL TIMING:
   Message queue + storage = two channels
   Linearizable storage prevents race condition between channels
```
