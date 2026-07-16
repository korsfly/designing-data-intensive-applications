# Summary & Key Concepts — Chapter 11

---

## 1. Tổng hợp toàn chương

Chương 11 trình bày **Batch Processing** — cách xử lý dữ liệu lớn offline. Đây là nền tảng của ETL, ML training, search indexing và nhiều analytical pipelines trong thực tế.

### Luồng kiến thức

```
UNIX TOOLS (Single machine):
   awk | sort | uniq -c | sort -r | head
   Sort-based vs hash-based trade-offs
        │
        ▼ Scale up to distributed
DISTRIBUTED INFRASTRUCTURE:
   DFS (HDFS) vs Object Store (S3)
   Job Orchestration: Executor + Resource Manager + Scheduler
   Workflow: DAG of jobs (Airflow, Dagster, Prefect)
   Fault handling: retry, spot instances, checkpoints
        │
        ▼ Processing
PROCESSING MODELS:
   MapReduce: map → sort+shuffle → reduce
   Spark: in-memory DAG, RDD lineage, lazy evaluation
   Flink: stream-native, checkpointing
   APIs: RDD → DataFrame → SQL
        │
        ▼ Operations
JOINS & AGGREGATIONS:
   Sort-Merge (large+large), Broadcast Hash (small+large), Partitioned Hash
   GROUP BY: shuffle by key, secondary sort for ordered grouping
        │
        ▼ Applications
BATCH USE CASES:
   Search indexes, Recommendations, ETL, Graph processing
        │
        ▼ Evolution
BATCH vs STREAM:
   Lambda (outdated): batch + stream, double code
   Kappa (modern): only stream, replay history
```

---

## 2. Bảng thuật ngữ (Glossary)

### Core Concepts

| Thuật ngữ                 | Định nghĩa                                                          |
| ------------------------- | ------------------------------------------------------------------- |
| **Batch processing**      | Offline computation on large immutable dataset; output from scratch |
| **Online system**         | Request-response; primary metric = response time                    |
| **Offline system**        | Batch; primary metric = throughput                                  |
| **Throughput**            | Data processed per unit time (GB/s, records/s)                      |
| **Human fault tolerance** | Re-run from immutable input after bug fix                           |
| **Time travel**           | Query data at past point in time (Delta Lake, Iceberg)              |
| **External merge sort**   | Sort files larger than memory using temp files on disk              |
| **Combiner**              | Pre-aggregation in MapReduce: reducer logic at mapper stage         |
| **Shuffle**               | Transfer mapper output to correct reducer over network              |
| **Straggler**             | Slow task holding up entire job                                     |
| **Speculative execution** | Start duplicate task on another node; take first to finish          |

### Infrastructure

| Thuật ngữ             | Định nghĩa                                                       |
| --------------------- | ---------------------------------------------------------------- |
| **HDFS**              | Hadoop Distributed Filesystem: NameNode + DataNodes              |
| **NameNode**          | HDFS metadata service: file locations, block assignments         |
| **DataNode**          | HDFS data service: stores actual blocks                          |
| **Erasure coding**    | Reed-Solomon codes: less storage overhead than 3-way replication |
| **Object store**      | S3, GCS, Azure Blob: immutable key-value, no POSIX semantics     |
| **Open table format** | Iceberg, Delta Lake, Hudi: ACID transactions on object stores    |
| **Data catalog**      | Maps table names to metadata locations (Unity Catalog, Polaris)  |
| **Task executor**     | YARN NodeManager, Kubernetes kubelet: runs tasks on node         |
| **Resource manager**  | YARN ResourceManager, Kubernetes: global cluster view            |
| **Scheduler**         | Assigns tasks to nodes: DRF, FIFO, priority, bin-packing         |
| **Cgroups**           | Linux control groups: resource isolation and limits per task     |
| **Gang scheduling**   | Reserve all task resources before starting any                   |
| **Spot instance**     | AWS: cheap but killable VM (up to 90% discount)                  |
| **Preemptible VM**    | GCP: same concept as spot instance                               |
| **Workflow / DAG**    | Directed Acyclic Graph of batch jobs with dependencies           |
| **Airflow**           | Python-based workflow scheduler (Airbnb, 2015)                   |
| **Dagster**           | Asset-centric workflow scheduler with strong data lineage        |
| **Prefect**           | Simplified workflow scheduler, cloud-managed option              |
| **dbt**               | SQL-centric analytics transformation tool                        |

### Processing Models

| Thuật ngữ              | Định nghĩa                                                              |
| ---------------------- | ----------------------------------------------------------------------- |
| **MapReduce**          | Map (emit kv) → Sort+Shuffle → Reduce (aggregate)                       |
| **Mapper**             | Stateless function: input record → (key, value) pairs                   |
| **Reducer**            | Aggregate function: (key, [values]) → output records                    |
| **Secondary sort**     | Sort by (primary key, secondary key) for ordered reducer input          |
| **Dataflow engine**    | Spark, Flink: general DAG, in-memory intermediate data                  |
| **RDD**                | Spark's Resilient Distributed Dataset: immutable, lazy, lineage-tracked |
| **Transformation**     | Spark lazy operation returning new RDD (map, filter, join)              |
| **Action**             | Spark operation triggering execution (count, save, collect)             |
| **Lineage graph**      | DAG of transformations from input to output (Spark)                     |
| **Catalyst optimizer** | Spark's query optimizer: logical → physical plan                        |
| **Tungsten engine**    | Spark's code generation + off-heap memory for performance               |
| **Checkpointing**      | Flink: periodic state snapshots for fault recovery                      |
| **BSP**                | Bulk Synchronous Parallel: compute → communicate → synchronize          |
| **Pregel**             | Google's graph processing model based on BSP                            |
| **GraphX**             | Spark's graph processing library (implements Pregel model)              |
| **DataFrame**          | Structured distributed data with schema; SQL-like API                   |

### Joins

| Thuật ngữ                 | Định nghĩa                                                             |
| ------------------------- | ---------------------------------------------------------------------- |
| **Sort-merge join**       | Sort both by key, co-locate at reducer, merge — for large×large        |
| **Broadcast hash join**   | Load small side to memory, broadcast to all nodes — for small×large    |
| **Partitioned hash join** | Pre-partitioned by same key, local hash join per bucket                |
| **Map-side join**         | Join in mapper phase (no shuffle), requires broadcast or pre-partition |
| **Reduce-side join**      | Join in reducer phase (requires shuffle), works for any size           |
| **Bucket join**           | Partitioned hash join using pre-bucketed tables                        |

### Use Cases and Architecture

| Thuật ngữ                   | Định nghĩa                                                                    |
| --------------------------- | ----------------------------------------------------------------------------- |
| **Search index**            | Inverted index or vector index built from document corpus                     |
| **Feature store**           | Storage for precomputed ML features (Feast, Tecton, Hopsworks)                |
| **Point-in-time lookup**    | Feature values as of specific historical timestamp (prevents data leakage)    |
| **Collaborative filtering** | Recommendation algorithm: user-item matrix factorization                      |
| **ALS**                     | Alternating Least Squares: iterative matrix factorization for recommendations |
| **Lambda architecture**     | Batch + stream layer (outdated: double codebase)                              |
| **Kappa architecture**      | Stream only: replay history for batch (modern preferred)                      |
| **ETL**                     | Extract-Transform-Load pipeline for data warehouse                            |
| **ELT**                     | Extract-Load-Transform (transform in warehouse, modern variant)               |
| **DRF**                     | Dominant Resource Fairness: multi-resource fair scheduler                     |

---

## 3. Mindmap khái niệm

```
                        BATCH PROCESSING
                               │
          ┌────────────────────┼──────────────────────┐
          │                    │                      │
   UNIX TOOLS           DISTRIBUTED              PROCESSING
   (single machine)     INFRASTRUCTURE            MODELS
          │                    │                      │
   awk|sort|uniq      ┌────────┴───────┐       ┌──────┴──────┐
   Hash vs Sort       DFS          Object     MapReduce  Dataflow
   GNU sort:          (HDFS)       Store     Map→Shuffle  (Spark/
   external sort    NameNode+      (S3)      →Reduce     Flink)
   sort -parallel   DataNodes     Immutable  Combiner    in-memory
                    Erasure        objects   Secondary   lazy eval
                    coding         ACID:     sort        lineage
                                  Iceberg              checkpoints
                    Orchestration:                           │
                    Executor+Manager+Scheduler          APIs:
                    DRF,FIFO,gang scheduling        RDD→DataFrame→SQL
                    Spot instances                  Catalyst optimizer
                         │
                    Workflow (DAG of jobs):
                    Airflow, Dagster, Prefect
                         │
                    Fault handling:
                    task retry, lineage recompute,
                    checkpoint, speculative exec
                               │
          ┌────────────────────┼──────────────────────┐
          │                    │                      │
       JOINS               GROUPING             USE CASES
          │                    │                      │
    Sort-Merge          GROUP BY+Agg          Search indexes
    (shuffle both)     Sessionization         Recommendations
    Broadcast Hash     (secondary sort)       ETL/ELT
    (load small)       BSP (graphs)           Graph processing
    Partitioned                               Feature stores
    (same key)                                     │
                                          BATCH vs STREAM:
                                          Lambda (outdated)
                                          Kappa (modern):
                                          same code for both
```

---

## 4. Câu hỏi tự kiểm tra (Self-Check Questions)

1. Phân biệt **online systems**, **batch processing**, và **stream processing** về primary metrics, use cases, và cách xử lý failures.

2. Liệt kê **5 lợi ích của immutable input** trong batch processing. Tại sao "human fault tolerance" quan trọng?

3. Mô tả **Unix pipeline** để tìm top 5 URLs. Tại sao sort-based approach tốt hơn in-memory hash table khi dataset rất lớn?

4. So sánh **DFS (HDFS)** và **Object Store (S3)** theo: mutability, atomic rename, directories, data locality, cost.

5. Tại sao **object stores đã thắng** trong cloud environments? Open table formats (Iceberg, Delta Lake) giải quyết vấn đề gì?

6. Mô tả **3 components** của job orchestration. Tại sao scheduling là NP-hard? Các heuristics nào được dùng?

7. Phân biệt **Workflow Scheduler** (Airflow) và **Job Scheduler** (YARN, Kubernetes). Chúng complement nhau như thế nào?

8. Tại sao **Spot Instances** phù hợp với batch processing? Tác động đến fault handling?

9. Mô tả **4 bước MapReduce** (input, map, sort+shuffle, reduce). Combiner giải quyết vấn đề gì? Tại sao AVERAGE không thể dùng làm combiner?

10. Liệt kê **4 limitations của MapReduce** dẫn đến Spark và Flink ra đời.

11. Giải thích **RDD lineage** trong Spark. Tại sao lazy evaluation là lợi thế (Catalyst optimizer)?

12. So sánh **Spark fault tolerance** (lineage recompute) và **Flink fault tolerance** (checkpointing). Khi nào nên dùng cái nào?

13. Khi nào dùng **Sort-Merge Join** vs **Broadcast Hash Join** vs **Partitioned Hash Join**?

14. Giải thích **secondary sort** với ví dụ sessionization.

15. Mô tả batch processing workflow cho **recommendation systems** (feature engineering, model training, inference, feature store).

16. Tại sao **graph processing** (PageRank) khó với MapReduce? BSP/Pregel giải quyết thế nào?

17. Phân biệt **Lambda architecture** và **Kappa architecture**. Tại sao Lambda được coi là outdated?

---

## 5. Liên kết tới các chương khác

| Chapter 11 đề cập                          | Liên quan đến                                          |
| ------------------------------------------ | ------------------------------------------------------ |
| Data Warehouse, ETL, Data Lake             | **Chapter 1** — Operational vs Analytical Systems      |
| Columnar storage (Parquet), LSM-tree       | **Chapter 4** — Storage and Retrieval                  |
| Avro (batch file format), Parquet          | **Chapter 5** — Encoding and Evolution                 |
| Sharding → parallel execution              | **Chapter 7** — Sharding                               |
| Node failure, preemption                   | **Chapter 9** — The Trouble with Distributed Systems   |
| ZooKeeper/etcd for YARN/Kubernetes         | **Chapter 10** — Coordination Services                 |
| Workflow schedulers (vs durable execution) | **Chapter 5** — Durable Execution (Temporal)           |
| Stream processing (Chapter 12)             | **Chapter 12** — Stream Processing                     |
| Open table formats (Iceberg, Delta)        | **Chapter 3** — Event Sourcing (immutable append-only) |

---

## 6. Trích dẫn mở đầu chương

> _"A system cannot be successful if it is too strongly influenced by a single person. Once the initial design is complete and fairly robust, the real test begins as people with many different viewpoints undertake their own experiments."_
> — **Donald Knuth**, "The Errors of TeX" (1989)

→ Knuth nhấn mạnh rằng thiết kế thực sự được kiểm chứng khi **nhiều người dùng với nhiều use cases khác nhau** sử dụng hệ thống — điều này áp dụng hoàn hảo cho batch processing: MapReduce ban đầu được thiết kế bởi Google cho web indexing, nhưng khi Hadoop release dưới dạng open-source, cộng đồng đã phát hiện ra vô số use cases mới mà Google không nghĩ đến — và cũng phát hiện ra nhiều limitations, dẫn đến Spark và Flink.
