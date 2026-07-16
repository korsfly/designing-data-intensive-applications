# Consensus Algorithms: Paxos, Raft, Zab

---

## 1. Consensus Problem — Định nghĩa Chính thức

### 4 Properties của Consensus

```
1. UNIFORM AGREEMENT:
   No two nodes decide different values

2. INTEGRITY:
   No node decides twice (once decided → never changes)

3. VALIDITY:
   If a node decides value v → v was proposed by some node
   (cannot decide value "out of thin air")

4. TERMINATION:
   Every node that does not crash eventually decides some value
   (liveness: algorithm makes progress)

Properties 1-3 = SAFETY (cannot be violated even without termination)
Property 4 = LIVENESS (must make progress eventually)

FLP Impossibility Result (Fisher, Lynch, Paterson, 1985):
   It is IMPOSSIBLE to guarantee consensus termination
   in an asynchronous system where even ONE node may crash
   → But: can solve in PARTIAL SYNCHRONY (works most of the time)
```

---

## 2. Single-Value Consensus → Shared Log

```
Most useful formulation: SHARED LOG (total order broadcast)
   Not just agreement on one value, but on SEQUENCE OF VALUES

Most consensus systems provide shared log out-of-the-box:
   Raft, Viewstamped Replication, Zab: shared log natively
   Paxos: single-value consensus → Multi-Paxos: shared log (practical version)

SHARED LOG = BASIS for:
   State machine replication (database replication)
   Serializable transactions (execute same operations in same order)
   Leader election
   Distributed locks/leases
   Atomic commit (fault-tolerant 2PC)
```

---

## 3. Architecture của Consensus Algorithms

### Common Structure (Raft, Multi-Paxos, Viewstamped Replication, Zab)

```
1. LEADER ELECTION:
   Quorum of nodes vote to elect a leader
   Leader gets EPOCH NUMBER (term in Raft, ballot in Paxos, view in VR)

2. LOG ENTRY PROPOSAL:
   Leader proposes entry to append to shared log
   Quorum of nodes vote to accept the entry

3. COMMITMENT:
   Once quorum votes yes → entry committed to log
   Confirmed to client that write succeeded
   Synchronously replicated to quorum BEFORE confirming
```

### Two Rounds of Voting

```
Round 1: ELECT A LEADER
   Any node can start election (triggered by timeout, no heartbeat)
   Election given NEW EPOCH NUMBER (higher than any previous)
   Nodes vote YES only if not aware of leader with higher epoch

Round 2: PROPOSE LOG ENTRIES
   Before appending, leader collects quorum votes
   Nodes vote YES if no higher-epoch leader known

CRUCIAL: Both quorums MUST OVERLAP
   At least 1 node in entry-vote quorum ALSO participated in recent election
   → Ensures new leader knows about all previously committed entries
```

### Epoch Number Mechanism

```
EPOCH NUMBER:
   Increments monotonically with each election
   Within each epoch: at most one leader (guaranteed unique)

If TWO LEADERS think they're current leader (from different epochs):
   Higher epoch number PREVAILS
   → No split brain (different from 2PC where both can commit!)

Analogy: election years
   Term 1: leader A
   (Leader A appears to fail)
   Term 2: leader B elected
   Leader A comes back: "I'm still leader!"
   → B has higher epoch → A must step down → B prevails
```

### 2 Rounds of Voting vs. 2PC

```
CONSENSUS (Raft, etc.):               2PC:
   Any node can start election           Only coordinator can request votes
   Only QUORUM needed to respond         ALL participants must respond
   Quorums overlap → safe                All-or-nothing → SPOF if coordinator fails
   Works as long as majority available   Stuck if coordinator crashes

→ CONSENSUS = fault-tolerant            2PC = can get stuck
```

---

## 4. Paxos

### Multi-Paxos (Practical Version)

```
Single-value Paxos: classic formulation
   → Each value independently decided
   → Expensive: 2 round trips per value

Multi-Paxos:
   1. Elect stable leader (Paxos phase 1 = prepare round)
   2. While leader stable: skip phase 1 for subsequent entries (only phase 2)
   → Single round trip per log entry when leader stable

Multi-Paxos ≡ Shared log (most practical use of Paxos is Multi-Paxos)
```

### Egalitarian Paxos (EPaxos)

```
PROBLEM with leader-based Paxos:
   Leader = bottleneck for all writes
   If leader's network connection poorly performing: bad performance for all

EGALITARIAN PAXOS (EPaxos):
   LEADERLESS: any node can propose values
   Conflicting proposals: resolved through standard Paxos rounds
   Non-conflicting proposals: fast-path (less coordination)

→ More robust against poorly performing nodes
→ Better performance for geographically distributed systems
```

---

## 5. Raft

### Key Design Philosophy

```
Raft designed for UNDERSTANDABILITY:
   Paxos: famous for being hard to understand
   Raft: same guarantees, designed to be more intuitive

Key Raft features:
   Strong leader: all writes go through leader
   Leader election: based on log completeness
   Log matching: ensures logs consistent across nodes

Raft published 2014 → quickly widely adopted:
   etcd, CockroachDB, TiDB, Consul, rqlite, YugabyteDB, TiKV, RabbitMQ quorum queues
```

### Raft Leader Election

```
Leader sends periodic HEARTBEATS to followers
If follower doesn't receive heartbeat within timeout → starts election:
   1. Increment term number (election term)
   2. Vote for itself
   3. Send RequestVote RPC to all other nodes

Node grants vote if:
   (a) Hasn't voted in this term yet
   (b) Candidate's log is AT LEAST AS UP-TO-DATE as own log

WHY (b)? Ensures new leader has all previously committed entries

Candidate wins if majority vote → becomes leader for that term
   Start sending heartbeats to prevent new elections
```

### Log Matching

```
Raft maintains LOG MATCHING INVARIANT:
   If two logs have an entry at same index with same term:
   → All entries up to that index are identical in both logs

New leader: may have entries NOT committed by old leader
   → These are OVERWRITTEN (not yet committed, so safe)

Raft log entry: (term, index, value)
   Committed if replicated to majority
   Not committed if only on leader (lost on leader crash)
```

### Pre-Vote Extension

```
PROBLEM: Leader isolated by network but other nodes can reach each other
   Isolated leader may not know it's isolated
   Other nodes start elections → leader continually forced to resign
   New leader elected → isolated leader comes back → another election...
   → System spends more time on elections than useful work!

PRE-VOTE PHASE (Raft extension):
   Before starting real election:
   → Ask peers: "Would you vote for me if I started an election?"
   → Only start actual election if likely to win

→ Prevents unnecessary disruptions from partitioned nodes
```

---

## 6. Zab (ZooKeeper Atomic Broadcast)

```
ZAB: consensus algorithm used in Apache ZooKeeper

Similar structure to Raft and Multi-Paxos:
   Leader broadcasts entries to followers
   Followers acknowledge
   Leader commits once majority ack

Primary-backup model:
   Leader (primary) receives all writes
   Followers (backups) receive replicated writes in order

Epoch-based leader election: similar to Raft's term numbers

ZooKeeper's ZAB = basis for ZooKeeper's strong consistency
   All ZooKeeper writes: linearizable
   ZooKeeper reads: NOT guaranteed linearizable by default
     (can enable with "sync" operation before read)
```

---

## 7. Viewstamped Replication (VR)

```
VR: early consensus algorithm (1988, before Paxos 1989)
   Similar to Paxos and Raft in structure

Views: equivalent to Raft's terms / Paxos's epochs
   View change = leader election

VR = foundation for many modern consensus systems
   PBFT (Byzantine fault-tolerant VR variant)
   Bolosky et al. used VR for Farsite distributed filesystem
```

---

## 8. Pros और Cons of Consensus

### Pros

```
✓ SAFETY: never makes incorrect decisions (split brain impossible)
✓ AUTOMATIC FAILOVER: no human intervention needed
✓ FAULT TOLERANCE: tolerates f failures with 2f+1 nodes
✓ "SINGLE-LEADER REPLICATION DONE RIGHT"
✓ Strong theoretical foundation: provably correct under crash-recovery model
✓ Any system with automatic failover NOT using proven consensus = likely unsafe
```

### Cons

```
✗ STRICT MAJORITY REQUIRED:
   3 nodes: tolerate 1 failure
   5 nodes: tolerate 2 failures
   → Cannot increase throughput by adding nodes
     (every node added makes algorithm SLOWER due to larger quorums)

✗ ONLY MAJORITY PORTION CAN MAKE PROGRESS:
   Network partition: minority side BLOCKED until partition heals
   → Availability reduced during partitions

✗ TIMEOUT SENSITIVITY:
   Rely on timeouts to detect failed leaders
   Too large: slow recovery
   Too small: unnecessary elections, poor performance

   Multi-region deployment: especially hard to tune
   (high variable network delays between regions)

✗ NETWORK SENSITIVITY:
   Raft: unreliable single network link → leadership continually bounces
   → Pre-vote phase helps but doesn't fully solve

   Paxos: similar issues with leader-based approach
   EPaxos: leaderless → more robust against poor network performance

✗ RECONFIGURATION COMPLEXITY:
   Most algorithms assume fixed set of voting nodes
   Adding/removing nodes requires careful reconfiguration protocol
   (important for adding regions or migrating between locations)
```

---

## Tóm tắt phần này

```
CONSENSUS PROPERTIES:
   Uniform agreement + Integrity + Validity + Termination
   FLP: impossible to guarantee termination in async system with crash
   → Partial synchrony: works most of the time (practical systems)

COMMON STRUCTURE:
   1. Elect leader (quorum vote + epoch number)
   2. Propose log entry (quorum vote, quorums must overlap)
   3. Commit (majority ack → confirmed to client)

ALGORITHMS:
   Paxos: classic, complex, Multi-Paxos = practical shared log
   Raft: understandable, widely adopted (etcd, CockroachDB, TiDB)
           Pre-vote phase for network robustness
   Zab: ZooKeeper's algorithm, primary-backup model
   VR: early algorithm (1988), foundation for others
   EPaxos: leaderless, robust against poor network connections

EPOCH NUMBERS: prevent split brain (higher epoch prevails)
TWO ROUNDS: leader election + entry proposal (quorums overlap)
CONSENSUS ≠ 2PC: fault-tolerant vs. SPOF coordinator

PROS/CONS:
   + Safety, automatic failover, fault tolerance
   - Majority required, adds overhead for every op, timeout-sensitive
   - Only majority portion can make progress during partition
```
