# Processing Streams: Operators, Windows, và Time

---

## 1. Stream Processing Operators

### Cơ bản: 1 Event tại 1 Thời điểm

```
STREAM PROCESSOR:
   Reads events from INPUT STREAM
   Applies some TRANSFORMATION
   Writes results to OUTPUT STREAM
   (OR: writes to database / sends emails / triggers external service)

CONTINUOUS:
   Never finishes (input stream is unbounded)
   Always listening for new events
```

### Loại Operators Cơ bản

| Operator          | Mô tả                              | Ví dụ                                           |
| ----------------- | ---------------------------------- | ----------------------------------------------- |
| **Filter**        | Keep/discard events by condition   | Keep only errors; discard bot traffic           |
| **Map/Transform** | Transform each event               | Parse JSON; extract fields; currency conversion |
| **FlatMap**       | 1 event → 0..N output events       | Tokenize sentence → words                       |
| **Aggregate**     | Combine multiple events → 1 output | Count, sum, min, max                            |
| **Join**          | Combine events from 2 streams      | User event + profile data                       |
| **Window**        | Group events by time interval      | 5-min rolling average                           |

### Stateless vs Stateful Processing

```
STATELESS OPERATORS (easy):
   Each event processed INDEPENDENTLY
   No memory of previous events
   Examples: filter, map, transform
   Fault tolerance: simple (just reprocess failed events)

STATEFUL OPERATORS (hard):
   Need to REMEMBER PAST EVENTS to compute result
   Examples: aggregations, joins, windowed computations
   State: typically stored in LOCAL STORE on stream processor node

   CHALLENGE: What happens if node crashes?
   → Must PERSIST state or replay events (fault tolerance)
```

---

## 2. Time in Stream Processing

### Vấn đề: Hai Loại Thời gian

```
EVENT TIME (when event ACTUALLY HAPPENED):
   Timestamp in the event itself
   Set by producer (device clock, server clock when event created)

   Problems:
   - Device clock may be wrong (Chapter 9: unreliable clocks)
   - Clock skew between devices

PROCESSING TIME (when event ARRIVES at stream processor):
   Processor's local clock when event is processed

   Problems:
   - Network delays: events arrive AFTER they happened
   - Backlog: if processing is slow, events may be hours behind
   - Restart: events replayed from old position → processing time ≠ event time
```

### Event Time vs Processing Time — Khi nào khác nhau?

```
Normal case:
   Event time ≈ Processing time (small, predictable lag)

Problematic cases:
   Mobile device OFFLINE → buffers events → comes online → sends all at once
      → Processing time = now, Event time = hours/days ago

   Slow consumer FALLS BEHIND → processing events from yesterday
      → Processing time = now, Event time = yesterday

   BUGS in producer → events arrive OUT OF ORDER
   NETWORK DELAYS → packet reordering

→ For business-correct analytics: MUST use EVENT TIME
   (not "how many events processed this hour" but "how many events HAPPENED this hour")
```

### Late Arrivals

```
PROBLEM: Events can arrive ARBITRARILY LATE
   Mobile offline for 3 days → events arrive 3 days late

If using event time windowing:
   Window for "9am-10am" closed → event arrives at 3pm for 9:30am
   → Include it in 9am window? (reopen old window?)
   → Drop it? (lose data)
   → Update downstream materialized view? (complexity)

No perfect answer → need POLICY for late events:
   1. IGNORE late events beyond threshold (simplest)
   2. INCLUDE in window but mark as LATE (more complex)
   3. EMIT CORRECTION when late event arrives (most accurate)
```

---

## 3. Windowing

### Tại sao cần Windows?

```
AGGREGATION over all time: usually not what we want
   "Total clicks ever" is less useful than "Clicks in last hour"

WINDOWING: group events into FINITE TIME INTERVALS for aggregation
   Allows: rolling averages, hourly counts, daily totals
```

### 4 Loại Windows

#### Tumbling Window (Non-overlapping)

```
Fixed-size windows, ADJACENT (back to back, no overlap)
Events belong to EXACTLY ONE window

Timeline: ─────|─────|─────|─────|─────▶
          12:00 12:05 12:10 12:15 12:20

Each window: exactly 5 minutes, computed independently

Use case: "Hourly sales totals", "Daily page views"
Result: 1 output per window, non-overlapping

Implementation:
   Group events by: timestamp // window_size
   Same window = same bucket
```

#### Hopping Window (Fixed-size, overlapping)

```
Fixed-size windows with SMALLER HOP INTERVAL
Windows OVERLAP

Window size: 10 minutes
Hop interval: 5 minutes

Windows:
   12:00-12:10
   12:05-12:15
   12:10-12:20
   12:15-12:25

Each event: belongs to MULTIPLE windows

Use case: "Moving average over last 10 minutes, updated every 5 minutes"
Result: smoother than tumbling windows
```

#### Sliding Window

```
Window of fixed size, CONTINUOUSLY SLIDING
Every NEW event arrives → new window computed
("last N minutes from NOW")

Use case: "Click rate in last 5 minutes, updated on every click"
More granular than hopping

Implementation: more complex than tumbling/hopping
   Often implemented as hopping with very small hop size
```

#### Session Window

```
VARIABLE SIZE: groups events from SAME USER into SESSIONS
Session = no gap longer than TIMEOUT between events

User clicks at: 12:01, 12:02, 12:05, 12:20, 12:25
Timeout: 10 minutes

Sessions:
   Session 1: 12:01-12:05 (gap of 15min before 12:20)
   Session 2: 12:20-12:25

Use case: "User session analysis", "Time spent on page"
Size: unknown ahead of time (depends on user behavior)
```

---

## 4. Watermarks — Handling Late Data

### Problem: When is Window "Complete"?

```
With event time windowing:
   Window 12:00-12:05 → when can we EMIT RESULT?

   If we emit at 12:05 (processing time):
   → Events from mobile that happened at 12:03 but arrive at 12:10 → MISSED

   If we wait for ALL events → wait forever (events may be arbitrarily late)

→ Need some notion of "MOST EVENTS FOR THIS WINDOW HAVE ARRIVED"
```

### Watermark Definition

```
WATERMARK: claim that "all events with timestamp ≤ T have been received"
   → Window [start, T] is now COMPLETE → safe to emit result

Where does watermark come from?
   HEURISTIC: look at event timestamps being received
   If latest event = 12:15, assume all events for 12:05 window received
   (assuming max 10 minutes of lateness)

WATERMARK GENERATION:
   Source (input stream) generates watermarks
   Stream processor PROPAGATES watermarks through DAG of operators
   Window operator: emits window result when watermark PASSES window end
```

### Perfect vs Heuristic Watermarks

```
PERFECT WATERMARKS:
   Know exactly when all events for a time range have arrived
   Possible if: know exactly what events exist and when they'll arrive
   Rare in practice

HEURISTIC WATERMARKS:
   Estimate based on observed data
   "Probably all events from 10+ minutes ago have arrived"
   May be wrong → trade-off:

   TOO EARLY (aggressive):
      Some late events missed
      Low latency (results emitted quickly)

   TOO LATE (conservative):
      Fewer missed events
      High latency (wait a long time before emitting)
```

### Handling Late Events AFTER Watermark

```
Even with watermark: some events arrive AFTER watermark has passed
Three strategies:

1. DROP: Ignore late events
   Simple, some data loss

2. INCLUDE AND RECOMPUTE: Update result when late event arrives
   Accuracy maintained
   Complexity: downstream must handle CORRECTIONS
   Kafka Streams + Flink: support this

3. TRIGGER AND ACCUMULATE:
   Emit EARLY RESULT (speculative) as events arrive
   Emit FINAL RESULT when watermark passes
   Emit CORRECTIONS for late events
   Apache Beam: "triggers" system for this
```

---

## 5. Stream Analytics Use Cases

### Real-Time Metrics

```
COMMON USE CASES:
   Request rate per service per second
   Error rate, p99 latency per endpoint
   Active users per minute
   Revenue per hour

APPROACH:
   Event stream (logs, metrics) → stream processor → time-series database
   Dashboard reads from time-series DB

Tools:
   Apache Flink, Spark Structured Streaming, Kafka Streams
   Time-series DBs: InfluxDB, Prometheus, TimescaleDB
```

### Search over Streams (Event-Triggered Queries)

```
PERCOLATOR PATTERN (Elasticsearch):
   Traditional search: store documents, query them
   Stream search: "reverse" — store QUERIES, search them when new doc arrives

   User defines: "Alert me when new article matches: machine learning AND Python"
   → Query stored in system
   → When new article arrives → check all stored queries → notify matching users

Elasticsearch: "Percolate API"
Apache Storm: "DRPC" (Distributed Remote Procedure Call)
```

---

## Tóm tắt phần này

```
STREAM OPERATORS:
   Stateless (filter, map): process each event independently
   Stateful (aggregate, join, window): need state across events
   Fault tolerance for stateful = harder

TIME IN STREAM PROCESSING:
   Event time: when event HAPPENED (timestamp in event)
   Processing time: when event ARRIVES at processor
   Usually different due to: network delays, mobile offline, slow consumers
   Analytics: use EVENT TIME for correct business metrics

4 WINDOW TYPES:
   Tumbling: fixed-size, non-overlapping, each event in 1 window
   Hopping: fixed-size, overlapping, each event in N windows
   Sliding: continuous, window follows most recent event
   Session: variable-size, grouped by user activity gap

WATERMARKS:
   Claim: "all events with timestamp ≤ T received" → window complete
   Perfect (rare) vs Heuristic (common trade-off: latency vs accuracy)
   Late events: drop / recompute / trigger-and-accumulate
```
