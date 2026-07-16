# Joins, Grouping, và Batch Use Cases

---

## 1. Joins in Batch Processing

### Tại sao Joins là Thách thức trong Distributed Batch?

```
Single-machine join: easy (nested loop, hash join, sort-merge join)
   One machine has access to all data

Distributed join: HARD
   Data sharded across many machines
   Can't just "look up" another table (data on different node)
   Must physically move data to same node before joining

→ Key challenge: HOW to get matching records to same machine?
```

### Join 1: Sort-Merge Join (cũng gọi là Reduce-Side Join)

```
WHEN TO USE: Both datasets LARGE (neither fits in memory)

ALGORITHM:
   Step 1: MAP PHASE
      Both datasets → same mapper
      Mapper tags each record with SOURCE (left or right)
      Emits: (join_key, tagged_record)

   Step 2: SORT+SHUFFLE
      All records with SAME JOIN KEY → SAME REDUCER
      (This is the key insight — shuffle ensures colocation)

   Step 3: REDUCE PHASE
      Reducer receives: all records for 1 join key from BOTH sides
      Secondary sort: records from "left" side come first
      → Load left side into memory, then iterate right side

EXAMPLE:
   Users table: (user_id, name, country)
   Orders table: (order_id, user_id, amount)

   Mapper (Users): emit (user_id, {type:"user", name, country})
   Mapper (Orders): emit (user_id, {type:"order", order_id, amount})

   Reducer for user_id=123:
      Gets: {type:"user", name:"Alice", country:"US"}
            {type:"order", order_id:1001, amount:99.99}
            {type:"order", order_id:1002, amount:45.00}
      Emits: (1001, "Alice", "US", 99.99)
             (1002, "Alice", "US", 45.00)

SECONDARY SORT:
   Sort by (join_key, record_type)
   → User record arrives at reducer BEFORE order records
   → Load user into memory, then stream orders
   → No need to load both sides into memory

COST: Network shuffle of BOTH datasets → expensive
```

### Join 2: Broadcast Hash Join (cũng gọi là Map-Side Join)

```
WHEN TO USE: One dataset is SMALL (fits in memory)

ALGORITHM:
   Step 1: Load ENTIRE SMALL dataset into memory (hash table)
      Broadcast copy to ALL nodes

   Step 2: Stream through LARGE dataset
      For each large-side record: probe hash table
      If match found: emit joined record

   No shuffle needed! Join happens entirely in MAP phase.

EXAMPLE:
   Large: 10 billion web log records (terabytes)
   Small: 100,000 users table (megabytes)

   → Load users into HashMap<user_id, user_info>
   → Stream log records, lookup user for each log line

SPARK: Automatic broadcast join if small side < threshold
   Default: 10MB (configurable: spark.sql.autoBroadcastJoinThreshold)
   Manual: df.join(broadcast(small_df), "key")

COST: Only stream large side once, no shuffle = much faster!

REQUIREMENT: Small side must fit in worker memory
```

### Join 3: Partitioned Hash Join (cũng gọi là Bucket Join)

```
WHEN TO USE: Both datasets LARGE, already partitioned by same key

PRE-CONDITION: Both tables partitioned by join key
   Users partitioned by user_id (10 buckets: 0-9)
   Orders partitioned by user_id (10 buckets: 0-9)

ALGORITHM:
   Bucket 0 of Users → joins with Bucket 0 of Orders
   Bucket 1 of Users → joins with Bucket 1 of Orders
   ...
   Each bucket pair: hash join in memory

No cross-bucket shuffling needed!

COST: Requires pre-partitioned data on same key
      If not pre-partitioned: add shuffle step first (same as sort-merge)

Spark: "bucket join" (pre-bucketed tables in Hive Metastore)
Delta Lake: Z-ORDER by join key → co-located data → faster joins
```

### Comparison: When to Use Each Join

| Join Type            | Small × Large? | Large × Large?              | Memory requirement     | Cost                      |
| -------------------- | -------------- | --------------------------- | ---------------------- | ------------------------- |
| **Broadcast Hash**   | ✓ BEST         | ✗ Won't work                | Small side fits in RAM | Lowest                    |
| **Sort-Merge**       | ✓ Works        | ✓ BEST                      | O(one key's data)      | Medium                    |
| **Partitioned Hash** | ✓ Works        | ✓ BEST (if pre-partitioned) | One bucket per side    | Lowest if pre-partitioned |

---

## 2. Grouping và Aggregation

### GROUP BY with Aggregation

```
Most common batch pattern: GROUP BY key, aggregate values

Algorithm (similar to join):
   Mapper: emit (group_key, value)
   Shuffle: all same group_key → same reducer
   Reducer: aggregate all values for same key

Example: count orders per country per month
   Mapper: emit ((country, month), 1)
   Reducer: sum all 1s → count for (country, month)
```

### Sessionization (Complex Grouping)

```
Example: Group web events by user SESSION
   (session = events within N minutes of each other)

CHALLENGE:
   Events for same user scattered across all mappers
   Can't determine session boundaries without ALL user events together

SOLUTION:
   Step 1: Mapper emits (user_id, event)
   Step 2: Sort by (user_id, timestamp) — secondary sort
   Step 3: Reducer receives all events for 1 user, in timestamp order
           → Apply sessionization logic to ordered stream

SECONDARY SORT is crucial:
   Sort by (user_id ASC, timestamp ASC)
   → User events arrive at reducer in time order
   → Simple scan to identify sessions (gap > threshold = new session)
```

---

## 3. Batch Use Cases

### Use Case 1: Building Search Indexes

```
BATCH JOB: Build inverted index for search engine

Input: Billions of documents (web pages, product descriptions, emails)
Output: Inverted index (term → list of doc IDs)

MapReduce approach:
   Mapper: parse doc, extract words → emit (word, doc_id)
   Combiner: collect all doc_ids for same word (local)
   Shuffle: all (word, doc_ids) with same word → same reducer
   Reducer: merge lists → emit full postings list for each word

Output: Lucene index segments → merge → deployed to search servers
   (Atomic swap: old index → new index)

Scale: Google rebuilds web search index with batch jobs
Frequency: Daily or weekly full rebuild (or incremental updates)

VECTOR SEARCH INDEX:
   Same principle but: doc → vector embedding → build HNSW/IVF index
   ML model generates embeddings (batch inference)
   Then build approximate nearest neighbor index
```

### Use Case 2: Recommendation Systems

```
BATCH JOB: Train collaborative filtering model + generate recommendations

Phase 1: FEATURE ENGINEERING
   Collect user-item interactions (purchases, clicks, ratings, views)
   Join with item metadata
   Compute user features (demographics, history)

Phase 2: MODEL TRAINING
   Matrix factorization (user × item → latent factors)
   Alternating Least Squares (ALS): iterative MapReduce
   → Each iteration: one pass over data
   → 30-100 iterations needed

Phase 3: SCORING / BATCH INFERENCE
   For each user: compute score for all items
   Top-N recommendations per user
   Write to database/cache

Output: Pre-computed recommendations per user
   → Served from cache in real-time (no ML at query time)

Example: Netflix, Amazon, Spotify generate recommendations nightly
```

### Use Case 3: ETL for Data Warehouse

```
BATCH JOB: Extract-Transform-Load pipeline

Extract:
   Pull data from operational databases (CDC or full dump)
   Pull from SaaS APIs (Salesforce, Stripe, Google Ads,...)

Transform:
   Clean data (handle nulls, fix types, remove duplicates)
   Apply business logic (compute derived fields)
   Join multiple sources
   Reshape for analytical queries (denormalization)

Load:
   Write to data warehouse (Snowflake, BigQuery, Redshift)
   Or: write Parquet files to data lake (Iceberg/Delta table)

Tools: dbt (SQL transformations), Spark (Python transformations)
Frequency: Daily (or hourly for fresher data)
```

### Use Case 4: ML Feature Engineering & Feature Stores

```
BATCH JOB: Compute features for ML models

Features: derived attributes used by ML models
   User's purchase frequency in last 30 days
   Item's average rating
   User's preferred categories

Batch feature computation:
   Run daily batch job → compute features → write to feature store

FEATURE STORE:
   Storage for precomputed features (Feast, Tecton, Hopsworks)
   Online store: serve features at low latency for real-time inference
   Offline store: historical features for model training

Point-in-time feature lookup:
   Training data: (user_id, timestamp, features_at_that_time, label)
   → MUST look up features AS OF specific past timestamp
   → Prevents "data leakage" (using future information to predict past)
   → Requires time-travel queries on feature store
```

### Use Case 5: Graph Processing

```
BATCH JOB: Process large graphs

Examples:
   Social network: friend-of-friend recommendations
   Web graph: PageRank (importance of web pages)
   Fraud detection: suspicious transaction patterns
   Knowledge graph: entity relationships

CHALLENGE: Graphs are ITERATIVE
   PageRank: need multiple iterations until convergence
   Each iteration: update each node based on neighbors

MapReduce: AWKWARD for graphs
   Each iteration = separate MapReduce job
   30 iterations = 30 MapReduce jobs = 30 × DFS I/O cycles

BETTER: Pregel model (Google, 2010) → GraphX (Spark), Flink Gelly
   BSP (Bulk Synchronous Parallel):

   Superstep:
   1. Each vertex: compute new value based on received messages
   2. Send messages to neighboring vertices
   3. SYNCHRONIZE: all vertices wait for barrier
   4. Repeat until convergence

   All vertices active simultaneously (parallel)
   Messages passed between vertices (in memory, not DFS)
   → Much faster than MapReduce for iterative graph algorithms
```

---

## 4. Batch Processing vs Stream Processing

### Latency vs Throughput Trade-off

```
BATCH PROCESSING:
   ✓ HIGH THROUGHPUT: bulk I/O, large batches, efficient
   ✓ SIMPLE FAULT TOLERANCE: restart from immutable input
   ✓ GOOD FOR: historical analysis, ETL, ML training
   ✗ HIGH LATENCY: must complete entire job (minutes to hours)
   ✗ STALE DATA: output is snapshot of past

STREAM PROCESSING:
   ✓ LOW LATENCY: process events as they arrive (seconds)
   ✓ FRESH DATA: near-real-time output
   ✓ GOOD FOR: fraud detection, real-time dashboards, alerting
   ✗ COMPLEX FAULT TOLERANCE: can't restart from beginning
   ✗ COMPLEX TIME REASONING: event time vs processing time
```

### Lambda Architecture (Outdated)

```
PROBLEM in early 2010s:
   Batch: accurate but slow
   Stream: fast but approximate

LAMBDA ARCHITECTURE solution:
   Run SAME QUERY on BOTH batch and stream:

   Batch layer: accurate historical data (Hadoop, hours delay)
   Speed layer: approximate real-time data (Storm, seconds delay)
   Serving layer: merge both → return combined result

PROBLEM with Lambda:
   Must maintain SAME LOGIC in two different systems
   Batch code (Java+Hadoop) ≠ Streaming code (Storm/Flink)
   → Bugs in one but not other → inconsistent results
   → Double the code to maintain, double the bugs
   → Operational complexity × 2
```

### Kappa Architecture (Modern Approach)

```
INSIGHT: Batch = bounded stream
   Just process historical data as if it were events in a stream

KAPPA ARCHITECTURE:
   ONE system, ONE codebase for both historical and real-time

   For historical processing:
   → Replay historical events from log (Kafka, data lake)
   → Run same streaming job on replay

   For real-time processing:
   → Same streaming job, but on live events

TOOLS: Apache Flink, Apache Beam, Spark Structured Streaming
   All support both bounded (batch) and unbounded (stream) processing

BENEFITS:
   Single codebase (no duplication)
   Simpler operations
   Consistent results (same code → same logic)

→ Lambda architecture: mostly OUTDATED
→ Kappa architecture: PREFERRED modern approach
```

---

## Tóm tắt phần này

```
JOINS IN BATCH PROCESSING:
   Sort-Merge Join: both large sides, shuffle to co-locate → reduce-side
   Broadcast Hash Join: one small side, load into memory → map-side, FASTEST
   Partitioned Hash Join: pre-partitioned on same key → bucket join

GROUP BY + Aggregation:
   Same mechanism as join (shuffle by group key)
   Sessionization: secondary sort (key + timestamp) at reducer

BATCH USE CASES:
   Search indexes: inverted index or vector index from documents
   Recommendation systems: ALS training + batch inference + feature store
   ETL: Extract → Transform → Load to data warehouse/lake
   Graph processing: iterative algorithms, BSP (Pregel model)

BATCH vs STREAM:
   Batch: high throughput, simple fault tolerance, high latency
   Stream: low latency, fresh data, complex fault tolerance

   Lambda architecture: batch + stream (outdated, double maintenance)
   Kappa architecture: only stream (replay history) = MODERN preferred
```
