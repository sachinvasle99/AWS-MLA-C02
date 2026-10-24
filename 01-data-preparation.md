# 01 – Data Preparation (Domain 1, 28%)

The biggest domain. Mostly "which AWS service do I use to move/clean/store this data".

---

## 1. Where data lives

### S3 — the default answer for ML storage
Everything in SageMaker reads from and writes to S3. If an option says "store training data in EBS/EFS only", be suspicious.

**S3 storage classes you must know:**

| Class | Use it when |
|---|---|
| S3 Standard | Active training data, frequently read |
| S3 Intelligent-Tiering | Access pattern unknown/changing — **the "least overhead" cost answer** |
| S3 Standard-IA | Accessed less than monthly, still need instant access |
| S3 Glacier Instant Retrieval | Archive but need it in milliseconds |
| S3 Glacier Flexible / Deep Archive | Long-term compliance archive of old training data |

**S3 Lifecycle policy** = automatically move old data to cheaper classes. Classic "reduce cost of old training data" answer.

### Other storage options

| Service | When it appears in the exam |
|---|---|
| **EFS** | Shared file system across many training instances / Studio notebooks home dirs |
| **FSx for Lustre** | **Fastest** option for large training datasets; links to S3; use when training is I/O bound and you re-read data many times (many epochs) |
| **EBS** | Attached to one instance only. Rarely the right ML answer. |

> **Trap:** "Training is slow because data loading is the bottleneck, dataset is huge, many epochs" → **FSx for Lustre**.

---

## 2. Getting data in (ingestion)

| Need | Service |
|---|---|
| Streaming records, you control consumers, replay needed | **Kinesis Data Streams** |
| Streaming → just dump into S3/Redshift/OpenSearch, no code | **Kinesis Data Firehose** (now "Amazon Data Firehose") |
| Transform a stream in real time (SQL / Flink) | **Managed Service for Apache Flink** |
| Video streams for ML | **Kinesis Video Streams** |
| Migrate a database into AWS / ongoing replication (CDC) | **AWS DMS** |
| Move files from on-prem NFS/SMB to S3 | **DataSync** |
| Physically ship huge data (petabytes, poor network) | **Snowball / Snowmobile** |
| Buy third-party datasets | **AWS Data Exchange** |
| Discover schema of files in S3 and build a catalog | **Glue Crawler → Glue Data Catalog** |

**Firehose vs Data Streams — the classic question:**

| | Data Streams | Firehose |
|---|---|---|
| Management | You manage consumers | Fully managed |
| Latency | ~200 ms (real-time) | ~60 s buffer (near real-time) |
| Replay | Yes (retention up to 365 days) | No |
| Destination | Anything you code | S3, Redshift, OpenSearch, Splunk, HTTP |
| Exam cue | "real time", "multiple consumers", "replay" | "least operational overhead", "just land it in S3" |

---

## 3. File formats — a guaranteed exam topic

| Format | Type | Use for |
|---|---|---|
| **CSV / JSON** | Row, text | Small data, human-readable. Slow and big. |
| **Parquet** | **Columnar**, compressed | **The default right answer** for analytics/ML data in S3. Cheaper Athena scans, faster reads. |
| **ORC** | Columnar | Similar to Parquet, Hive-world |
| **Avro** | Row, binary, schema-embedded | Streaming / schema evolution / Kafka |
| **RecordIO-Protobuf** | Binary | SageMaker **built-in algorithms' preferred format**; enables **Pipe mode** |

**Rules of thumb:**
- "Reduce Athena/Glue cost and scan time" → **convert to Parquet + partition + compress (Snappy)**.
- "Best performance for SageMaker built-in algorithm training" → **RecordIO-protobuf**.
- "Schema changes over time in a stream" → **Avro**.

### Partitioning
Store as `s3://bucket/data/year=2025/month=10/day=20/`. Athena/Glue then skips irrelevant folders. Exam phrase: "reduce the amount of data scanned" → **partitioning + columnar format**.

### Compression
Snappy (fast, splittable with Parquet), gzip (small but not splittable), bzip2 (splittable, slow). For Parquet just say Snappy.

---

## 4. Transforming data (ETL)

| Service | What it is | Pick it when |
|---|---|---|
| **AWS Glue (ETL jobs)** | Serverless Spark | Standard serverless ETL, Python/Scala, scheduled |
| **Glue Studio** | Visual drag-drop job builder | Visual ETL, low code |
| **Glue DataBrew** | 250+ prebuilt no-code transforms + data profiling | **Analysts with no code**, data quality profiling |
| **Glue Data Catalog** | Metadata store (tables/schemas) | Shared schema for Athena, EMR, Redshift Spectrum |
| **Glue Crawler** | Infers schema, populates catalog | "automatically discover schema" |
| **SageMaker Data Wrangler** | Visual data prep **inside SageMaker Studio**, 300+ transforms, exports to a Pipeline/Feature Store | ML-specific prep by a data scientist in Studio |
| **SageMaker Processing Job** | Run your own script (sklearn/Spark container) on managed instances | Custom pre/post-processing as a reproducible job |
| **EMR** | Managed Hadoop/Spark cluster | Huge data, need Spark tuning/control, existing Spark code, long-running |
| **Athena** | Serverless SQL over S3 | Ad-hoc queries, no infra |
| **Redshift** | Data warehouse | Structured BI, joins at scale; **Redshift ML** = train models with SQL |
| **Lambda** | Small, fast, event-driven transform | File lands in S3 → quick transform (<15 min) |
| **Step Functions** | Orchestrate all of the above | Multi-step workflow with branching/retries |

**Glue vs EMR — the recurring question:**
- Glue = serverless, less overhead, pay per job. Choose when the question says *minimal operational overhead*.
- EMR = you manage a cluster, more control/cheaper at massive sustained scale, supports Hive/Presto/HBase and Spot cost savings.

**Data Wrangler vs DataBrew:**
- Data Wrangler → inside Studio, by data scientists, flows into ML pipeline/Feature Store.
- DataBrew → standalone, by business analysts, data-quality profiling.

---

## 5. Data quality

- **Glue Data Quality** — define rules (DQDL) on catalog tables; detects anomalies; "ensure data quality before training".
- **DataBrew profile job** — statistics, missing values, correlations, distributions.
- **Deequ** (library on Spark/EMR) — unit tests for data. Rarely the answer but know the name.

---

## 6. Data labelling

| Service | What |
|---|---|
| **SageMaker Ground Truth** | You run a labelling job; workforce = your own team, Mechanical Turk, or a vendor |
| **Ground Truth Plus** | Turnkey — AWS manages the whole labelling operation for you |
| **Automated data labelling** | Ground Truth trains a model as humans label and auto-labels the easy ones → **reduces labelling cost** |
| **Active learning** | Same idea: only send uncertain samples to humans |

> **Trap:** "Reduce the cost/time of labelling a large dataset" → Ground Truth with **automated data labelling / active learning**.
> **Trap:** "Confidential data must not be seen by the public" → **private workforce**, never Mechanical Turk.

---

## 7. Data access and sharing

- **Lake Formation** — fine-grained (table/column/row-level) permissions over the data lake, centralised. Exam cue: "column-level access control for the data lake".
- **S3 Access Points / bucket policies / IAM** — coarser control.
- **Athena workgroups** — isolate query cost and results per team.

---

## Exam traps for this chapter

- Default to **S3 + Parquet + partitions** unless the question says otherwise.
- "Minimal operational overhead" in ETL → **Glue**, not EMR.
- "No-code for analysts" → **DataBrew**. "Visual prep in Studio for ML" → **Data Wrangler**.
- "Near real-time, just land it in S3" → **Firehose**. "True real-time with replay" → **Kinesis Data Streams**.
- "Shuffle/stream data into training without downloading it all" → **Pipe mode** (or FastFile mode) with RecordIO.
- EFS = shared, FSx Lustre = fast + S3-linked, EBS = single instance.
- **Pipe mode** streams data from S3 → faster start, less disk. **File mode** downloads everything first. **FastFile mode** = stream but looks like a file system (modern default for large data).
