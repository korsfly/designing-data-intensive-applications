# Online vs Offline Systems và Unix Tools

---

## 1. Ba Loại Hệ thống theo Cách Xử lý Data

```
1. ONLINE (Request/Response) Systems:
   User gửi request → system phản hồi NHANH
   Primary metric: RESPONSE TIME, AVAILABILITY
   Examples: web servers, databases, caches, APIs

2. BATCH PROCESSING (Offline) Systems:
   Chạy computation lớn trên large dataset
   Input data: READ-ONLY (immutable)
   Output: GENERATED FROM SCRATCH mỗi lần chạy
   Primary metric: THROUGHPUT (data processed per unit time)
   Chạy định kỳ: daily, weekly
   Examples: train AI model, ETL pipelines, log analytics

3. STREAM PROCESSING (Near-real-time):
   Xử lý unbounded stream of events
   Primary metric: LATENCY + THROUGHPUT
   Examples: fraud detection, real-time dashboards
   → Chapter 12
```

---

## 2. Đặc điểm của Batch Processing

### Immutable Input

```
CORE PROPERTY: Input data không bao giờ bị modify
   Batch job chỉ READ, không WRITE to input files

BENEFITS:
   1. HUMAN FAULT TOLERANCE:
      Bug trong code → delete output, fix, rerun → correct output
      (OLTP database: rollback code không fix data đã sai)

   2. TIME TRAVEL:
      Object stores, open table formats (Delta Lake, Iceberg):
      Query data as of specific past point
      Restore to earlier state nếu cần

   3. AGILITY: Minimizing irreversibility → safer to move faster
   4. REUSE: Same input → multiple different job types
   5. EFFICIENCY: Batch frameworks optimize bulk reads
```

### Job Execution Pattern

```
TYPICAL BATCH JOB CYCLE:
   Schedule trigger (cron job, workflow scheduler)
        ↓
   Read input data (DFS/object store)
        ↓
   Process data (compute transformation)
        ↓
   Write output (new DFS/object store files)
        ↓
   Mark job COMPLETE

Duration: minutes to hours to DAYS (for very large datasets)
Fault handling: restart failed tasks (not entire job)
Boundary: Long-running DB analytical query ≈ batch process
```

---

## 3. Unix Pipeline — Nền tảng Tư duy

### Log Analysis với Unix Tools

```
Bài toán: Tìm 5 URLs phổ biến nhất trong access log của web server

cat /var/log/nginx/access.log |
  awk '{print $7}' |        # extract URL (field 7)
  sort |                     # sort alphabetically
  uniq -c |                  # count consecutive duplicates
  sort -r -n |               # sort by count descending
  head -n 5                  # take top 5
```

**Output:**

```
4189 /favicon.ico
3631 /2016/02/08/how-to-do-distributed-locking.html
2492 /2016/04/02/decoding-protobuf.html
1987 /css/tufte.css
1375 /
```

### Tại sao pipeline này hiệu quả?

```
Từng command trong pipeline:
   STDIN → process → STDOUT

DATA FLOWS THROUGH PIPELINE as stream:
   Không cần load toàn bộ file vào memory
   Commands run CONCURRENTLY (parallel stages in pipeline)

GNU sort: xử lý files LARGER THAN MEMORY
   External merge sort (Mergesort trên disk)
   Automatically parallel across CPU cores

→ Đây là BỘ TỨ cốt lõi: awk | sort | uniq | sort
```

### Unix Philosophy

```
4 TENETS của Unix Philosophy:
   1. Do ONE THING and do it well
   2. Programs work together (via stdin/stdout)
   3. Handle TEXT STREAMS (uniform interface)
   4. Loose coupling (don't assume anything about other programs)

→ Composability: nhỏ, độc lập, kết nối qua pipes

Distributed batch processing frameworks =
   DISTRIBUTED VERSION of Unix philosophy:
   - Immutable files = stdin/stdout equivalents
   - Each processing stage = one Unix command
   - Data flows between stages = pipes
```

---

## 4. Sort-based vs Hash-based Processing

### Sort-based Approach (Unix pipeline dùng sort)

```
Algorithm:
   1. Emit all (URL, 1) pairs
   2. SORT tất cả pairs by URL
   3. Now all same-URL pairs are adjacent
   4. Scan through sorted list, count consecutive same keys

ADVANTAGE:
   External merge sort: works with datasets MUCH LARGER than memory
   Predictable memory usage (fixed buffer sizes)
   Natural for range queries

GNU sort: production-grade external sort
   --parallel flag: use multiple CPU cores
   -T flag: specify temp directory for spill files
   Estimated 1 GB/s throughput on modern hardware
```

### Hash-based Approach (Custom Python program)

```python
# In-memory hash table approach
from collections import defaultdict

url_counts = defaultdict(int)
for line in sys.stdin:
    url = line.split()[6]
    url_counts[url] += 1

# Sort by count descending, take top 5
for url, count in sorted(url_counts.items(),
                          key=lambda x: x[1], reverse=True)[:5]:
    print(f"{count} {url}")
```

```
ADVANTAGE: FASTER if working set fits in memory
   (avoid disk I/O of external sort)

LIMITATION: Working set = number of DISTINCT URLs
   If more distinct URLs than memory → out of memory crash

WHEN TO USE:
   ✓ Distinct values fit comfortably in memory
   ✓ Need fast lookups during processing
   ✗ Dataset too large → fallback to sort-based
```

### Distributed Equivalent

| Unix Tool                    | Batch Framework Equivalent            |
| ---------------------------- | ------------------------------------- |
| `cat file`                   | Read from DFS/object store            |
| `awk '{print $7}'`           | Mapper function                       |
| `sort`                       | Shuffle + sort phase                  |
| `uniq -c`                    | Reducer function                      |
| `\|` pipe                    | Network transfer of intermediate data |
| Multiple CPU cores in `sort` | Multiple parallel tasks               |

---

## 5. Giới hạn của Unix Tools

```
Unix tools chạy trên SINGLE MACHINE:
   Limited by: RAM, CPU cores, disk of that single machine

Dataset quá lớn cho 1 machine?
   → Need DISTRIBUTED batch processing

Same problem, same solution:
   Single machine: sort | uniq -c
   Distributed: Map (emit URL) → Shuffle → Reduce (count)

The LOGIC is the same, the EXECUTION is distributed
→ This insight drives MapReduce design
```

---

## Tóm tắt phần này

```
ONLINE vs BATCH:
   Online: response time, request-driven
   Batch: throughput, data-driven, immutable input

IMMUTABLE INPUT BENEFITS:
   Human fault tolerance (re-run after bug fix)
   Time travel (query historical state)
   Agility, reuse, efficiency

UNIX PIPELINE (awk | sort | uniq | sort | head):
   Composable, concurrent stages
   External merge sort: handles larger-than-memory
   Template for distributed batch processing

SORT-BASED vs HASH-BASED:
   Sort: larger-than-memory, predictable
   Hash: faster when working set fits in memory

UNIX → DISTRIBUTED:
   Same logic, different execution scale
   Unix pipe = MapReduce data flow
```
