# Knowledge, Truth, và Lies trong Distributed Systems

---

## 1. Bản chất Epistemological của Distributed Systems

```
Node trong network CANNOT KNOW ANYTHING FOR SURE về other nodes
   → Chỉ có thể GUESS dựa trên messages nhận được (hoặc KHÔNG nhận)

Node biết về state của remote node:
   ONLY through message exchange

Remote node không respond:
   KHÔNG THỂ know state của nó
   (network problems CANNOT be reliably distinguished from node problems)

→ Borderline philosophical:
   "What do we know to be true or false in our system?"
   "How sure can we be of knowledge if perception mechanisms are unreliable?"

→ Fortunately: we don't need to solve meaning of life!
   → State assumptions (SYSTEM MODEL)
   → Design actual system to meet those assumptions
   → Algorithms can be PROVED correct within system model
```

---

## 2. The Majority Rules — Quorum Decision Making

### 3 Tình huống minh họa

**Tình huống 1: Asymmetric network fault**

```
Node có thể RECEIVE tất cả messages gửi đến nó
NHƯNG: TẤT CẢ outgoing messages bị dropped/delayed

→ Other nodes không nghe thấy responses
→ After timeout: other nodes declare it DEAD
→ Node "kicking and screaming 'I'm not dead!'"
   nhưng không ai nghe thấy
```

**Tình huống 2: Semi-disconnected nhưng aware**

```
Node nhận thấy messages của mình NOT BEING ACKNOWLEDGED
→ Biết có network fault
NHƯNG: vẫn bị wrongly declared dead, không thể làm gì
```

**Tình huống 3: Process Pause (ghê nhất)**

```
Node pauses execution for 1 MINUTE
   → No requests processed, no responses sent
   → Other nodes wait, retry, declare it DEAD

Pause finishes → node's threads continue AS IF NOTHING HAPPENED
   → "Supposedly dead node suddenly raises its head out of coffin"
   → Node doesn't even realize it was paused
   → Từ its perspective: hardly any time passed
```

### Bài học: Node Cannot Trust Its Own Judgment

```
A node CANNOT NECESSARILY trust its own judgment of a situation

Distributed system CANNOT exclusively rely on SINGLE NODE:
   Single node may fail → system stuck and unable to recover

→ Distributed algorithms rely on QUORUM (voting among nodes):
   Decisions require MINIMUM NUMBER OF VOTES from several nodes
   → Reduce dependence on any particular node

Includes declaring nodes dead:
   If QUORUM OF NODES declares another node dead
   → That node IS CONSIDERED DEAD, even if IT still feels alive
   → Individual node must ABIDE by quorum decision and STEP DOWN

Most common: absolute majority (more than half the nodes)
   3 nodes: tolerate 1 faulty
   5 nodes: tolerate 2 faulty

Only 1 majority possible in system at any time
   → No conflicting decisions simultaneously
   → Chapter 10: Consensus algorithms in detail
```

---

## 3. Distributed Locks and Leases — Common Source of Bugs

### Khi nào cần Distributed Lock?

```
System requires ONLY ONE OF SOME THING:
   1. Only 1 leader per DB shard (avoid split brain)
   2. Only 1 transaction/client update a particular resource
      (prevent corruption by concurrent writes)
   3. Only 1 node processes a given input file
      (avoid wasted effort from multiple nodes doing same work)
```

### Vấn đề: Client Holding Lease for Too Long (Figure 9-4)

```
Scenario:
   Client 1 acquires lease (lock with timeout)
   Client 1 PAUSES for too long → lease EXPIRES
   Client 2 acquires NEW LEASE for same file
   Client 2 starts writing

   Client 1 comes back → believes (incorrectly) still has VALID LEASE
   Client 1 continues writing

→ SPLIT BRAIN: 2 clients writing simultaneously → FILE CORRUPTED
(HBase had this exact bug!)
```

### Vấn đề: Delayed Network Request (Figure 9-5)

```
Scenario:
   Client 1 crashes, sends write request JUST BEFORE crash
   Request DELAYED in network (can be delayed 1+ minute!)
   Client 1's lease times out → Client 2 acquires lease
   Client 2 issues write

   DELAYED request from Client 1 FINALLY ARRIVES:
   Client 1 lease already expired
   But request has already been sent!

→ Same corruption scenario as Figure 9-4
   Client 1 is now a "ZOMBIE" — former leaseholder acting as if still valid
```

---

## 4. Fencing Tokens — Robust Solution

### Zombie Problem

```
ZOMBIE: Former leaseholder that has NOT YET FOUND OUT it lost the lease
   → Still acting as if it were the current leaseholder

Cannot completely eliminate zombies → Instead: ensure they CAN'T DO DAMAGE
→ Gọi là: FENCING OFF THE ZOMBIE
```

### Naive Approach: Shoot the Zombie (STONITH)

```
STONITH (Shoot The Other Node In The Head):
   Disconnect zombie from network
   Shutdown VM via cloud provider's management interface
   Physically power down machine

WHY THIS DOESN'T WORK:
   Doesn't protect against large network delays (Figure 9-5):
      Zombie's delayed request may arrive BEFORE we can shut it down
   All nodes could shut each other down!
   By time zombie detected + shut down → data may already be corrupted
```

### Fencing Tokens — Correct Solution (Figure 9-6)

```
Every time lock service grants lock/lease:
   Also returns FENCING TOKEN = number that INCREASES EVERY TIME lock granted
   (monotonically increasing)

Client must include fencing token in EVERY WRITE REQUEST to storage

Storage service: tracks HIGHEST FENCING TOKEN it has processed
   Reject any write with LOWER OR EQUAL token number
```

```
Example (Figure 9-6):
   Client 1 acquires lease with token 33
   → Client 1 long pause → lease expires

   Client 2 acquires lease with token 34
   → Client 2 writes to storage with token 34

   Client 1 comes back → sends write with token 33
   Storage: already processed token 34 → REJECT request with token 33
   → Client 1 (zombie) is FENCED OFF
```

### Tên gọi khác của Fencing Tokens

| System                       | Tên                                             |
| ---------------------------- | ----------------------------------------------- |
| Google Chubby (lock service) | Sequencer                                       |
| Kafka                        | Epoch number                                    |
| Paxos (consensus algorithm)  | Ballot number                                   |
| Raft (consensus algorithm)   | Term number                                     |
| ZooKeeper                    | zxid (transaction ID) / cversion (node version) |
| etcd                         | Revision number + Lease ID                      |
| Hazelcast                    | FencedLock API                                  |

### Implement Storage Service với Fencing

```
Storage service needs: way to CHECK if write based on outdated token

SIMPLEST: support write that SUCCEEDS ONLY IF object has NOT been written
   by another client since current client last read it
   → Like ATOMIC CAS (Compare-and-Set) operation

Object storage services:
   Amazon S3: CONDITIONAL WRITES
   Azure Blob Storage: CONDITIONAL HEADERS
   Google Cloud Storage: REQUEST PRECONDITIONS

With fencing tokens in LWW replicated systems:
   Put writer's fencing token in MOST SIGNIFICANT BITS of timestamp
   → New leaseholder timestamps always > old leaseholder timestamps
   → Works even if old leaseholder's writes arrive late
```

### Fencing vs Optimistic Concurrency Control (OCC)

```
Both: prevent conflicting concurrent operations

DIFFERENCE:
   OCC (SSI, Chapter 8): failure can be RETRIED (not permanent)
   Fencing: PERMANENT barrier — zombie never gets through
```

---

## 5. Byzantine Faults — Malicious Nodes

### Tình huống Byzantine

```
Fencing token scenario: node bị PAUSED (innocent mistake)
   → Chỉ cần OUTPACE (outs-timestamp, out-sequence) zombie

BYZANTINE FAULT: node có thể DELIBERATELY LIE, SEND WRONG MESSAGES
   Ví dụ: adversary controlling nodes
          Radiation causing bit flips in memory
          Hardware silently returning wrong results
```

### Byzantine Fault-Tolerant Systems

```
DISTRIBUTED CONSENSUS với Byzantine faults:
   Algorithm phải tolerate LYING NODES (≤ 1/3 nodes malicious)
   → Much more complex than crash-fault-tolerant algorithms

USE CASES:
   Spacecraft/aircraft (cosmic radiation causing bit flips)
   Nuclear power plants
   Bitcoin/Ethereum blockchain (mistrusting participants)
   Peer-to-peer networks (untrusted participants)

SERVER-SIDE DISTRIBUTED SYSTEMS (databases, etc.):
   Generally assume NON-BYZANTINE faults only
   → Nodes may crash but HONEST (no malicious behavior)
   → Much simpler algorithms sufficient
```

---

## Tóm tắt phần này

```
NODE CANNOT TRUST OWN JUDGMENT:
   May be wrongly declared dead while still alive
   May be alive but unaware it was paused for 1+ minute
   → Quorum decisions needed: majority vote, not single-node

DISTRIBUTED LOCKS:
   Leases = locks with timeout (used for: single leader, exclusive resource access, single processor)
   Problems: process pause + lease expiry → zombie leaseholder
             delayed network request from zombie → corruption

ZOMBIE PROBLEM:
   Former leaseholder acting as if still valid
   STONITH: not robust enough (doesn't handle network delays)

FENCING TOKENS (CORRECT SOLUTION):
   Monotonically increasing token from lock service
   Client includes token in every write
   Storage rejects lower-or-equal tokens
   Names: sequencer (Chubby), epoch (Kafka), ballot (Paxos), term (Raft)

   Implementation: conditional writes (S3, Azure Blob, GCS)
                   or LWW with token in most-significant timestamp bits

BYZANTINE FAULTS: nodes lying/malicious
   Complex algorithms needed (≤ 1/3 malicious nodes)
   Server-side distributed systems: usually assume non-Byzantine
```
