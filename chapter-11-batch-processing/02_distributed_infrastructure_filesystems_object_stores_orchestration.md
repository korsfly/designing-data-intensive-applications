# Distributed Infrastructure: Filesystems, Object Stores, và Orchestration

---

## 1. Distributed Filesystems (DFS)

### HDFS (Hadoop Distributed File System)

```
Thiết kế: Lưu files phân tán trên nhiều machines
   Inspired by Google File System (GFS) paper (2003)

2 COMPONENTS:
   NAMENODE: Metadata service
      - Lưu: file names, directory structure, permissions
      - Lưu: file → block mapping (block trên node nào)
      - Single machine (SPOF — highly available NameNode ra đời sau)

   DATANODES: Data storage
      - Lưu actual blocks của files
      - Nhiều machines, tất cả đều DataNode
      - Gửi heartbeats đến NameNode

BLOCK SIZE: 128 MB (so với ext4: 4 KB)
   Lý do: Petabyte-scale → ít metadata overhead hơn
   1 PB / 128 MB = ~8 million blocks (vs 256 billion blocks ở 4KB)
```

### Replication trong HDFS

```
Mỗi block: replicated across DEFAULT 3 nodes
   Write: Primary → 2 replicas (pipeline write)
   Read: Client chọn replica gần nhất

Fault tolerance:
   1 node fail: still 2 copies
   Concurrent failures: rare with 3 replicas

Erasure coding (newer HDFS versions):
   Reed-Solomon codes: tolerate k failures with n-k parity shards
   Example: 6+3 coding: 9 blocks total, tolerates any 3 failures
   Storage overhead: 50% (vs 200% for 3-way replication)
   Trade-off: higher CPU cost for reconstruction
```

### Alternatives to HDFS

```
GlusterFS: POSIX-compliant, no central metadata server
           Uses DHT for file location

CephFS: POSIX-compliant, uses RADOS (Reliable Autonomic Distributed Object Store)
        Better for random I/O

JuiceFS: Cloud-native, stores data on object stores (S3/GCS)
         Metadata on Redis/TiKV

Lustre: High-performance computing, very fast sequential I/O
```

---

## 2. Object Stores

### Đặc điểm so với DFS

```
Object Store (S3, GCS, Azure Blob):
   Key-value API: GET/PUT/DELETE objects
   Objects: IMMUTABLE once written (update = full rewrite)
   No POSIX filesystem semantics
   No true directories (prefix convention)
   NO atomic rename (copy + delete = not atomic)

   "Directories": just naming convention (key prefix)
   List: like recursive ls -R (prefix listing)
```

### Object Store API

```
PUT object:
   s3://mybucket/2024/01/web-logs-00001.parquet
   Content: binary data, up to 5 TB per object

GET object:
   Range request: GET bytes 1000-2000 (partial fetch)
   → Useful for columnar formats (fetch only needed columns)

LIST objects:
   Prefix: "s3://mybucket/2024/01/" → all objects in "directory"
   Sorted lexicographically by key

MULTIPART UPLOAD:
   Large objects: upload in parts, commit atomically
   Each part: can be retried independently

CONDITIONAL WRITE (compare-and-swap):
   AWS S3: conditional PUT (if-none-match, if-match)
   → Enable distributed coordination on object stores
```

### DFS vs Object Stores — Chi tiết So sánh

| Feature               | DFS (HDFS)                                        | Object Store (S3)                      |
| --------------------- | ------------------------------------------------- | -------------------------------------- |
| **API**               | POSIX filesystem (open, read, seek, write, close) | REST API (GET, PUT, DELETE)            |
| **Object mutability** | Files mutable (append, random write)              | Objects immutable (PUT = full rewrite) |
| **Block size**        | 128 MB                                            | 4 MB (multipart upload)                |
| **Directories**       | Real filesystem hierarchy                         | Naming convention (prefix)             |
| **Atomic rename**     | ✓ Yes (for commits)                               | ✗ No (copy + delete)                   |
| **Data locality**     | Run tasks on nodes with data                      | Separate compute + storage             |
| **Latency**           | Lower (local disk or LAN)                         | Higher (WAN, API overhead)             |
| **Throughput**        | Very high (local disk)                            | High (parallel requests)               |
| **Cost**              | High (need dedicated hardware)                    | Low (cheap storage)                    |
| **Scalability**       | Limited by cluster size                           | Virtually unlimited                    |
| **Managed ops**       | Need to manage cluster                            | Fully managed                          |

### Tại sao Object Stores Thắng trong Cloud

```
OBJECT STORES (S3, GCS, Azure Blob) đã THAY THẾ HDFS trong nhiều deployments:

1. DISAGGREGATED COMPUTE + STORAGE (Chapter 1):
   - Scale storage và compute INDEPENDENTLY
   - Don't need to provision compute for storage capacity (và ngược lại)
   - Cheaper: only pay for what you use

2. DURABILITY: Multi-zone replication built-in
   - S3 Standard: 99.999999999% durability (11 nines)

3. DATA SHARING: Multiple compute services, teams access same data
   - No need to copy data between systems

4. COST: Object storage much cheaper per GB than DFS cluster

LIMITATION: Higher latency than local disk
   BUT: Modern datacenter networks are fast enough
   AWS S3 Express One Zone: single-digit ms latency
   → Acceptable for most batch workloads
```

### Open Table Formats (trên Object Stores)

```
Object store objects: immutable, no atomic rename
→ Hard to implement ACID transactions on object stores

OPEN TABLE FORMATS giải quyết vấn đề này:
   Apache Iceberg, Delta Lake (Databricks), Apache Hudi

Provide:
   ACID transactions on top of object stores
   Schema evolution (add/drop/rename columns safely)
   Time travel (query table at historical point)
   Partition evolution (change partitioning without rewriting data)
   Concurrent writers (optimistic concurrency)

Mechanism (Iceberg example):
   Metadata file → snapshot → manifest list → manifest → data files
   New write: create new data files + new metadata (atomic via conditional write)
   Old metadata: kept for time travel

Data Catalog needed:
   Track which table name → which metadata file
   Apache Polaris, Databricks Unity Catalog, Hive Metastore
```

---

## 3. Job Orchestration

### 3 Components of Distributed Job Execution

#### Component 1: Task Executors

```
Run on EACH NODE in cluster:
   YARN: NodeManager
   Kubernetes: kubelet

Responsibilities:
   - Start/stop task processes
   - Send HEARTBEATS to resource manager (signal alive)
   - Track task status (running, success, failure)
   - Report resource usage (CPU, memory)
   - Manage cgroups (resource isolation + security)
     → Each task gets dedicated CPU + memory limits
     → Prevent one task from starving others
     → Security isolation (multitenant clusters)
```

#### Component 2: Resource Manager

```
GLOBAL VIEW of cluster:
   YARN: ResourceManager
   Kubernetes: kube-apiserver + kube-scheduler

Knows:
   - Which nodes in cluster, their capacity
   - What's running where
   - Available resources

State storage:
   YARN → ZooKeeper
   Kubernetes → etcd
   → Fault-tolerant storage of cluster state
```

#### Component 3: Scheduler

```
Receives JOB REQUESTS → assigns tasks to nodes

Scheduling problem: WHICH task runs on WHICH node?

CONSIDERATIONS:
   Data locality: prefer node that has input data
   Resource fit: node has enough CPU/memory for task
   Job priority: higher-priority jobs get resources first
   Gang scheduling: all tasks of a job start simultaneously
      (needed for MPI, some ML frameworks)

ALGORITHMS (approximations to NP-hard optimal):
   FIFO (First In First Out): simple, no starvation prevention
   FAIR SCHEDULING: each job gets equal share of resources
   DRF (Dominant Resource Fairness): multi-resource fairness
   Priority queues: high-priority preempts low-priority
   Bin-packing: maximize resource utilization

FUNDAMENTAL CHALLENGE: NP-HARD problem
   → All production schedulers use HEURISTICS
   → Apache Hadoop's YARN Capacity Scheduler
   → Kubernetes Scheduler
   → Google Borg → Omega → Borg2 (academic research)
```

### Job Request Metadata

```
When submitting a batch job:
   Number of TASKS (parallelism)
   CPU and MEMORY requirements per task
   GPU requirements (for ML training)
   Location of executable code (container image, JAR file)
   Input and output data locations
   Job ID and credentials
   Max retries on failure
   Priority level
   Deadline (optional, for soft real-time)
```

---

## 4. Workflow Schedulers

### Workflow = DAG of Batch Jobs

```
Real-world pipelines: NOT just 1 job
   → SEQUENCE of jobs, each depending on previous
   → Called WORKFLOW or PIPELINE

Represented as DAG (Directed Acyclic Graph):
   Nodes = jobs
   Edges = dependencies (output of A → input of B)

Example 5-job workflow:
   [Extract logs] → [Parse/clean] → [Join with users DB] → [Compute metrics] → [Write to DW]

   Can have BRANCHING:
   [Parse/clean] → [Metrics] + [Search Index] + [User Profile Update]

Complexity: Large organizations: 50-100 job workflows
```

### Workflow Schedulers vs Job Schedulers

```
JOB SCHEDULER (YARN, Kubernetes):
   Schedules TASKS within a single job
   Knows nothing about other jobs or dependencies

WORKFLOW SCHEDULER (Airflow, Dagster, Prefect):
   Orchestrates MULTIPLE JOBS across a pipeline
   Tracks dependencies: "Job B can't start until Job A succeeds"
   Handles scheduling: "Run this pipeline every day at 2am"
   Error handling: "If Job B fails, retry 3 times, then alert"
   Monitoring: Show DAG visualization, job status, logs

→ Complementary (not competing)
   Workflow scheduler calls Job scheduler for each individual job
```

### Modern Workflow Schedulers

| Feature           | Airflow                     | Dagster                     | Prefect                     |
| ----------------- | --------------------------- | --------------------------- | --------------------------- |
| **Philosophy**    | DAG-first                   | Asset-oriented              | Workflow-first              |
| **Config**        | Python code (DAG)           | Python code (ops/assets)    | Python code (flows/tasks)   |
| **UI**            | DAG view, task status       | Asset lineage, data lineage | Flow runs, state management |
| **Type checking** | Limited                     | Strong (typed metadata)     | Limited                     |
| **Backfill**      | Good                        | Excellent                   | Good                        |
| **Observability** | Basic                       | Rich (data quality)         | Good                        |
| **Deployment**    | Complex                     | Moderate                    | Cloud-managed option        |
| **When to use**   | ETL pipelines, complex deps | Data teams, data lineage    | Simplicity-first            |

```
Common features of modern workflow schedulers:
   - Define pipelines as code (not XML/YAML hell)
   - DAG visualization
   - Retry policies, alerting
   - Parameterization (run same pipeline with different dates)
   - Backfill (run historical periods)
   - Integration with cloud services (S3, BigQuery,...)
   - Data versioning and lineage tracking
```

### Hadoop-era vs Modern

```
Hadoop-era schedulers:
   Apache Oozie: XML-based, tightly coupled to Hadoop
   Apache Azkaban: better UI but still Hadoop-centric

Modern workflow schedulers:
   Airflow (2015, Airbnb): Python DAGs, became de-facto standard
   Dagster (2018): asset-centric, type-safe, better for data teams
   Prefect (2018): simpler, cloud-managed option
   dbt: SQL-centric transformation pipelines (analytics engineering)

→ Trend: More Python-native, more cloud-native, asset-oriented
```

---

## 5. Fault Handling in Batch Systems

### Why Fault Handling Matters

```
Long-running batch jobs: hours to DAYS
Cluster with thousands of nodes:
   Node failure rate: 1-3% per month per machine
   1000-node cluster: 10-30 node failures per month
   → EXPECTED to have failures during long jobs

→ CANNOT just crash the job on any failure
   Must handle gracefully at TASK level
```

### Task-Level Retry

```
Tasks are INDEPENDENT units of work:
   Task A failure → DOESN'T affect Task B
   Failed task → RETRY on different node

Why different node?
   Current node may have hardware issue
   May be overloaded
   May have been preempted

Number of retries: configurable (typically 3-4)
After max retries: job FAILS (alert to operators)
Speculative execution: if task is SLOW, start copy on another node
   → Take result from whichever finishes first (straggler mitigation)
```

### Preemption — Spot Instances

```
PREEMPTION: Scheduler kills running task to make room
   for higher-priority work

Common scenarios:
   Higher-priority job arrives → low-priority task preempted
   Spot/Preemptible instances: cheap but killable

SPOT INSTANCES (AWS): up to 90% cheaper than on-demand
PREEMPTIBLE VMs (GCP): similar concept

Trade-off:
   COST: much cheaper
   RELIABILITY: instance can be terminated with 2-minute warning

Batch processing: PERFECT fit for spot instances
   - Jobs are fault-tolerant (task retry)
   - Don't need guaranteed availability
   - Long-running: big savings at 90% discount

Best practice: Mix on-demand + spot (spot for cheap, on-demand as backup)
```

### Intermediate Data Storage (Fault Tolerance Trade-offs)

```
MapReduce approach: Write intermediate data to DFS after EACH STAGE
   → SAFE: survives any node failure (data on durable DFS)
   → BUT: Many DFS reads/writes → very EXPENSIVE I/O
   → Jobs can be re-run from any intermediate checkpoint

Spark approach: Keep intermediate data IN MEMORY (local disk if spill)
   Write ONLY final result to DFS
   Track LINEAGE (transformation graph)
   On failure: RECOMPUTE lost partitions using lineage

   Trade-off:
   + Much faster (avoid DFS writes for each stage)
   - Recomputation needed on failure (not always cheap)
   - Works best when: data fits in memory, jobs are short

Flink approach: Periodic CHECKPOINTING of task state snapshots
   On failure: restart from last checkpoint (not from beginning)
   Trade-off: checkpoint overhead, but predictable recovery

→ Modern systems (Spark, Flink): prefer in-memory with recompute/checkpoints
   MapReduce: safer but much slower
```

---

## Tóm tắt phần này

```
HDFS:
   NameNode (metadata) + DataNodes (blocks)
   Block size 128MB, 3-way replication, erasure coding option

OBJECT STORES (S3, GCS):
   Immutable objects, no atomic rename, prefix "directories"
   Won in cloud: disaggregated compute/storage, cheap, managed
   Open Table Formats (Iceberg, Delta): ACID on object stores

JOB ORCHESTRATION:
   Task Executor (NodeManager/kubelet): runs tasks, heartbeats
   Resource Manager (YARN/Kubernetes): global cluster view
   Scheduler (DRF, FIFO, priority): NP-hard → heuristics

WORKFLOW SCHEDULERS (Airflow, Dagster, Prefect):
   DAG of jobs with dependencies, retry policies, monitoring
   ≠ Job scheduler (which schedules tasks within 1 job)

FAULT HANDLING:
   Task-level retry (on different node)
   Spot instances: 90% cheaper, killable → perfect for batch
   Intermediate data: DFS (safe, slow) vs in-memory+lineage (fast) vs checkpoint
```
