# Processing Models: MapReduce, Spark, Flink

---

## 1. MapReduce — Nền tảng Lịch sử

### 4 Bước của MapReduce

```
Inspired by Google's MapReduce paper (2004)
Apache Hadoop: open-source implementation

STEP 1: READ INPUT
   Break input into records
   Each mapper gets a chunk of input (typically 1 HDFS block = 128MB)

STEP 2: MAP (Transform)
   Mapper function called once per input RECORD
   Stateless: no state between records
   Emits (key, value) pairs
   Can emit 0, 1, or multiple pairs per record

STEP 3: SORT & SHUFFLE (Implicit, automatic)
   All (key, value) pairs SORTED BY KEY
   Pairs with same key → same reducer
   Happens over the network (data moves from mappers to reducers)
   This is the MOST EXPENSIVE STEP (network I/O)

STEP 4: REDUCE (Aggregate)
   Reducer receives: (key, [list of values])
   All values sorted, all for same key
   Aggregates, computes, emits output records
   Output written to DFS
```

### Ví dụ: Word Count

```python
# MAPPER (called once per line of text)
def mapper(line):
    for word in line.split():
        emit(word, 1)

# After shuffle: ("hello", [1, 1, 1]), ("world", [1, 1])

# REDUCER (called once per unique key)
def reducer(word, counts):
    emit(word, sum(counts))
```

### Ví dụ: Web Log URL Analysis

```python
# MAPPER: extract URL from log line
def mapper(log_line):
    url = log_line.split()[6]  # field 7 = URL
    emit(url, 1)               # emit (url, 1) pair

# SORT: all (url, 1) pairs sorted by url
# → all same URLs now adjacent

# REDUCER: count occurrences
def reducer(url, counts):
    emit(url, sum(counts))     # total count per URL
```

### Combiner — Pre-aggregation Optimization

```
PROBLEM: After map, ship ALL (key, value) pairs over network
   Millions of ("hello", 1) pairs from one mapper
   → Huge network transfer for common words!

COMBINER: Run reducer-like code AFTER mapper, BEFORE network transfer
   Like a "local reducer" on same machine as mapper

Example:
   Mapper emits: ("hello",1), ("hello",1), ("world",1), ("hello",1)
   Combiner: ("hello", 3), ("world", 1)  ← fewer bytes to transfer

   (Much less network data for common keys!)

REQUIREMENT: Combiner must be safe to apply multiple times
   (same as reducer: associative and commutative)
   SUM: safe (can partial-sum at mapper, then final-sum at reducer)
   AVERAGE: NOT safe as combiner! (average of averages ≠ global average)
             → Instead: combiner emits (sum, count), reducer computes avg
```

### MapReduce в Functional Programming

```
MapReduce mirrors FUNCTIONAL PROGRAMMING:
   Mapper: pure function (no side effects)
   Reducer: pure function
   Immutable inputs

Benefits:
   DETERMINISTIC (re-run gives same result)
   Easy to TEST (pure functions)
   Easy to REASON about data flow
   Human FAULT TOLERANCE (delete output, re-run)

→ Same benefits as immutable input discussed earlier
```

### MapReduce Limitations

```
LIMITATIONS that led to Spark/Flink:

1. ALWAYS WRITES TO DFS between stages:
   - Each MapReduce stage: write output to HDFS
   - Next stage: read from HDFS
   - Fault tolerance reason: survives node failure
   - PROBLEM: HDFS reads/writes = very expensive I/O
   - 3-4 stage pipeline → 3-4x DFS writes beyond just final output

2. ONLY 2 STAGES (map + reduce):
   - Complex computation: chain MULTIPLE MapReduce jobs
   - Each chained job: MORE DFS writes
   - Hard to express iterative algorithms (ML, graph algorithms)

3. SLOW for iterative algorithms:
   - PageRank: need 30+ iterations
   - Each iteration: separate MapReduce job
   - Each job: 2 DFS reads + 2 DFS writes
   - 30 iterations × 4 DFS I/Os = 120 DFS operations!

4. ALWAYS SORTS intermediate data:
   - Sorting always happens (even if not needed)
   - Sort = expensive
```

---

## 2. Spark — In-Memory Dataflow Engine

### Core Innovation: RDDs

```
RDD (Resilient Distributed Dataset):
   Distributed collection of records across many nodes
   IMMUTABLE (transformations create new RDDs)
   LAZY EVALUATION (not computed until action called)
   LINEAGE tracking (knows how it was derived)

Key concepts:
   TRANSFORMATION: Returns new RDD (lazy)
      map(), filter(), groupByKey(), join(), flatMap(),...

   ACTION: Triggers computation, returns result or writes output
      collect(), count(), save(), take(), reduce(),...
```

### RDD Lineage и Fault Tolerance

```
Each RDD knows its PARENT RDDs and how it was derived:

   inputRDD = sc.textFile("logs.txt")
        ↓ .map(parse_line)
   parsedRDD
        ↓ .filter(is_error)
   errorsRDD
        ↓ .map(extract_message)
   messagesRDD

LINEAGE GRAPH: DAG of transformations from input to output

ON FAILURE (node crash, data lost):
   DON'T restart entire job
   Use lineage to RECOMPUTE only the lost partitions
   → Start from last checkpoint OR from input
   → Efficient when recomputation is cheap

Vs MapReduce: recompute vs re-read from DFS checkpoint
   Spark: faster recompute if transformation is cheap
   Spark: slower if many stages need recomputation
```

### Lazy Evaluation — Query Optimization

```
Spark does NOT execute immediately:
   rdd.map(f1).filter(f2).groupBy(key).agg(sum)

   → Builds a LOGICAL PLAN
   → Applies OPTIMIZATIONS:
      - Push filter before groupBy (reduce data early)
      - Combine map + filter into single pass
      - Choose join algorithm based on data size
   → PHYSICAL PLAN: actual execution strategy

Action call (.count(), .save(),...) triggers execution

Benefits:
   Optimization opportunities
   Avoid computing intermediates that are thrown away
   Better resource utilization
```

### Spark DataFrames and SQL

```
Low-level: RDD API (flexible but verbose)
High-level: DATAFRAME API (like SQL table)

DataFrame operations:
   df.select("col1", "col2")           # SELECT
   df.filter(df.age > 18)              # WHERE
   df.groupBy("country").agg(...)      # GROUP BY
   df.join(other_df, on="user_id")     # JOIN
   df.orderBy("revenue", desc=True)    # ORDER BY
   df.limit(100)                       # LIMIT

SPARK SQL:
   spark.sql("SELECT country, SUM(revenue) FROM sales GROUP BY country")
   → Returns DataFrame
   → Same optimizer (Catalyst) as DataFrame API

CATALYST OPTIMIZER:
   Analyzes logical plan → applies rules → physical plan
   Rule examples:
   - Constant folding: WHERE 1=1 → remove filter
   - Predicate pushdown: filter before join
   - Column pruning: only read needed columns

TUNGSTEN ENGINE:
   Code generation: compile plan to Java bytecode
   Off-heap memory: avoid Java GC overhead
   Cache-aware data structures: better CPU cache utilization
```

---

## 3. Flink — Stream-Native with Batch Support

### Flink Philosophy

```
Apache Flink: designed as STREAMING ENGINE
   Batch processing = BOUNDED STREAM processing

Natively handles: BOTH bounded (batch) and unbounded (streaming)
   Same API, same engine, different data sources

Key difference from Spark:
   Spark: originally batch, streaming added later
   Flink: streaming-native, batch is special case

→ For unified batch + stream workloads: Flink is strong
```

### Flink Fault Tolerance: Checkpointing

```
CHECKPOINTING: Periodically save state snapshots

For batch:
   Long-running job
   Checkpoint every N minutes
   On failure: restart from last checkpoint (not from beginning)

For streaming (Chapter 12):
   Save state of ALL operators at consistent global point
   Chandy-Lamport distributed snapshot algorithm
   Recovery: rollback to last checkpoint, replay events from source

ADVANTAGES over Spark lineage:
   Predictable recovery time (bounded to checkpoint interval)
   Good for long-running jobs with expensive recomputation

DISADVANTAGES:
   Checkpoint overhead (write to DFS periodically)
   Recovery means re-running events since last checkpoint
```

---

## 4. Programming APIs — Low-Level to High-Level

### API Pyramid

```
                  ┌─────────────┐
                  │  GUI Tools  │  ← Tableau, Looker, Power BI
                  └──────┬──────┘
                  ┌──────┴──────┐
                  │    SQL      │  ← BigQuery, Snowflake, Spark SQL
                  └──────┬──────┘
                  ┌──────┴──────┐
                  │  DataFrame  │  ← PySpark, Pandas, Flink Table API
                  └──────┬──────┘
                  ┌──────┴──────┐
                  │   RDD/Low   │  ← Spark RDD, Flink DataStream
                  └──────┬──────┘
                         │
                    Business Analysts
                    Analytics Engineers ←──────── each level
                    Application Engineers ◄─────── serves different
                    Data Scientists      ←─────── users
```

### DataFrame API

```python
# PySpark DataFrame example
from pyspark.sql import SparkSession
from pyspark.sql.functions import col, sum, desc

spark = SparkSession.builder.getOrCreate()

df = spark.read.parquet("s3://mybucket/sales/")

result = (df
    .filter(col("year") == 2024)          # WHERE year = 2024
    .groupBy("country", "product_category") # GROUP BY
    .agg(sum("revenue").alias("total_revenue"),  # SUM(revenue)
         sum("quantity").alias("total_quantity"))
    .orderBy(desc("total_revenue"))        # ORDER BY revenue DESC
    .limit(100)                            # LIMIT 100
)

result.write.parquet("s3://mybucket/output/top_countries_2024/")
```

### Spark SQL

```sql
-- Spark SQL (identical to many SQL dialects)
SELECT
    country,
    product_category,
    SUM(revenue) AS total_revenue,
    SUM(quantity) AS total_quantity
FROM sales
WHERE year = 2024
GROUP BY country, product_category
ORDER BY total_revenue DESC
LIMIT 100
```

### When to Use Each Level

| Level             | Best for                                       | Examples          |
| ----------------- | ---------------------------------------------- | ----------------- |
| **SQL**           | Ad-hoc analytics, simple transformations       | Business analysts |
| **DataFrame**     | Complex multi-step pipelines, ML preprocessing | Data engineers    |
| **RDD/Low-level** | Custom algorithms, full control, ML libraries  | ML engineers      |
| **GUI**           | Exploration, dashboards, non-technical users   | Business users    |

---

## Tóm tắt phần này

```
MAPREDUCE:
   Map (emit key-value) → Sort+Shuffle (group by key) → Reduce (aggregate)
   Combiner: pre-aggregate at mapper (reduce network I/O)
   Limitation: always writes to DFS, only 2 stages, slow for iterative

SPARK:
   RDD: immutable, lazy, lineage-tracked
   Lazy evaluation → Catalyst optimizer → physical plan
   Fault tolerance: recompute lost partitions via lineage
   DataFrame API → SQL → Catalyst + Tungsten for performance

FLINK:
   Stream-native: batch = bounded stream
   Fault tolerance: periodic checkpointing (predictable recovery)
   Unified batch + stream API

API PYRAMID:
   RDD (flexible) → DataFrame (structured) → SQL (declarative)
   Different levels serve different user types
   All ultimately execute on same distributed engine
```
