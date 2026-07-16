# Messaging Systems cho Stream Processing

---

## 1. Stream vs Batch — Sự Khác biệt Cơ bản

```
BATCH (Chapter 11):
   Input: BOUNDED dataset (known, finite size)
   Job: biết khi nào kết thúc đọc input
   Latency: minutes → hours (sau khi toàn bộ input sẵn sàng)

STREAM:
   Input: UNBOUNDED — dữ liệu đến dần dần, KHÔNG BAO GIỜ "complete"
   Job: không bao giờ kết thúc (always more data coming)
   Latency: milliseconds → seconds (xử lý từng event khi đến)

Ví dụ thực tế:
   Users produced data yesterday, today, will produce MORE tomorrow
   → Dataset never "complete" → CANNOT wait for it to finish

Batch xử lý unbounded data:
   Artificially divide into FIXED INTERVALS (hourly, daily jobs)
   → Problem: changes reflected 1 hour/day LATER → too slow for many users

→ Solution: Process events as they happen = STREAM PROCESSING
```

---

## 2. Events và Topics

### Event là gì?

```
EVENT: small, self-contained, IMMUTABLE OBJECT
   Contains: details of something that happened at a point in time
   Usually contains: TIMESTAMP (when it happened, per time-of-day clock)

Examples:
   - User viewed a page
   - User made a purchase
   - Temperature sensor reading
   - CPU utilization metric
   - Line in web server log

Encoding: text string, JSON, binary (Protobuf, Avro)
   → Can store (append to file, insert to DB) or send over network

Batch equivalent: record in input file
Stream equivalent: event in stream
```

### Topics và Streams

```
BATCH: File = identified by filename, contains set of related records
STREAM: Topic/Stream = identified by name, contains group of related events

Producer (publisher/sender): generates event → publishes to topic
Consumer (subscriber/recipient): processes events from topic

MULTIPLE CONSUMERS:
   1 event → processed by MULTIPLE independent consumers
   (batch equivalent: several jobs reading same input file)
```

---

## 3. Messaging System Approaches

### Approach 1: Direct Messaging (No Intermediary)

```
Producers → directly send to Consumers (no broker in between)

Examples:
   UDP multicast: financial industry, stock market feeds (low latency)
                  UDP = unreliable, but application-level retransmit
   ZeroMQ, nanomsg: brokerless pub/sub over TCP or IP multicast
   StatsD: UDP metrics collection from all machines
   Webhooks: producer makes HTTP/RPC request directly to consumer

PROS:
   Very low latency
   Simple (no broker to manage)

CONS:
   Both producer AND consumer must be online simultaneously
   If consumer offline → misses messages (unless producer buffers and retries)
   If producer crashes → buffered retries lost
   Limited fault tolerance
   Application must handle message loss explicitly
```

### Approach 2: Message Brokers (Intermediary)

```
MESSAGE BROKER (Message Queue):
   Database optimized for handling message streams
   Runs as server; producers and consumers connect as clients

   Producer → writes to broker
   Broker → delivers to consumers

BENEFITS over direct messaging:
   Clients can come and go (connect, disconnect, crash)
   Durability: broker holds messages
   Decoupling: producer doesn't need to know about consumers
   Backpressure: producer can slow down when broker is full

DURABILITY:
   Some brokers: in-memory only (lose messages on crash)
   Others: write to disk (survive crashes, configurable)
```

### 2 Consumer Patterns (Figure 12-1)

```
LOAD BALANCING (Point-to-Point Queue):
   Each message delivered to ONE of the consumers
   Consumers SHARE the work of processing
   Useful: expensive messages, want to parallelize
   AMQP: multiple clients consuming from same queue
   JMS: called "shared subscription"

FAN-OUT (Publish-Subscribe Topic):
   Each message delivered to ALL consumers
   Consumers each "tune in" to same broadcast, independently
   Useful: multiple independent consumers for same event stream
   JMS: topic subscriptions; AMQP: exchange bindings

COMBINING: Kafka Consumer Groups:
   Messages in topic → ONE consumer in each group (load balance within group)
   Multiple groups → EACH gets copy of every message (fan-out across groups)
```

### Acknowledgments và Redelivery

```
Consumer may crash at any time during processing
MESSAGE BROKER: use ACKNOWLEDGMENTS
   Consumer explicitly tells broker: "finished processing this message"
   → Broker removes from queue

If no ack received (connection closed, timeout):
   Broker ASSUMES message not processed
   → Delivers message AGAIN to another consumer

POTENTIAL DUPLICATE: message processed but ack lost in network
   → Requires atomic commit protocol (Chapter 8)
   OR: idempotent operations

REDELIVERY + LOAD BALANCING → Message reordering (Figure 12-2):
   Consumer 2 crashes processing m3
   m3 redelivered to Consumer 1
   Consumer 1 now processes: m4, m3, m5 (not in order!)

→ If causal dependencies between messages: ordering matters!
   If messages independent: reordering not a problem
```

### Dead Letter Queues (DLQ)

```
PROBLEM: Malformed message → consumer crashes → broker redelivers → crash again
   → Infinite loop, blocks entire queue

DEAD LETTER QUEUE (DLQ):
   After N retry attempts → move message to separate DLQ
   → Unblocks main queue, consumers can continue

Monitoring on DLQ:
   Any message = error → alert operators
   Operator decides: drop permanently, manually fix and reproduce, fix consumer code

Traditional: JMS/AMQP-style brokers
More recently: Apache Pulsar, Kafka Streams now support DLQs
```

---

## 4. Log-Based Message Brokers (Kafka, Kinesis)

### Motivation: Best of Both Worlds

```
TRADITIONAL MESSAGE BROKERS (AMQP/JMS):
   Messages: TRANSIENT (deleted after delivered)
   Processing message: DESTRUCTIVE operation (acknowledgment = delete)
   New consumer: only receives messages sent AFTER registration
   Can't replay: prior messages already gone

FILES/DATABASES:
   Data: DURABLE (persists until explicitly deleted)
   New client: can read data written ARBITRARILY FAR IN THE PAST

→ WHY can't we have BOTH durable storage + low-latency notification?
   → That's the idea behind LOG-BASED MESSAGE BROKERS
```

### Log Structure (Figure 12-3)

```
LOG = append-only sequence of records on disk
   (same as Chapter 4 log-structured storage, Chapter 6 replication log)

Producer: APPENDS message to end of log
Consumer: READS log sequentially (like Unix tail -f)
   If at end: WAITS for new messages to be appended

SHARDING for throughput:
   Log sharded across multiple machines (Chapter 7)
   Each SHARD = separate log (read/write independently)
   TOPIC = group of shards carrying messages of same type

Within each shard (Kafka: PARTITION):
   Monotonically increasing OFFSET assigned to each message
   Messages within 1 partition: TOTALLY ORDERED
   No ordering guarantee ACROSS partitions

Apache Kafka, Amazon Kinesis Streams: log-based brokers
Google Cloud Pub/Sub: architecturally similar but JMS-style API
```

### Consumer Offsets

```
Consumer reads log sequentially → tracks CURRENT OFFSET

Messages processed: offset < current offset
Messages not yet seen: offset > current offset

BROKER: only records consumer offsets PERIODICALLY (not per-message)
   → Much lower bookkeeping overhead than AMQP (per-message acks)
   → Higher throughput

CONSUMER FAILURE → restart from LAST RECORDED OFFSET (not latest)
   → May process some messages TWICE (if offset not recorded after processing)
   → Handle with idempotency!

Compare with database replication:
   Message broker ≈ LEADER database
   Consumer ≈ FOLLOWER (replays log from offset = follower lag position)
```

### Log-Based vs Traditional Message Brokers

| Feature                | Traditional (AMQP/JMS)          | Log-Based (Kafka)                      |
| ---------------------- | ------------------------------- | -------------------------------------- |
| **Message durability** | Short-lived (deleted after ack) | Durable (kept for configurable period) |
| **Replay**             | ✗ Not possible                  | ✓ Rewind offset                        |
| **Fan-out**            | Via topic subscriptions         | ✓ Multiple independent readers         |
| **Load balancing**     | Per-message assignment          | Per-shard assignment                   |
| **Ordering**           | May lose order with LB          | Preserved within shard                 |
| **Throughput**         | Moderate                        | Very high (sequential disk write)      |
| **Slow consumers**     | Blocks or fills queue           | Only affects own offset                |
| **Adding consumer**    | Misses past messages            | ✓ Start from any offset                |

### Disk Space Usage và Tiered Storage

```
LOG GROWS → eventually runs out of disk space
Solution: DIVIDE log into segments, DELETE or ARCHIVE old segments
   → Implements BOUNDED BUFFER (circular/ring buffer on disk)

Capacity estimation:
   20 TB disk, 250 MB/s sequential write
   → ~22 hours before full at max write rate
   → In practice: SEVERAL DAYS TO WEEKS of buffer

Slow consumer falls behind → may miss deleted messages
   → Only that consumer affected (not others) ← big operational advantage
   → Monitor consumer lag; alert if too far behind

TIERED STORAGE (Kafka, Redpanda):
   Older messages → moved to OBJECT STORAGE (S3, GCS)
   Hot messages → served from local disk
   → Virtually unlimited retention
   → Messages stored as Iceberg tables → batch/warehouse jobs can read directly

WarpStream, Confluent Freight, Bufstream:
   Store ALL data in object store (Zero-Disk Architecture from Chapter 6)
```

### Replaying Old Messages

```
Log-based broker: consuming messages = READ-ONLY operation
   → DOESN'T CHANGE THE LOG (unlike AMQP acknowledgment)

Only side effect: consumer offset moves forward
   → Offset under CONSUMER'S CONTROL

REPLAY USE CASES:
   Reprocess last day: start consumer with yesterday's offset → different output location
   Debug production: consume production log for testing without disrupting others
   Add new consumer: start from offset 0 → get FULL HISTORY

→ More like BATCH PROCESSING than traditional messaging:
   Derived data clearly separated from input
   Repeatable transformation process
   Easier recovery from errors and bugs
```

---

## Tóm tắt phần này

```
STREAM vs BATCH:
   Stream: unbounded input, continuous processing, low latency

2 MESSAGING APPROACHES:
   Direct (UDP, ZeroMQ, webhooks): fast but limited fault tolerance
   Message Brokers: durable, decoupled, handles slow consumers

2 CONSUMER PATTERNS:
   Load balancing: distribute work, may reorder messages
   Fan-out: all consumers get all messages, independent

TRADITIONAL (AMQP/JMS):
   Transient, delete after ack, per-message tracking
   DLQs for handling malformed messages

LOG-BASED (Kafka, Kinesis):
   Durable, replayable, offset-based tracking
   Fan-out trivial (multiple independent readers)
   Load balance by assigning whole shards
   Tiered storage: extend to object store (virtually unlimited)

KEY ADVANTAGE: Replay old messages → experimentation, reprocessing
```
