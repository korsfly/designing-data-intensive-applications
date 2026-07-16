# DDIA 2nd Edition — Chapter 11: Batch Processing

> **Sách:** Designing Data-Intensive Applications, 2nd Edition  
> **Tác giả:** Martin Kleppmann & Chris Riccomini  
> **Chương:** 11 — Batch Processing

---

## Mục tiêu chương

Chương 11 khám phá **Batch Processing** — cách xử lý dữ liệu lớn theo từng lô (batch), trái ngược với online systems phản hồi từng request. Batch processing là nền tảng của ETL, ML pipelines, search indexing, và nhiều analytical use cases.

> Epigraph:
> _"A system cannot be successful if it is too strongly influenced by a single person. Once the initial design is complete and fairly robust, the real test begins as people with many different viewpoints undertake their own experiments."_
> — Donald Knuth, "The Errors of TeX" (1989)

---

## Cấu trúc tài liệu này

| File                                   | Nội dung                                                                                                     |
| -------------------------------------- | ------------------------------------------------------------------------------------------------------------ |
| `01_Online_vs_Batch_and_Unix_Tools.md` | Online vs Offline systems, Unix pipeline philosophy, sorting vs hashing                                      |
| `02_Distributed_Infrastructure.md`     | Distributed filesystems (HDFS), Object stores (S3), Job Orchestration (YARN/Kubernetes), Workflow schedulers |
| `03_Processing_Models.md`              | MapReduce (4 steps, shuffle, combiner), Dataflow engines (Spark, Flink), DataFrame/SQL APIs                  |
| `04_Joins_Grouping_and_Use_Cases.md`   | Sort-Merge join, Broadcast hash join, Partitioned hash join, Batch use cases                                 |
| `05_Summary_and_Key_Concepts.md`       | Glossary, mindmap, câu hỏi ôn tập, liên kết các chương                                                       |

---

## Big Picture

```
┌──────────────────────────────────────────────────────────┐
│                   BATCH PROCESSING                        │
│                                                            │
│  IMMUTABLE INPUT → DERIVED OUTPUT (from scratch)          │
│  Primary metric: THROUGHPUT (not latency)                  │
│                                                            │
│  UNIX TOOLS:  awk | sort | uniq -c | sort -r | head       │
│  → Distributed version = Batch processing frameworks      │
│                                                            │
│  INFRASTRUCTURE:                                           │
│  Distributed Filesystem (HDFS) / Object Store (S3)        │
│  Job Orchestrator (YARN / Kubernetes)                      │
│  Workflow Scheduler (Airflow / Dagster / Prefect)          │
│                                                            │
│  PROCESSING MODELS:                                        │
│  MapReduce (map → sort → reduce)                           │
│  Dataflow Engines (Spark, Flink): in-memory DAG           │
│  APIs: Low-level RDD → DataFrame → SQL                    │
└──────────────────────────────────────────────────────────┘
```

---

## Các chương liên quan

- **Chapter 1:** Operational vs Analytical, ETL, Data Lake, Data Warehouse
- **Chapter 4:** LSM-tree (mergesort), Columnar storage (Parquet)
- **Chapter 5:** Encoding formats (Avro, Parquet used as batch file formats)
- **Chapter 7:** Sharding — parallel execution across shards
- **Chapter 9:** Hardware faults, preemption — fault handling in batch
- **Chapter 10:** ZooKeeper/etcd — YARN/Kubernetes cluster coordination
- **Chapter 12:** Stream Processing — batch as special case of stream
