# Unreliable Clocks và Process Pauses

---

## 1. Hai loại Clock trong Modern Computers

### Time-of-Day Clocks (Wall Clock)

```
Trả về: current date and time theo calendar
API: clock_gettime(CLOCK_REALTIME) — Linux
     System.currentTimeMillis() — Java
Unit: seconds (or ms) since epoch (midnight UTC, Jan 1 1970)

Synced với NTP
→ Timestamp từ machine A (ideally) = same as machine B

VẤN ĐỀ:
   Clock quá fast so với NTP server:
   → May be FORCIBLY RESET → jump BACKWARD in time
   Leap seconds: minute = 59 or 61 seconds → mess up timing assumptions
   DST jumps (nếu không dùng UTC)

→ KHÔNG PHẢI SỬ DỤNG để measure elapsed time!
```

### Monotonic Clocks

```
Dùng cho: đo DURATION (time intervals), timeouts, response time
API: clock_gettime(CLOCK_MONOTONIC) hoặc CLOCK_BOOTTIME — Linux
     System.nanoTime() — Java

"Monotonic" = guaranteed to ALWAYS MOVE FORWARD
   (Time-of-day clock có thể jump back)

Absolute value: MEANINGLESS (có thể là ns since boot)
→ KHÔNG SO SÁNH monotonic clock values từ 2 computers!

Multi-socket server: SEPARATE TIMER PER CPU
   OS compensates, but take monotonicity guarantee with pinch of salt

NTP: có thể adjust RATE of monotonic clock (slewing), tối đa ±0.05%
     → Không cause jumps, nhưng ảnh hưởng speed

→ Safe for measuring durations (không assume cross-node sync)
```

---

## 2. Clock Synchronization và Accuracy

### Nguồn gốc Clock Drift

```
Quartz crystal oscillator: NOT perfectly accurate
   Drift depends on temperature, varies machine to machine

Google assumes: drift up to 200 PPM (parts per million)
   = 6 ms drift nếu resync mỗi 30 seconds
   = 17 SECONDS DRIFT nếu resync once a day

NTP mitigates drift, nhưng LIMITED BY NETWORK DELAY:
   Best achievable qua internet: ~35 ms accuracy
   Occasional spikes in network delay → errors ~1 second
   Large network delays → NTP client gives up entirely
```

### Các vấn đề Clock Accuracy

| Vấn đề                              | Chi tiết                                                                 |
| ----------------------------------- | ------------------------------------------------------------------------ |
| **Quartz drift**                    | Depends on temperature, 200 PPM đối với Google's servers                 |
| **NTP forced reset**                | Clock ahead of server → forcibly jump backward                           |
| **Firewall blocking NTP**           | Misconfiguration → large unnoticed drift                                 |
| **NTP accuracy limited by network** | Best: 35ms; typical: more; with congestion: >100ms                       |
| **Wrong NTP servers**               | Servers off by hours; NTP clients query several + ignore outliers        |
| **Leap seconds**                    | Minute 59 or 61 seconds → crash many systems (disabled from 2035)        |
| **VM clock virtualization**         | VM paused → clock jumps forward when resumed; NTP inside VM doesn't know |
| **Mobile/embedded devices**         | Users can set arbitrary wrong time; don't trust hardware clock           |

### High-Accuracy Clock (MiFID II)

```
MiFID II (EU financial regulation):
   High-frequency trading funds: sync clocks within 100 MICROSECONDS of UTC

Cách đạt được: Special hardware (GPS receivers/atomic clocks)
               Precision Time Protocol (PTP)
               Careful deployment and monitoring

Giới hạn: GPS signals can easily be JAMMED
   (near military facilities: happens frequently)

Some cloud providers: high-accuracy clock sync for VMs
   But: requires care, can drift quickly if misconfigured
```

---

## 3. Relying on Synchronized Clocks — Nguy hiểm

### Timestamps for Ordering Events — LWW Problem

```
Multi-leader replication (Figure 9-3):
   Client A writes x = 1 on node 1; replicated to node 3
   Client B increments x on node 3 (x = 2)
   Both replicated to node 2

Timestamps:
   Node 1's write (x=1): timestamp 42.004 seconds
   Node 3's write (x=2): timestamp 42.003 seconds
   → B's write CAUSALLY LATER but has EARLIER TIMESTAMP!

With LWW conflict resolution:
   Node 2 sees x=1 has HIGHER timestamp → keeps x=1 → x=2 DISCARDED!
   → Increment is LOST (silent data loss)
```

### 3 Vấn đề của LWW với Real-Time Clocks

| Vấn đề                                          | Chi tiết                                                                                                                    |
| ----------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------- |
| **Database writes disappear**                   | Node với lagging clock → cannot overwrite values written by node với faster clock until clock skew elapsed                  |
| **Cannot distinguish sequential vs concurrent** | LWW cannot tell if writes are sequential (B after A) or truly concurrent (neither aware of other)                           |
| **Same timestamp**                              | 2 nodes independently generate writes with same timestamp (ms resolution) → tiebreaker needed → still can violate causality |

```
"Even with tightly NTP-synchronized clocks:
 you could send packet at 100ms (sender's clock)
 and arrive at 99ms (recipient's clock)
 → Appears packet arrived BEFORE it was sent"

→ Correct ordering would require clock error << network delay
   → IMPOSSIBLE with current technology
```

### Giải pháp: Logical Clocks

```
PHYSICAL CLOCKS: measure actual elapsed time (time-of-day, monotonic)
   → Unreliable for event ordering across nodes

LOGICAL CLOCKS: based on INCREMENTING COUNTERS (not oscillating quartz)
   → Only track RELATIVE ORDERING (happened-before relationships)
   → NOT measure time-of-day or elapsed seconds

Xem: "ID Generators and Logical Clocks" (Chapter 10)
```

### Clock Readings với Confidence Interval

```
Đọc clock với microsecond precision:
   KHÔNG CÓ NGHĨA là accurate đến microsecond

Drift vài ms là common, dù sync với local NTP mỗi phút
Best với public internet NTP: tens of milliseconds
With congestion: >100ms errors

→ Better to think of clock reading as RANGE, not point:
   "Time is between 10.3 and 10.5 seconds past the minute (95% confident)"

Most systems DON'T expose this uncertainty:
   clock_gettime: return value doesn't tell you expected error
```

### Google Spanner's TrueTime API

```
TrueTime API: explicitly reports CONFIDENCE INTERVAL on local clock
   Returns [earliest, latest] timestamps

Two intervals A = [Aearliest, Alatest] and B = [Bearliest, Blatest]:
   Non-overlapping (Alatest < Bearliest): B DEFINITELY happened after A
   Overlapping: unsure about order

Spanner: deliberately WAIT for length of confidence interval before committing
   → Ensures any transaction reading data is at sufficiently later time
   → Confidence intervals DON'T OVERLAP

To keep wait time short: minimize clock uncertainty
   → Google deploys GPS receiver/atomic clock in EACH datacenter
   → Clocks synchronized within ~7 ms

Other systems adopting similar approach:
   YugabyteDB: leverages ClockBound on AWS
```

---

## 4. Process Pauses

### Ví dụ: Lease-based Leader

```java
while (true) {
    request = getIncomingRequest();
    if (lease.expiryTimeMillis - System.currentTimeMillis() < 10000) {
        lease = lease.renew();
    }
    if (lease.isValid()) {
        process(request);
    }
}
```

**2 vấn đề với code này:**

```
1. Relying on SYNCHRONIZED CLOCKS:
   Expiry time set by DIFFERENT machine, compared to LOCAL clock
   → Clocks out of sync → "start doing strange things"

2. Even với monotonic clock: ASSUMES LITTLE TIME PASSES
   between checking time and processing request

   Thread có thể PAUSE for 15 SECONDS right before lease.isValid():
   → Lease expired, another node took over as leader
   → But THIS THREAD doesn't know → processes request anyway
   → UNSAFE!
```

### 8 Nguyên nhân Process Pauses (Có thể xảy ra bất kỳ lúc nào)

| Nguyên nhân                                  | Chi tiết                                                                                                         |
| -------------------------------------------- | ---------------------------------------------------------------------------------------------------------------- |
| **Thread contention**                        | Shared lock/queue → threads spend time waiting; worse on more CPU cores                                          |
| **GC pause**                                 | JVM stop-the-world GC: historically minutes; now improved but still noticeable                                   |
| **VM suspend/resume**                        | VM suspended (contents saved to disk) for live migration → arbitrary duration pause                              |
| **End-user device suspend**                  | Laptop lid closed → execution suspended indefinitely                                                             |
| **OS context switch / hypervisor VM switch** | Running thread paused at arbitrary code point; VM: "steal time"                                                  |
| **Synchronous disk access**                  | Thread waits for slow I/O; disk access can happen surprisingly (Java classloader); EBS latency = network latency |
| **Memory paging / thrashing**                | Page fault → load from disk; high memory pressure → swap → thrashing (OS spends most time swapping)              |
| **SIGSTOP signal**                           | Unix process paused by Ctrl-Z → resumes at SIGCONT; can be sent accidentally                                     |

```
All these can preempt thread at ANY POINT and resume LATER
Thread DOESN'T NOTICE the pause!

Similar to multithreaded code on single machine:
   Can't assume anything about timing

DIFFERENCE: Distributed system has NO SHARED MEMORY
   → Tools like mutex, semaphore DON'T TRANSLATE
   → Only messages over unreliable network

→ Node MUST ASSUME its execution can be paused for SIGNIFICANT TIME
   at ANY POINT, even in middle of a function
   During pause: rest of world keeps moving, may declare node dead
```

---

## 5. Response Time Guarantees (Hard Real-Time Systems)

### Hard Real-Time Systems

```
Some software: failure to respond within specified time = CATASTROPHE
   Aircraft, rockets, robots, cars: must respond quickly to sensor inputs
   → "Hard real-time" systems: must meet deadline or system fails

   Airbag example: sensor detects crash → airbag CANNOT be delayed
   by inopportune GC pause!
```

### Yêu cầu của Hard Real-Time

```
RTOS (Real-Time Operating System): guaranteed CPU allocation
Library functions: documented worst-case execution times
Dynamic memory allocation: restricted or disallowed
Testing: enormous amount to ensure guarantees met

→ Very expensive development
→ Severely restricts languages/libraries/tools
→ Most commonly used in safety-critical embedded devices

NOTE: "Real-time" ≠ "high-performance"
   Real-time systems may have LOWER THROUGHPUT (prioritize timeliness above all)

→ For most server-side data processing: NOT economical or appropriate
   → Must live with pauses and clock instability
```

### Limiting GC Impact

```
Modern GC algorithms: much improved — properly tuned ≈ few ms pauses
Java GC options: CMS, G1, ZGC, Epsilon, Shenandoah
Go: simpler concurrent mark-and-sweep

To avoid GC pauses entirely:
   Use language WITHOUT garbage collector:
   Swift: automatic reference counting
   Rust, Mojo: track object lifetimes via type system → compiler determines duration

Mitigating with GC language:
   Object pools (reuse rather than discard)
   Allocate data off-heap
   Treat GC pause as "brief planned outage":
      Stop new requests to node before GC
      Wait for outstanding requests to complete
      Perform GC with no requests in progress
      → Hides GC pauses from clients, reduces high percentiles

Restart processes periodically BEFORE accumulating long-lived objects
   → One node at a time, shift traffic first (like rolling upgrade)
```

---

## Tóm tắt phần này

```
TWO KINDS OF CLOCKS:
   Time-of-day (wall clock): can jump backward, not for measuring duration
   Monotonic: always moves forward, good for timeouts/intervals, NOT for cross-node ordering

CLOCK SYNC ACCURACY:
   NTP: limited by network delay (~35ms best via internet, spikes >100ms)
   Quartz drift: 200 PPM Google's assumption → 17s/day if only daily sync
   VM pauses, leap seconds, wrong NTP servers: additional problems

LWW WITH REAL-TIME CLOCKS = DANGEROUS:
   Silent data loss (lagging clock can't overwrite values from fast-clock node)
   Cannot distinguish sequential vs concurrent writes
   → Use LOGICAL CLOCKS instead (Chapter 10)

SPANNER TRUETIME:
   Explicit confidence interval [earliest, latest]
   Wait for interval length before commit → ensures non-overlapping intervals
   GPS/atomic clock per datacenter → ~7ms accuracy

PROCESS PAUSES:
   8 causes: GC, VM suspend, OS context switch, disk I/O, paging, SIGSTOP,...
   Thread doesn't know it was paused
   Can happen at ANY POINT, for ARBITRARY DURATION

HARD REAL-TIME: possible but very expensive → not for typical web services
GC mitigation: object pools, off-heap, treat GC as planned outage, periodic restart
```
