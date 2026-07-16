# Unreliable Networks

---

## 1. Bối cảnh: Shared-Nothing Systems

```
Distributed systems mà sách này tập trung:
   SHARED-NOTHING systems = machines connected by network

Mỗi machine: CPU + Memory + Disk RIÊNG
Machine KHÔNG thể access memory/disk của machine khác
   (trừ khi gửi request qua network)

Ngay cả object storage (S3, GCS): communicate qua network
```

---

## 2. Bản chất của Asynchronous Packet Networks

```
Internet và internal datacenter networks (Ethernet):
   ASYNCHRONOUS PACKET NETWORKS

Một node gửi message tới node khác:
   Network KHÔNG guarantee: khi nào đến, có đến không, arrive in order không

8 điều có thể xảy ra khi gửi request (Figure 9-1):
   1. Request bị LOST (network cable bị rút)
   2. Request đang WAITING IN QUEUE (network/recipient overloaded)
   3. Remote node FAILED (crash, powered down)
   4. Remote node tạm thời STOPPED RESPONDING (GC pause) → sẽ respond sau
   5. Remote node đã PROCESS request, nhưng RESPONSE bị LOST
   6. Remote node đã PROCESS request, RESPONSE bị DELAYED → đến sau
```

```
CRITICAL INSIGHT:
Sender KHÔNG THỂ phân biệt được các trường hợp này!
   (a) Request bị lost
   (b) Remote node down
   (c) Response bị lost

→ Tất cả đều TRÔNG GIỐNG NHAU: không nhận được response

→ ONLY option: TIMEOUT
   Nhưng khi timeout: vẫn KHÔNG BIẾT request có được processed không
   (nếu request đang trong queue, có thể vẫn được deliver sau khi giveup!)
```

---

## 3. Giới hạn của TCP

### TCP "Reliability" — Hiểu đúng

```
TCP: "Reliable" delivery nghĩa là:
   ✓ Detect và retransmit dropped packets
   ✓ Detect reordered packets, put back in order
   ✓ Detect packet corruption (simple checksum)
   ✓ Congestion control, flow control, backpressure

NHƯNG TCP KHÔNG có nghĩa là:
   ✗ Không lo về network being unreliable

TCP vẫn KHÔNG ĐẢM BẢO:
   - Biết packet có bị lost không nếu không nhận ack:
     (có thể outbound packet bị lost, HOẶC ack bị lost)
   - Guarantee new packet sẽ reach (nếu cable unplugged)
   - Biết bao nhiêu data đã được processed nếu connection closes with error

   TCP ack chỉ mean: OS kernel của remote node đã RECEIVE packet
   → Application có thể đã CRASH TRƯỚC KHI HANDLE data đó
```

### TCP cho large messages

```
Network packets: maximum size (vài KB)
TCP: break large messages thành packets, reassemble at receiver
→ Useful, nhưng không giải quyết underlying reliability issues
```

---

## 4. Network Faults trong Thực tế

### Số liệu từ nghiên cứu

```
Medium-sized datacenter: ~12 network faults/month
   - Half: disconnect SINGLE MACHINE
   - Half: disconnect ENTIRE RACK

Redundant networking gear: KHÔNG giảm faults như mong đợi
   (vì không guard against human error: misconfigured switches)

Cloud regions: round-trip times lên đến NHIỀU PHÚT ở high percentiles!
Within single datacenter: packet delay >1 PHÚT trong network reconfiguration

Asymmetric faults: A↔B OK, B↔C OK, nhưng A↔C FAIL
   (hoặc: interface drops ALL INBOUND packets but sends outbound fine)
```

### Những tác nhân gây đứt cáp submarine

```
- Cows biting through fiber links (!)
- Beavers chewing through cables (!!)
- Sharks attacking submarine cables (!!!)
- Scavenging humans stealing copper cables
- Accidental misconfiguration
- Sabotage (subsea cables)
```

### Nguyên tắc quan trọng

```
Nếu error handling của network faults KHÔNG ĐƯỢC TEST:
   Có thể xảy ra arbitrary bad things:
   - Cluster deadlock (permanently unable to serve requests)
   - Delete all data (!!)

→ Cần test error handling, kể cả chỉ để show error message đến users

Tolerating network faults KHÔNG phải là bắt buộc
   → Nhưng PHẢI BIẾT software reacts như thế nào và can recover
   → Deliberately trigger network problems + test (chaos engineering)
```

---

## 5. Fault Detection

### Khi nào có explicit feedback

```
Một số trường hợp CÓ thể biết:
   ✓ Port không có process listening → OS gửi RST/FIN packet
   ✓ Process crashed → management script notify other nodes (HBase)
   ✓ Datacenter management interface → query for link failures
   ✓ Router biết IP unreachable → ICMP Destination Unreachable
```

### Thường không thể biết

```
Hầu hết trường hợp:
   → Chỉ có thể retry vài lần, chờ timeout
   → Eventually declare node dead nếu không nhận response

   Vì node COULD BE alive (chỉ slow):
   → Balance between: false positives vs false negatives
   Too short timeout: alive nodes wrongly declared dead
   Too long timeout: slow recovery
```

---

## 6. Timeouts và Unbounded Delays

### Không có "đúng" giá trị cho timeout

```
Long timeout: long wait until node declared dead
   Users wait, see error messages

Short timeout: detect faults faster
   BUT: risk incorrectly declaring dead (load spike, network glitch)
   → Responsibilities transferred → more load on other nodes + network
   → If system already struggling: declaring node dead prematurely
     → CASCADING FAILURE
   (extreme case: all nodes declare each other dead → everything stops)

→ No simple answer
```

### Lý thuyết: Nếu có bounded delay

```
Fictitious system:
   Mọi packet delivered trong time d HOẶC LOST
   Node luôn handle request trong time r
   → Timeout safe to use: 2d + r

THỰC TẾ: Hầu hết systems KHÔNG có những guarantees này
   Asynchronous networks: UNBOUNDED delays
   Server implementations: KHÔNG GUARANTEE bounded response time
```

### Nguyên nhân của Variable Delays — Queueing

```
Network delay variability = CHỦ YẾU DO QUEUEING:

   1. Network switch queue: nhiều nodes gửi đến cùng destination
      → Switch must queue, feed one-by-one into network link
      If queue fills: packet DROPPED → phải resend

   2. OS receive queue: CPU cores busy → incoming requests queued
      → Có thể wait arbitrary amount of time

   3. Virtualization: OS paused while another VM uses CPU
      → Incoming data queued by VM monitor

   4. TCP congestion control: limits send rate → queuing at sender
      + automatic retransmit of lost packets → delay (không transparent to app)
```

### Multitenancy và Variable Delays

```
Public clouds và multitenant datacenters: resources SHARED

Network links, switches, network interface, CPUs: ALL SHARED
"Noisy neighbor" using lots of resources → high network delays for you

→ Timeouts chỉ có thể được chọn EXPERIMENTALLY:
   Measure distribution of round-trip times over extended period
   Determine expected variability
   Choose trade-off: failure detection delay vs risk of premature timeout

   Better: Auto-adjust timeouts based on observed response time distribution
   Phi Accrual failure detector (Akka, Cassandra): does this
```

---

## 7. Synchronous vs Asynchronous Networks

### Telephone Networks — Synchronous

```
Telephone circuit: FIXED, GUARANTEED bandwidth reserved for call
Along entire route between callers

ISDN: fixed 4,000 frames/second → call gets 16 bits/frame
→ Each side guaranteed to send exactly 16 bits every 250 microseconds

→ SYNCHRONOUS network: no queueing → BOUNDED DELAY (no unbounded delay)
```

### Internet/Datacenter — Asynchronous (Packet-Switched)

```
TCP connection: opportunistically use WHATEVER BANDWIDTH available
   Variable-sized data, minimize transfer time
   Idle connection: uses NO bandwidth

→ ASYNCHRONOUS: UNBOUNDED delays

KHÔNG CÓ concept "circuit" → no guaranteed bandwidth
→ Queueing delays happen → CANNOT guarantee bounded round-trip time
```

### Tại sao không làm Internet như Telephone?

```
Internet optimized for BURSTY TRAFFIC:
   Web page, email, file transfer: KHÔNG có particular bandwidth requirement
   → Just want it to complete as quickly as possible

Circuit-switched cho bursty traffic:
   Must GUESS bandwidth allocation trước
   Too low → unnecessarily slow
   Too high → cannot set up circuit

TCP: DYNAMICALLY adapts transfer rate → more efficient
→ Variable delays = consequence of COST/BENEFIT tradeoff
   Not a law of nature!

Latency guarantees achievable với static resource partitioning:
   BUT: more expensive, lower utilization
   Multitenancy + dynamic partitioning: better utilization, cheaper, but variable delays
```

---

## Tóm tắt phần này

```
UNRELIABLE NETWORKS:
   8 possible outcomes của 1 request/response — all look the same to sender
   Sender CANNOT tell why no response received
   Only option: TIMEOUT (but still don't know if request was processed!)

TCP "reliability" ≠ reliable distributed communication:
   Acks only mean OS received, not that application processed
   App crash after ack → data lost without knowing

NETWORK FAULTS IN PRACTICE:
   ~12 faults/month in medium datacenter
   Cloud: round-trips up to minutes at high percentiles
   Sharks, beavers, cows, humans: cause cable cuts
   → Must design for failure handling

TIMEOUTS — KHÔNG CÓ đúng giá trị:
   Too short → cascading failure risk
   Too long → slow recovery
   Unbounded delays → timeouts can only be set experimentally

VARIABLE DELAYS: Queueing (switch, OS, VM, TCP) + multitenancy
   Not law of nature → static partitioning can give bounded delays
   BUT: more expensive, lower utilization
```
