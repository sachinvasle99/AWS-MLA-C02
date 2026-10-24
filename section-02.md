# Section 2 – Data Engineering Fundamentals

This section is mostly **not AWS services** — it's general data engineering theory that the exam guide explicitly asks for.

**Parts in this section**
1. [Types of Data](#1-types-of-data) — structured / unstructured / semi-structured
2. [Properties of Data – The 3 V's](#2-properties-of-data--the-3-vs) — volume / velocity / variety
3. [Data Warehouse vs Data Lake vs Lakehouse](#3-data-warehouse-vs-data-lake-vs-lakehouse) — ETL vs ELT, schema-on-write vs schema-on-read
4. [Data Mesh](#4-data-mesh) — who *owns* the data (organisation, not technology)
5. [ETL Pipelines in Detail](#5-etl-pipelines-in-detail) — what E, T and L each actually involve + orchestration
6. [Data Sources and Data Formats](#6-data-sources-and-data-formats) — JDBC/ODBC, streams · CSV, JSON, Avro, Parquet
7. [Amazon S3 – Introduction](#7-amazon-s3--introduction) — buckets, objects, keys, prefixes, metadata
8. [Amazon S3 – Security](#8-amazon-s3--security) — IAM vs bucket policies, ACLs, Block Public Access
9. [Amazon S3 – Performance](#9-amazon-s3--performance) — prefix limits, multi-part, Transfer Acceleration, byte-range fetches
10. [Amazon S3 – Access Points](#10-amazon-s3--access-points) — simplifying security at scale, VPC origin
11. [Amazon S3 – Object Lambda](#11-amazon-s3--object-lambda) — transform objects on the way out

> Note: only the lectures I didn't already know are written up here, so the numbering follows my notes, not the course's lecture numbers.

---
---

# 1. Types of Data

The one line to remember:

> **Structured = schema exists and is consistent. Semi-structured = schema exists but is inconsistent. Unstructured = no schema at all.**

## 1.1 Structured Data

**Definition:** data organised in a defined manner or schema — typically found in relational databases.

Columns are already defined, each with a known name and data type, and the relationships between them are already thought through.

**Characteristics**
- **Easily queryable** — you can throw SQL at it directly, because you already know the column names and types
- Organised into **rows and columns**
- **Consistent structure** — no surprises, already cleaned up

**Examples**
- Database tables — Oracle, Redshift, MySQL, PostgreSQL
- **CSV files** — *as long as* the columns are consistent (a CSV with varying numbers of columns per line is not structured)
- **Excel spreadsheets** — the normal kind, organised into proper rows and columns

> Caveat worth remembering: CSV and Excel *can* contain unstructured junk. What makes them structured is **consistency**, not the file extension.

## 1.2 Unstructured Data

**Definition:** data with **no predefined structure or schema**. Just a collection of data where you must first figure out what it is before you can query it.

**Key point:** you **cannot query it without pre-processing first**. You have to extract meaning from it and index it before it becomes searchable.

**Examples**
- **Raw text files** with no fixed format — all of Wikipedia, all of Reddit, the text of a book (e.g. for full-text search or training an LLM)
- **Images, audio, video** — you can't search these directly
- **Emails and word-processing documents**

**What "pre-processing" means here** — for a video you'd extract: the transcript, when it was created, how long it is, who made it, what the topic is. That extracted metadata becomes the structure you then query.

## 1.3 Semi-Structured Data

**Definition:** data that is **not as organised as structured data, but has some level of structure** in the form of tags, hierarchies, or other patterns.

The structure *is* in there — you just need to do a bit of work to tease it out.

**Characteristics**
- Elements are **tagged or categorised** in some way
- **More flexible** than structured, but **not as chaotic** as unstructured
- The schema may **not be consistent**, but at least it **exists**

**Examples**
- **XML and JSON files** — different records in the same document can have different shapes, but the tags tell you what each piece is
- **Email headers** — date, subject, sender are tagged fields (while the email *body* is unstructured)
- **Log files** — *the most important example in data engineering, because it's the most common*
  - Apache logs, server logs, service logs
  - Formats vary, and the information written isn't always consistent
  - Lots of missing data; some fields appear on some lines but not others
  - Needs **parsing line by line** to work out that line's structure

> Best one-line summary: **"the structure is in there, but it might not be consistent throughout the document."**

## 1.4 Side-by-side

| | **Structured** | **Semi-structured** | **Unstructured** |
|---|---|---|---|
| Schema | Defined and **consistent** | Exists but **inconsistent** | **None** |
| Queryable directly? | Yes, with SQL | Partly — needs parsing | No — needs pre-processing first |
| Flexibility | Low | Medium | High |
| Examples | DB tables, clean CSV, Excel | JSON, XML, log files, email headers | Images, audio, video, raw text, emails, docs |
| Typical AWS home | RDS, Redshift | S3, DynamoDB, OpenSearch | **S3** |

## 1.5 Quick self-test

1. A MySQL `customers` table → **Structured**
2. An Apache access log → **Semi-structured**
3. A `.mp4` CCTV recording → **Unstructured**
4. An API response in JSON → **Semi-structured**
5. A scanned PDF invoice → **Unstructured**
6. A clean sales `.csv` with the same 8 columns on every row → **Structured**
7. An email's subject + date fields → **Semi-structured** (the body is unstructured)

## 1.6 Bonus (needed later in feature engineering, not in this lecture)

Within *structured* data, columns are further split by value type:

```
Numerical    ├── Discrete    → whole counts        e.g. 3 orders
             └── Continuous  → any value in range  e.g. 36.6 °C

Categorical  ├── Nominal     → no order            e.g. city: Pune / Delhi   → one-hot encode
             └── Ordinal     → has order           e.g. size: S < M < L      → label encode
```

---
---

# 2. Properties of Data – The 3 V's

After *what kind* of data you have, the next question is *what is the data like*.

> **Volume** = how much? **Velocity** = how fast? **Variety** = how many different kinds?

(You'll see a 4th V — **Veracity**, how trustworthy the data is — in books and blogs. The exam guide only lists three.)

**Why these matter:** each V pushes you toward a *different architecture*. The 3 V's are really 3 design questions.

## 2.1 Volume — *How much data?*

The amount/size of data you're dealing with. Megabytes? Gigabytes? Petabytes?

**What it changes:** how you **store, move, and process** the data.

**Example — moving on-prem data into AWS**

| How much data | Sensible choice |
|---|---|
| A few GB | Just upload it over the internet / console / CLI |
| Several TB | DataSync or Direct Connect over the network |
| Petabytes, slow network | **AWS Snowball / Snowmobile** — physically ship the disks |

The volume alone decides whether you upload it or have AWS drive a truck to your data centre.

**Example — processing**

| How much data | Sensible choice |
|---|---|
| Small | A single relational database is perfectly fine — don't over-engineer |
| Big | **Distributed systems** that process in parallel — S3 + Spark/EMR, Glue, Redshift |

**Real-world examples**
- **Social media platform** — terabytes *every day* of posts, images, videos. How do you even store and query that?
- **Large retailer** — several years of transactions = petabytes. You can't do this on one monolithic database.

> Key idea: **big data brings big problems.** Past a certain size a single standalone database stops being an option and you must think distributed/parallel. But below that size, a plain database *is* the right answer.

## 2.2 Velocity — *How fast is data arriving?*

The speed at which new data is generated, collected, and processed.

**What it changes:** **batch** processing vs **streaming** (real-time) processing.

```
Low velocity   →  Batch        →  "Process everything once at the end of the day"
High velocity  →  Streaming    →  "Process each record as it arrives, continuously"
```

**Real-world examples**
- **IoT sensors** sending a reading every millisecond → streaming
  *(AWS appears to be reducing IoT content on the exam — treat it only as a convenient streaming example.)*
- **High-frequency trading** — every millisecond counts, transaction order must be correct, nothing can wait → real time
- **Daily sales report** — data can pile up all day and be processed at midnight → batch

**The nuance the exam loves: real-time vs near real-time**

Usually appears as **Kinesis Data Streams vs Kinesis Data Firehose**:

| | **Kinesis Data Streams** | **Kinesis Data Firehose** |
|---|---|---|
| Speed | **Real time** (~200 ms) | **Near real time** (~60 s buffer) |
| Who manages consumers | You do | Fully managed |
| Replay old data | Yes | No |
| Exam cue words | "real time", "custom processing", "replay" | "near real time", "just deliver to S3", "least operational overhead" |

> "Real time" and "near real time" are **not** the same phrase on this exam — they point to different services.

## 2.3 Variety — *How many different kinds and sources?*

Loops back to part 1: is the data **structured, semi-structured, or unstructured** — and where is it coming from?

**What it changes:** how many different stores you need, and how you query *across* them.

**Example — one business, three formats at once**

| Source | Format | Type |
|---|---|---|
| Relational database (sales) | Tables | **Structured** |
| Customer emails | Free text | **Unstructured** |
| Application logs | JSON | **Semi-structured** |

All three belong to the same business and must be analysed together.

**Example — healthcare system**
- Electronic medical records → structured
- Health device readings → streaming, semi-structured
- Patient feedback forms → unstructured text

> With high variety you'll often **store data in more than one place** (each store good at one format), then need a **unified way to query across all of them** — exactly why data lakes, Glue Data Catalog and Athena exist.

## 2.4 Putting it together

| V | The question | What it decides | Example answer |
|---|---|---|---|
| **Volume** | How much? | Storage + single vs distributed processing | Snowmobile vs console upload; EMR vs one database |
| **Velocity** | How fast? | Batch vs streaming | Firehose vs Kinesis Data Streams vs nightly Glue job |
| **Variety** | How many kinds? | How many stores + how to query across them | S3 data lake + Glue Catalog + Athena |

**One sentence for the exam:**
> Volume decides *how big your pipes are*, Velocity decides *whether the pipes run constantly or once a day*, and Variety decides *how many different pipes you need*.

---
---

# 3. Data Warehouse vs Data Lake vs Lakehouse

The single sentence that separates them:

> **Data Warehouse = clean it first, then store it. Data Lake = store it first, clean it later.**

## 3.1 Data Warehouse

The **legacy / traditional** approach. A **centralised repository optimised for analysis**, where data from many different sources is stored in a **structured** format.

**Characteristics**
- Designed for **complex queries and analysis**
- Data is **cleaned, transformed, and loaded** before it goes in → an **ETL** process
- Typically stored in a **star or snowflake schema** (covered later in data modelling)
- Optimised for **read-heavy** operations

**AWS example: Amazon Redshift.** (Equivalents elsewhere: Google BigQuery, Azure SQL Data Warehouse — not on the exam. Before Redshift existed, Amazon used a giant Oracle database for this — very expensive and hard to maintain.)

### Picture it — Amazon's own warehouse

```
   SOURCES                    DATA WAREHOUSE                VIEWS / DATA MARTS
 ┌───────────────┐
 │ Clickstream   │─┐
 │ (server logs) │ │        ┌──────────────────┐        ┌──► Accounting view
 └───────────────┘ │        │                  │        │
 ┌───────────────┐ ├──ETL──►│  Big structured  │────────┼──► Customer-behaviour mart
 │ Purchase data │ │        │    warehouse     │        │
 └───────────────┘ │        │                  │        └──► Machine-learning mart
 ┌───────────────┐ │        └──────────────────┘             (extract features →
 │ Catalog data  │─┘                                          train recommender)
 └───────────────┘
```

Purchase data and clickstream data both need to join to catalog data to make sense ("what did they click on, what did they buy").

**Key idea: one size does not fit all.** Different teams need different subsets and different indexes, so the warehouse exposes different **views** — a **data mart** is a focused subset of the warehouse for one team or purpose (accounting, analytics, ML).

## 3.2 Data Lake

A big storage repository holding a **vast amount of raw data in its native format** — structured, semi-structured, and unstructured all together.

The attitude is: **"throw it all in one place and figure it out later."**

**Characteristics**
- Stores large volumes of raw data with **no predefined schema**
- Data is loaded **as-is**, with no (or minimal) upfront pre-processing
- Supports **batch, real-time, or stream** processing
- Queried later for transformation or exploration

**AWS example: Amazon S3.** Dump logs and everything else in, extract structure later.

### The typical AWS data-lake pipeline

```
Raw files in S3  ──►  AWS Glue crawler  ──►  Glue Data Catalog  ──►  Athena / Redshift
(logs, JSON,          (infers the            (the schema,            Spectrum / EMR
 images, CSV)          schema)                stored separately)      query it with SQL
```

**The crucial point:** the data itself **never moves and is never converted**. It stays raw in S3. Glue only records *what the structure looks like*, and query tools read that catalog to know how to interpret the files.

(Other platforms: Azure Data Lake, HDFS/Hadoop. Not on the exam.)

## 3.3 The core difference — schema-on-write vs schema-on-read

This is the most important concept in this lecture.

| | **Data Warehouse** | **Data Lake** |
|---|---|---|
| Schema | **Schema-on-write** — you know the structure *before* writing it to disk | **Schema-on-read** — you work out the structure *when you read it* |
| Pipeline | **ETL** → Extract → **Transform** → Load | **ELT** → Extract → Load → **Transform** (later) |
| What's stored | Structured, **already transformed** data | **Raw data** in its original form |

**ETL vs ELT in plain words**

```
ETL (warehouse):  get data → reshape it to the known schema → put it in the database
ELT (lake):       get data → dump it in S3 as-is → reshape it whenever you need it
```

With ELT you can run **several different transformations** on the same raw data for different applications — because the original is still sitting there untouched.

## 3.4 Full comparison

| Aspect | **Data Warehouse** | **Data Lake** |
|---|---|---|
| Schema | Schema-on-write | Schema-on-read |
| Pipeline | **ETL** | **ELT** |
| Data stored | Structured, transformed | Raw, original form |
| Data types | Structured only — "you know what you've got" | Structured + semi-structured + unstructured |
| Agility | **Less flexible** — changing the schema means a painful migration project, downtime, and expensive specialists | **More flexible** — keep adding anything; deal with schema in a layer above |
| Cost | **More expensive** — more people, pricier tools, heavy optimisation work | **Cheaper** (S3 is cheap) — but huge volumes still add up |
| Optimised for | Fast, complex queries; BI | Storage, exploration, ML |
| AWS service | **Redshift** | **S3** (+ Glue + Athena) |

## 3.5 Which one do I choose?

**Choose a Data Warehouse when:**
- Your sources are **structured** and you **know the schema upfront** (so you know which indexes to build and how to optimise queries)
- You need **fast and complex queries**
- **Integrating data from different sources** is essential — *otherwise it's just a database, not a warehouse*
- The main use case is **business intelligence and analytics**

**Choose a Data Lake when:**
- You have a **mixture** of structured, semi-structured and unstructured data
- You need **scale and cost-effectiveness** for a massive amount of data
- You **don't yet know what you'll do with the data** — which is usually the reality
- You want flexibility later without reloading everything just to analyse it a new way
- The use case is **advanced analytics, machine learning, or data discovery**

> Very common ML pattern: raw logs / behaviour data sit in a **data lake** → extract features → train a model.

**And it's not either/or.** Plenty of companies run **both**, sometimes with the same data in each: raw data in the lake for flexibility, a copy loaded into the warehouse for BI. Right tool for the right job.

## 3.6 Data Lakehouse — the hybrid

Popularised by Databricks. A **hybrid architecture combining both**: the performance, reliability and capabilities of a warehouse, with the flexibility, scale and cheap storage of a lake.

**What it gives you**
- Supports **both structured and unstructured** data
- Allows **both schema-on-write and schema-on-read**
- Serves **both detailed analytics and machine learning**
- Built on cloud / distributed architectures
- Can bring **ACID transactions to big data** (e.g. Delta Lake) — ACID being the guarantees a database makes about how data is processed

**The AWS example to remember: Lake Formation + S3 + Redshift Spectrum.**

```
Data sits raw in S3  ──►  Redshift Spectrum queries it with SQL, like a warehouse
                     ──►  Lake Formation handles permissions / governance

  Feels like a warehouse on top, but the real storage underneath is still S3.
```

(Delta Lake on Apache Spark, Databricks Lakehouse Platform, and Azure Synapse Analytics are the non-AWS examples — mentioned for context only, not examinable.)

> This hybrid is where the industry is heading, so pay attention to Lake Formation + S3 + Redshift Spectrum throughout the course.

---
---

# 4. Data Mesh

A recent buzzword. The most important thing to understand:

> **A data mesh is NOT a technology. It is a way of organising people and ownership of data.**

Data warehouse and data lake answer *"where do I put the data?"*
A data mesh answers *"who owns the data and who is responsible for it?"*

## 4.1 The easy way to understand it

**The old way (centralised):** one central data team owns everything. Every other team sends them raw data and asks them for reports. That team becomes a bottleneck — and they don't actually understand the data as well as the team that produced it.

**The data mesh way (decentralised):** the team that *knows* the data best **owns** it, and publishes it for everyone else to use.

### Analogy — a shopping mall

Think of a big shopping mall:

- Each **shop** is run by its own owner, who knows their products best → that's a **data domain** (a team)
- Each shop puts clean, packaged goods on the shelf for customers → that's a **data product**
- A customer walks around and buys from several shops → that's a **use case** consuming data from multiple domains
- The **mall management** sets common rules: opening hours, safety, billing standards → that's **federated governance with central standards**
- The mall provides the building, electricity, security, parking → that's **self-service tooling and infrastructure**

Nobody goes into the shop's back-room storage to dig through boxes. They buy what's on the shelf. Same idea: **consumers use the published data product, not the raw data.**

## 4.2 The picture

```
        ┌──────────── Central standards & governance ────────────┐
        │   (security rules, access policy, catalog, tooling)    │
        └────────────────────────────────────────────────────────┘
              │                    │                   │
   ┌──────────▼────────┐ ┌─────────▼────────┐ ┌────────▼─────────┐
   │  DOMAIN: Sales    │ │ DOMAIN: Payments │ │ DOMAIN: Shipping │
   │  owns its data    │ │ owns its data    │ │ owns its data    │
   │  ↓ publishes      │ │  ↓ publishes     │ │  ↓ publishes     │
   │ [Data Product]    │ │ [Data Product]   │ │ [Data Product]   │
   └─────────┬─────────┘ └────────┬─────────┘ └────────┬─────────┘
             └──────────┬─────────┴────────────────────┘
                        ▼
            USE CASE: "fraud analysis"
            consumes data products from several domains,
            never touches anyone's raw data
```

Also called **domain-based data management**.

## 4.3 The four principles

| Principle | In plain words |
|---|---|
| **Domain ownership** | The team that produces/understands the data owns and maintains it |
| **Data as a product** | They don't just hoard it — they publish it in a clean, usable, documented form for others |
| **Federated governance with central standards** | Each domain enforces integrity, security and access control — but following **one common set of organisation-wide rules** |
| **Self-service infrastructure** | Domains aren't left to build everything alone; a shared platform is provided for them to build data products on |

> "Federated" is the key word: **decentralised ownership + centralised rules.** Not a free-for-all.

**Why Amazon likes this idea:** it's how Amazon has been structured internally for years — common governing principles, but individual teams own their own stuff.

## 4.4 How AWS fits in

A data mesh **doesn't prescribe any technology** — but AWS services are a good way to implement one:

| Need in a data mesh | AWS service |
|---|---|
| Govern access and permissions across domains | **AWS Lake Formation** |
| Central catalog so people can **discover** what data products exist | **AWS Glue Data Catalog** |
| Where the data actually lives | **S3**, Redshift, data lakes/warehouses |
| Self-service infrastructure for domains to build on | The AWS platform generally |

## 4.5 Don't confuse these

| Term | Answers the question |
|---|---|
| **Data Warehouse** | Where do I store *structured, transformed* data? |
| **Data Lake** | Where do I store *raw* data of any kind? |
| **Data Lakehouse** | Can I get both at once? |
| **Data Mesh** | **Who owns, maintains and publishes the data?** |

The first three are **architecture**. The fourth is **organisation and governance**. A data mesh can be *built on top of* lakes and warehouses.

---
---

# 5. ETL Pipelines in Detail

We met ETL earlier. Now let's open up each letter.

> **E**xtract = get the data out of wherever it lives
> **T**ransform = clean it and reshape it
> **L**oad = write it into the destination

**ETL vs ELT = the same three steps, just reordered.** Warehouse → ETL (transform before loading). Lake → ELT (load raw, transform later). The letters mean exactly the same thing in both.

```
ETL  (warehouse)   Extract ──► Transform ──► Load
ELT  (data lake)   Extract ──► Load ──────► Transform  (on demand, later)
```

---

## 5.1 E — Extract

**What it is:** retrieving raw data from wherever it comes from.

**Typical sources**
- External databases
- CRM systems (e.g. Salesforce)
- Flat files — log files, CSVs
- APIs into other systems
- Other data repositories

Your data is probably coming from **many places at once**, and you need to know how to interface with each one.

**Two things you must get right:**

**1. Data integrity — don't lose or corrupt anything.**
Real questions you have to answer:
- What if an API call fails mid-extract?
- Do you retry? How many times? With what backoff?
- How do you know afterwards that you got *everything*?

> This is the unglamorous part that breaks real pipelines. "Did we silently miss 2% of yesterday's rows?" is a genuine production problem.

**2. Velocity — how often do you extract?** (back to the 3 V's)

| Requirement | Extraction style |
|---|---|
| Every event matters immediately | **Real time** (streaming) |
| A few minutes' delay is fine | **Near real time** |
| Yesterday's data is fine today | **Batch** — nightly or weekly |

Decide this from the *requirements*, not by habit: how is the data generated, and how will it be queried?

---

## 5.2 T — Transform

**What it is:** converting the raw extracted data into a format suitable for the destination.

(In a warehouse this happens **upfront**. In a data lake it happens **on demand, later**. Same work, different timing.)

### The types of transformation

| Transformation | What it means | Example |
|---|---|---|
| **Cleansing** | Fix errors, remove duplicates, handle missing values | Two rows for the same order ID → keep one |
| **Enrichment** | Add extra data from other sources | Join a customer ID to the customer's region from another system |
| **Format conversion** | Change data types | **Date stored as the string `"2026-10-22"` → a real datetime type** (the most common example) |
| | | Text → integers; text → binary; for ML, categories → **one-hot encoding** |
| **Aggregation / computation** | Calculate totals, averages, counts | Daily revenue per store instead of (or alongside) every individual sale |
| **Encoding / decoding** | Unpack or repack the data | Incoming file is zipped or encrypted → decode it first; outgoing data → encode to **Parquet** (columnar) |
| **Handling missing data** | Decide what to do with nulls | See below |

### Missing data — the three choices

If a column must not contain nulls, you have to pick a policy:

1. **Drop** the rows that have missing values
2. **Impute** a placeholder or estimated value
3. **Reject and report** — produce a report saying "I rejected these rows, go fix them"

> There's no universally correct answer — **it depends on your requirements.** That phrase is the real lesson of this part.

### Enrichment needs more than one extract

Sometimes a single transform requires **combining two extractions** — you can't enrich sales data with customer details unless you've pulled both.

---

## 5.3 L — Load

**What it is:** moving the transformed data into the target warehouse or repository.

**How you load it** — same decision as extract, driven by **volume and velocity**:
- **Batch** — load everything at once, on a schedule
- **Streaming** — load continuously, as data becomes available

**Data integrity matters here too.** Just like extraction:
- Disk writes fail
- Things get backed up / throttled

> The rule: **never drop data on the floor without knowing about it.** A failed load that nobody noticed is worse than a load that crashed loudly.

---

## 5.4 Managing and orchestrating the pipeline

Look at how many steps we just described. Something has to **run them in the right order, on the right schedule, and retry them when they fail.** That's **orchestration**.

```
         ┌──────────── ORCHESTRATOR ────────────┐
         │  schedules · ordering · retries      │
         └──────────────────────────────────────┘
               │          │            │
          [Extract] → [Transform] → [Load]  → alert if anything fails
```

**AWS services for this (covered in depth later — just recognise them for now):**

| Service | Role |
|---|---|
| **AWS Glue** | Does the ETL/ELT work itself — can run automatically in response to incoming data events |
| **Glue Workflows** | Orchestration *within* Glue |
| **Amazon EventBridge** | Event-driven triggering ("when this happens, start that") |
| **Amazon MWAA** (Managed Workflows for Apache Airflow) | Full-featured scheduling/orchestration, Airflow DAGs |
| **AWS Step Functions** | Serverless state machine — multi-step workflows with branching and retries |
| **AWS Lambda** | Small, short, event-driven glue code between steps |

> Glue = *does* the work. EventBridge / Step Functions / MWAA / Lambda = *coordinate* the work.

---
---

# 6. Data Sources and Data Formats

Two separate questions: **where does the data come from** (sources), and **how is it written down** (formats).

---

## Part A — Data Sources

### JDBC vs ODBC — the pair you must know

Both are ways to connect to and query an external database. The exam cares about **one difference**, and it's a neat mirror image:

| | **JDBC** (Java Database Connectivity) | **ODBC** (Open Database Connectivity) |
|---|---|---|
| Platform | **Platform-independent** ✅ | **Platform-dependent** ❌ (needs specific drivers) |
| Language | **Language-dependent** ❌ (Java only) | **Language-independent** ✅ |

**Why:** JDBC is built on Java, so it runs anywhere Java runs (platform-independent) — but you must write Java (language-dependent). ODBC isn't tied to one language, but it needs a **driver installed for your platform/database**.

> **Memory hook:** **J**DBC = **J**ava. Java runs everywhere → platform-free, but language-locked. ODBC is the opposite on both counts.

**When to use which:** building your extraction tool in Java → JDBC. Not in Java → you need an intermediary layer → ODBC.

> In practice most ETL tools support **both**, hidden at a lower layer you never see. You just need to know what they stand for and how they differ.

### Other sources

| Source | Example |
|---|---|
| **Raw log files** | Writing server logs to S3 and ingesting them as-is |
| **APIs** | Some other interface into an external system (not JDBC/ODBC) — e.g. a REST API, Salesforce |
| **Streams** | Real-time data as it's produced — **Kafka**, **Kinesis** (covered in depth later) |

---

## Part B — Data Formats

### Quick comparison

| Format | Human readable? | Structure | Best for |
|---|---|---|---|
| **CSV** | ✅ Yes (text) | Tabular / structured | Small–medium data, interchange between systems |
| **JSON** | ✅ Yes (text) | **Semi-structured**, nested, key-value | APIs, config files, flexible schemas |
| **Avro** | ❌ No (**binary**) | Data **+ schema together** | **Schemas that change**, efficient transport |
| **Parquet** | ❌ No (**binary, columnar**) | Columnar | **Analytics** — reading a few columns out of many |

---

### CSV — Comma Separated Values

A **text-based, human-readable** tabular format. Each line = one row; values separated by commas.

```
order_id,customer,city,amount
1001,Sachin,Pune,2500
1002,Anita,Delhi,1800
```

**The delimiter doesn't have to be a comma.** Pipes `|` or tabs are common, especially because commas *inside* the data create escaping/quoting headaches. It's still loosely called a CSV.

**Use it when:**
- Data is **small to medium** — it's not encoded or compressed, so it's inefficient for large volumes
- You need a **common denominator between systems** — almost everything can read a CSV (the "lingua franca" of data interchange)
- You want it **human-readable and editable** — open it in a text editor, read it, change it, no encoding/decoding needed
- Importing/exporting from **databases or spreadsheets** (a spreadsheet implies the data isn't huge anyway)

**Used by:** SQL databases, Microsoft Excel, pandas (Python), R, most ETL tools.

---

### JSON — JavaScript Object Notation

Lightweight, **text-based, human-readable** interchange format using **key-value pairs**. Represents structured *or* **semi-structured** data.

```json
{
  "order_id": 1001,
  "customer": { "name": "Sachin", "city": "Pune" },
  "items": [
    { "sku": "A1", "qty": 2 },
    { "sku": "B7", "qty": 1 }
  ]
}
```

**The key difference from CSV:** because every value is **tagged with a key**, different records can have **different fields** — and you can **nest** structures inside each other. CSV can't do either.

**Use it when:**
- Exchanging data between a **web server and web client** (the de-facto standard)
- **Configuration and settings** files
- You need a **flexible schema** or **nested data structures**

**Used by:** web browsers, JavaScript/Python/Java, **RESTful APIs**, **NoSQL databases** like MongoDB (a natural home for semi-structured data).

---

### Avro — binary, carries its own schema

Now we leave human-readable territory. Avro is a **binary** format that stores **both the data and its schema together**.

```
┌──────────────────────────────┐
│  SCHEMA  │       DATA        │   ← one compact binary package
└──────────────────────────────┘
```

Because the schema travels with the data, **any other system can process it later without needing the original system's context**.

**Use it when:**
- **Your schema changes over time** ← this is the real use case
- Big data and **real-time processing** systems
- **Efficient serialisation for transport** between systems (A → B), since binary is compact and self-describing

> **Important nuance:** if your schema **never changes**, Avro **wastes space** by re-encoding the schema alongside every batch of data. Only worth it when the schema evolves.

**Used by:** Apache **Kafka**, **Spark**, **Flink**, **Hadoop**.

---

### Parquet — columnar, built for analytics

**This one comes up a lot.** Parquet is a **columnar storage format**: it stores data **by column instead of by row**.

**Row storage vs columnar storage:**

```
ROW-BASED (CSV):     [1001, Sachin, Pune, 2500][1002, Anita, Delhi, 1800]
                      └── whole row together ──┘

COLUMNAR (Parquet):  [1001, 1002]  [Sachin, Anita]  [Pune, Delhi]  [2500, 1800]
                      └ id col ─┘   └── name col ─┘  └ city col ┘   └ amt col ┘
```

**Why that's powerful:** if your query is `SELECT AVG(amount)`, Parquet reads **only the amount column** and ignores everything else. A row-based format would have to read every full row.

**Two big benefits:**
1. **Optimised for analytics** — queries usually need a few specific columns, not whole rows
2. **Efficient compression and encoding** — a column holds values of the **same type**, which compresses far better than mixed row data

**Use it when:**
- Analysing **large datasets** in analytics engines
- You have **many columns but each query needs only a few** ← the signature use case
- Storing data on **distributed systems** — different columns can live on different servers, so processing can be split by column, minimising I/O and storage overhead

**Used by:** Hadoop, Apache **Spark**, Apache **Hive**, Apache **Impala**, and **Redshift Spectrum** (very important in the AWS data world).

> If you remember one thing: **lots of columns, queries need only some of them → Parquet.**

---
---

# 7. Amazon S3 – Introduction

S3 is **one of the main building blocks of AWS** — advertised as **infinitely scaling storage**. Much of the web runs on it, and many AWS services use it for integrations. For us, it's where almost all ML data lives.

---

## 7.1 What people use S3 for

| Use case | Example |
|---|---|
| **Backup and storage** | Files, disk backups |
| **Disaster recovery** | Copy data to another **region** so it survives a region outage |
| **Archival** | Store old files very cheaply, retrieve later (S3 Glacier) |
| **Hybrid cloud storage** | On-prem storage extended into the cloud |
| **Application hosting** | Host app data |
| **Media hosting** | Video files, images |
| **Data lake / big data analytics** | ← **the one that matters most for this exam** |
| **Software delivery** | Distribute software updates |
| **Static website hosting** | Plain HTML/CSS/JS sites |

**Real customers:** NASDAQ stores **7 years** of data in **S3 Glacier** (the archival tier). Sysco runs analytics on its S3 data for business insights.

---

## 7.2 Buckets — the containers

> **Buckets are directories. Objects are the files inside them.**

**Key facts:**
- Buckets are **defined at the region level** — a bucket belongs to one specific AWS region
- The S3 **console looks global** (you see all your buckets together), but each bucket physically lives in one region

### Bucket naming

**The old rule:** bucket names had to be **globally unique** — across every region and every AWS account in the world. If someone took `my-data`, nobody else could ever use it.

**The new rule (recent change):** with the **account-regional namespace** feature, you can **reuse the same bucket name** across regions and accounts, even if someone else already used it. AWS appends a **suffix** to keep it unique behind the scenes.

**Naming constraints (still apply):**

| Rule | ✅ / ❌ |
|---|---|
| No uppercase letters | `MyBucket` ❌ → `mybucket` ✅ |
| No underscores | `my_bucket` ❌ → `my-bucket` ✅ |
| Must not look like an IP address | `192.168.1.1` ❌ |
| Must start with a lowercase letter or number | `-data` ❌ → `data1` ✅ |
| Must not start with the prefix `xn--` | `xn--test` ❌ |
| Must not end with the suffix `-s3alias` | `mydata-s3alias` ❌ |

> **In practice:** stick to lowercase letters, numbers and hyphens, and you'll never hit a problem.

---

## 7.3 Objects and keys — the part that trips people up

An **object** is a file. Every object has a **key**.

> **The key is the FULL PATH of the file** — not just its name.

```
Bucket: my-bucket

  my-bucket/my_file.txt
            └──── key = "my_file.txt"

  my-bucket/my_folder1/another_folder/my_file.txt
            └──────── key = "my_folder1/another_folder/my_file.txt" ────────┘
```

**A key is made of two parts:**

```
my_folder1/another_folder/     my_file.txt
└──────── PREFIX ────────┘    └ OBJECT NAME ┘
```

### The important concept

> **S3 has NO real concept of directories.** Everything is just a key.

The console *shows* you folders, and lets you *create* folders — but that's only the UI being friendly. Underneath, there are no directories at all. There are only **very long key names that happen to contain slashes**.

This matters later: when you "partition" data in a data lake as `year=2026/month=10/`, you're not making folders — you're just naming keys in a pattern that query tools understand.

---

## 7.4 What's inside an object

| Part | Detail |
|---|---|
| **Value / body** | The actual content of the file — anything you want |
| **Max object size** | **50 TB** (raised from 5 TB at re:Invent, Dec 2025) |
| **Multi-part upload** | **Required for any file larger than 5 GB** — the file is uploaded in several parts |
| **Metadata** | List of **key-value pairs**, set by the **system** or by the **user**, describing the file |
| **Tags** | **Unicode key-value pairs, up to 10** per object — very useful for **security and lifecycle rules** |
| **Version ID** | Present only **if versioning is enabled** on the bucket |

### Why the numbers are what they are

Multi-part upload allows a maximum of **10,000 parts**, each up to **5 GB**:

```
10,000 parts  ×  5 GB  =  50 TB   ← exactly the max object size
```

So the limits aren't arbitrary — the 50 TB ceiling *is* the multi-part maximum.

**The numbers to memorise:**

```
5 GB    → largest single upload; above this you MUST use multi-part upload
50 TB   → largest possible object
10,000  → maximum parts in a multi-part upload
10      → maximum tags per object
```

> 📅 **Recent change:** the max object size was **5 TB for years** and was raised to **50 TB in December 2025** (re:Invent). Older courses, books and practice exams will still say 5 TB — if an exam question offers both, go with what the rest of the question implies, but **50 TB is the current correct value**.

---
---

# 8. Amazon S3 – Security

There are **two directions** you can grant access from, and that's the whole mental model:

```
   USER-BASED                              RESOURCE-BASED
   "what is this user allowed to do?"      "who is allowed to touch this bucket?"
   ┌──────────────┐                        ┌──────────────────┐
   │ IAM policy   │ ──────────────────────►│  S3 Bucket       │
   │ on the user  │                        │  + bucket policy │
   └──────────────┘                        └──────────────────┘
   Attached to the IAM user/role           Attached to the bucket itself
```

## 8.1 The four mechanisms

| Mechanism | Type | Notes |
|---|---|---|
| **IAM policies** | User-based | Authorise which API calls a specific IAM user/role may make |
| **S3 Bucket policies** | Resource-based | **Bucket-wide rules**, set from the S3 console. **The most common way to secure S3 today.** |
| **Object ACLs** | Resource-based | Finer-grained, per object — **can be disabled** |
| **Bucket ACLs** | Resource-based | Bucket level — **much less common**, can also be disabled |

> **Bucket policies are the focus.** ACLs are the legacy approach and are usually switched off.

**Bucket policies let you:**
- Grant **public access** to the bucket
- **Force objects to be encrypted** at upload
- Grant access to **another AWS account** (cross-account)

Plus, separately: **encryption** of objects with encryption keys is another security layer.

---

## 8.2 The access rule — when can a principal access an object?

> An IAM principal can access an S3 object **IF**
> (**the IAM permissions allow it** **OR** **the resource/bucket policy allows it**)
> **AND there is no explicit DENY.**

Two things to notice:
1. It's an **OR** between the two policy types — either one granting access is enough
2. **An explicit DENY always wins**, no matter what else allows it

---

## 8.3 Anatomy of a bucket policy

Bucket policies are **JSON documents**. Four parts:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "PublicRead",
    "Effect": "Allow",
    "Principal": "*",
    "Action": ["s3:GetObject"],
    "Resource": ["arn:aws:s3:::examplebucket/*"]
  }]
}
```

| Field | Meaning | In this example |
|---|---|---|
| **Resource** | Which buckets/objects the policy applies to | `examplebucket/*` → the `*` means **every object** in the bucket |
| **Effect** | **Allow** or **Deny** | Allow |
| **Action** | Which S3 API calls | `s3:GetObject` → read/retrieve an object |
| **Principal** | Which account or user the policy applies to | `*` → **anyone** |

**Read it in one sentence:** *allow **anyone** to **get** **any object** in examplebucket* → this bucket is **publicly readable**.

---

## 8.4 The four access scenarios

### 1. Public access (a website visitor)
A random visitor on the internet wants to read files.
→ Attach a **bucket policy** allowing public access (the one above).

### 2. An IAM user in your own account
→ Attach an **IAM policy** to that user granting S3 permissions.

### 3. An **EC2 instance** needs access
→ **Do NOT use an IAM user.** Create an **EC2 instance role** with the right permissions and attach it.

> This is a general AWS principle: **applications and services use IAM roles, never IAM users with keys.**

### 4. **Cross-account access** (a user in a *different* AWS account)
→ You **must use a bucket policy** that allows that specific external IAM user.

> **This is the one to memorise: cross-account → bucket policy.** You can't attach an IAM policy to someone else's user — you don't control their account.

| Who needs access | Use |
|---|---|
| Anyone on the internet | **Bucket policy** |
| IAM user in your account | **IAM policy** |
| EC2 instance | **IAM role** (instance role) |
| User in another AWS account | **Bucket policy** (cross-account) |

---

## 8.5 Block Public Access — the safety net

A **bucket-level setting** (the one turned on by default when you create a bucket).

**Why it exists:** AWS invented it as an **extra layer of security to prevent company data leaks** — the classic "someone misconfigured an S3 bucket and leaked customer data" headline.

**How it behaves:**

> **Even if you write a bucket policy that makes the bucket public, Block Public Access will override it and the bucket will NOT be public.**

```
Bucket policy says "public"  ──X──►  Block Public Access = ON  ──►  still private ✅
```

**Best practice:**
- If a bucket should never be public → **leave these settings ON**. It protects you from your own mistake.
- If **no** bucket in your account should ever be public → set it **at the account level**.

---
---

# 9. Amazon S3 – Performance

*(Course lecture 29)*

Two halves: **the baseline you get for free**, and **three techniques to go faster**.

---

## 9.1 Baseline performance

S3 **automatically scales** to a very high request rate. You don't configure anything.

| Metric | Value |
|---|---|
| **First-byte latency** | **100–200 ms** |
| **Write requests** (PUT / COPY / POST / DELETE) | **3,500 per second, per prefix** |
| **Read requests** (GET / HEAD) | **5,500 per second, per prefix** |
| **Number of prefixes** | **Unlimited** |

> The phrase **"per second, per prefix"** is the whole trick. Because prefixes are unlimited, your **total** throughput is effectively unlimited — you just have to spread data across prefixes.

### What counts as a "prefix"?

> **The prefix is everything between the bucket name and the file name.**

```
bucket/folder1/sub1/file     →  prefix = /folder1/sub1
bucket/folder1/sub2/file     →  prefix = /folder1/sub2
bucket/1/file                →  prefix = /1
bucket/2/file                →  prefix = /2
```

All four files are called `file`, but they sit under **four different prefixes** — so each one gets its own 3,500 / 5,500 budget.

### The worked example

Four prefixes, reads spread **evenly** across them:

```
4 prefixes  ×  5,500 GET/HEAD per second  =  22,000 requests per second
```

> The word **"evenly"** matters. If all your traffic hammers one prefix, you're capped at 5,500 — the other prefixes' capacity is wasted. This is why data lakes partition keys (`year=/month=/day=/`): it spreads load across prefixes as a side effect.

---

## 9.2 Multi-part Upload — parallelise uploads

| Rule | Size |
|---|---|
| **Recommended** | files over **100 MB** |
| **Required** | files over **5 GB** |

**What it does:** splits the file into chunks and uploads them **in parallel**, maximising your bandwidth. S3 then reassembles the parts into the original file automatically.

```
        BIG FILE
   ┌──────┬──────┬──────┬──────┐
   │ part │ part │ part │ part │
   └───┬──┴───┬──┴───┬──┴───┬──┘
       │      │      │      │      ← all uploaded in PARALLEL
       ▼      ▼      ▼      ▼
   ┌──────────────────────────┐
   │        Amazon S3         │ → puts the parts back together
   └──────────────────────────┘
```

---

## 9.3 S3 Transfer Acceleration — speed up long-distance transfers

**Works for both upload and download.**

**The idea:** send the file to a nearby **AWS edge location** instead of straight to the distant bucket. The edge location then forwards it to the target region over **AWS's fast private network**.

There are **200+ edge locations** — far more than there are regions, so one is almost always close to you.

### Example — file in the USA → bucket in Australia

```
  USA file                 Edge location (USA)              Bucket (Australia)
     │                            │                                │
     │─── public internet ───────►│                                │
     │     (short hop, fast)      │─── AWS private network ───────►│
                                        (long hop, very fast)
```

> **The principle in one line:** *minimise the public internet, maximise the AWS private network.*

**It is compatible with multi-part upload** — you can use both together.

---

## 9.4 S3 Byte-Range Fetches — speed up downloads

Request only **specific byte ranges** of a file instead of the whole thing.

**Two use cases:**

**1. Parallelise downloads (speed)**
Split a huge file into ranges and fetch them **all at once**:

```
   [ bytes 0–99 ]  [ bytes 100–199 ]  [ bytes 200–299 ]  ...
         ▼                 ▼                  ▼
      all requested in parallel  →  faster download
```

Bonus: **better resilience** — if one range fails, you retry just that small range instead of the whole file.

**2. Retrieve only part of a file (efficiency)**
If you know the **first 50 bytes are a header**, request only those 50 bytes and get the information instantly — no need to download gigabytes.

---

## 9.5 Summary — which technique for what?

| Goal | Technique |
|---|---|
| Upload a large file faster | **Multi-part upload** (required >5 GB, recommended >100 MB) |
| Transfer across a long distance (either direction) | **S3 Transfer Acceleration** |
| Download a large file faster, or read just a piece of it | **Byte-range fetches** |
| More total requests per second | **Spread keys across more prefixes** |

---
---

# 10. Amazon S3 – Access Points

*(Course lecture 34)*

## 10.1 The problem they solve

Imagine one bucket holding **finance data** and **sales data**, with different teams needing different access.

Without access points, you write **one giant bucket policy** covering every user and every prefix. Every new team or dataset makes it longer. Eventually it becomes **unmanageable** — one huge policy file that nobody dares edit.

> **The core idea: stop putting all the security in the bucket policy. Give each group its own door into the bucket.**

## 10.2 How they work

An **access point** is a named entry point into a bucket, with **its own policy** (an **access point policy**, which looks just like a bucket policy).

```
                        ┌──────────── S3 BUCKET ────────────┐
                        │                                   │
  Finance team ──► [ finance access point ] ──► /finance    │  (read + write)
                        │                                   │
  Sales team   ──► [ sales access point ]   ──► /sales      │  (read + write)
                        │                                   │
  Analytics    ──► [ analytics access point ] ─► /finance   │  (READ ONLY)
                        │                      └► /sales    │
                        └───────────────────────────────────┘
                          + one simple bucket policy
```

- **Finance access point** → policy grants read/write on the `finance` prefix
- **Sales access point** → policy grants read/write on the `sales` prefix
- **Analytics access point** → policy grants **read-only** on **both** prefixes

Each team then needs the **IAM permission to use their access point**, and that's it — finance users can only reach the finance data, analytics can read both but write neither.

**The result:** security management moves **out of the bucket policy and into the access points**. The bucket policy stays **small and simple**, and each access point carries its own rules. That's how you **scale access** to a bucket.

## 10.3 Key properties

| Property | Detail |
|---|---|
| **Purpose** | Simplify security management for S3 buckets |
| **DNS name** | **Each access point gets its own DNS name** — that's how you connect to it |
| **Origin** | Either **internet** (public) or **VPC** (private traffic only) |
| **Policy** | An **access point policy**, very similar to a bucket policy |

## 10.4 VPC origin — keeping traffic private

If you set an access point's origin to **VPC**, it becomes **privately accessible only** — e.g. an EC2 instance reaching S3 **without going through the internet**.

To make that work you need a **VPC endpoint**:

```
┌─────────────── VPC ───────────────┐
│                                   │
│  EC2 instance                     │
│       │                           │
│       ▼                           │
│  [ VPC Endpoint ]                 │   ← must have a policy allowing
│       │   (has its own policy)    │     BOTH the access point AND the bucket
└───────┼───────────────────────────┘
        ▼
  [ S3 Access Point (VPC origin) ]      ← access point policy
        ▼
  [ S3 Bucket ]                         ← bucket policy
```

**Three layers of security stack up here:**
1. **VPC endpoint policy** — must allow access to **both the target bucket and the access point**
2. **Access point policy**
3. **S3 bucket policy**

> The detail people get wrong: the **VPC endpoint policy must allow BOTH** the access point *and* the underlying bucket. Allowing only one of them fails.

---
---

# 11. Amazon S3 – Object Lambda

Another use case **built on top of S3 Access Points** (the previous part).

## 11.1 The problem

You want **different applications to see different versions of the same object**.

The naive solution: **duplicate the bucket** — one copy with the original data, another copy with the data redacted, another copy converted to a different format… Now you're storing the same data three times and keeping all the copies in sync forever.

> **S3 Object Lambda's idea: keep ONE copy of the data, and modify the object *as it is being retrieved*.**

Nothing is duplicated. The transformation happens **on the fly**, per request.

## 11.2 How it works

```
                                      ┌──────────────────────────┐
  E-commerce app ─────────────────────►  S3 Bucket (ORIGINAL)    │
  (owns the data, GET/PUT direct)     │  one single copy         │
                                      └────────────▲─────────────┘
                                                   │ reads original
                                      ┌────────────┴─────────────┐
  Other app ──► [ Object Lambda  ] ──►│   Lambda function        │
                [ Access Point   ]    │   transforms the object  │
                        │             └──────────────────────────┘
                        └──► returns the MODIFIED object to the caller
```

The pieces, in order:
1. An **S3 Access Point** sits on the bucket (this is why Object Lambda needs access points)
2. An **Object Lambda Access Point** sits on top of that
3. When a caller does a GET through the Object Lambda Access Point, a **Lambda function runs**, reads the original object, transforms it, and returns the modified result
4. The **object in S3 is never changed** — the original stays exactly as it is

**The owning application** (the e-commerce app in the lecture) can still hit the bucket directly to GET and PUT the **original** object. Only callers going through the Object Lambda Access Point see the transformed version.

## 11.3 Typical use cases

| Use case | What the Lambda does |
|---|---|
| **Redacting PII** | Strip names, emails, card numbers before an analytics or test environment reads the data |
| **Converting formats** | XML → JSON on the fly for a client that can't read XML |
| **Enriching data** | Add information from another source (e.g. a database) into the returned object |
| **Resizing / watermarking images** | Return a thumbnail or a watermarked copy |
| **Different views per audience** | Dev/test sees masked data, production sees real data — **from the same single bucket** |

> The ML-relevant one: **one dataset in S3, with PII automatically redacted for the data science team**, while the owning application still sees the full records.

---

⚠️ **The transcript for this lecture was cut off mid-explanation** (it stops at the e-commerce app accessing the original object). The above completes the standard explanation — but if the rest of the lecture covered extra detail, paste it and I'll extend this part.

---
---

# Things to note (Section 2 exam traps)

**Types of data**
- "CSV = structured" is only true if the columns are **consistent**.
- **Log files are semi-structured**, not unstructured — the most commonly tested example.
- Unstructured data cannot be queried until it's **pre-processed and indexed**.
- Unstructured data lives in **S3**; structured lives in RDS/Redshift.
- Semi-structured's defining trait is an **inconsistent but present** schema (tags/hierarchy).
- Email: **headers are semi-structured, the body is unstructured**.

**3 V's**
- The exam guide lists **three V's: Volume, Velocity, Variety.** Veracity is real but **not** on the exam guide.
- **Velocity** is the one most likely to become a service-choice question → Kinesis Data Streams (real time) vs Firehose (near real time) vs batch (Glue/EMR).
- **Real time ≠ near real time.** Read that phrase carefully in every question.
- High **volume** → distributed/parallel (S3, EMR, Glue, Redshift); physically huge transfers → Snowball/Snowmobile.
- Low volume → a single database is acceptable. Don't pick the most complex answer by reflex.
- High **variety** → data lake on S3 + Glue Data Catalog + Athena.
- Don't over-study IoT — it's only a streaming example.

**Warehouse vs Lake vs Lakehouse**
- Memorise the pairing: **Warehouse = schema-on-write = ETL = Redshift.** **Lake = schema-on-read = ELT = S3.**
- In a data lake the raw data is **never converted or moved** — Glue only stores the *schema*, and Athena reads that catalog to query the files in place.
- A **data mart** = a focused subset/view of a warehouse for one team or use case.
- "Integrating data from multiple sources" is part of the definition of a warehouse — without it, it's just a database.
- Lakes are **cheaper and more agile**; warehouses are **faster for complex queries** but painful to re-schema.
- Mixed structured + unstructured data, or "we don't know what we'll do with it yet", or ML/data discovery → **data lake**.
- Known schema + BI/reporting + complex fast queries → **data warehouse**.
- You can legitimately have **both** — the exam answer isn't always one or the other.
- **Lakehouse on AWS = Lake Formation + S3 + Redshift Spectrum.** Query S3 with SQL as if it were a warehouse.
- **Delta Lake** = ACID transactions for big data (Databricks/Spark world, not AWS — context only).

**Data Mesh**
- If an answer option describes a data mesh as *a technology or an architecture*, it's wrong. It's a **data management paradigm** — organisation, ownership and governance.
- Keywords that point to data mesh: **domain ownership**, **data as a product**, **decentralised**, **federated governance**, **self-service infrastructure**, **domain-based data management**.
- **Decentralised ownership + centralised standards.** Both halves matter — it is not "every team does whatever it wants".
- On AWS, a mesh is usually implemented with **Lake Formation** (permissions/governance) + **Glue Data Catalog** (discovery) + **S3** (storage).
- Consumers use the **published data product**, not another domain's raw data.
- Lake/warehouse/lakehouse = *where data lives*. Mesh = *who owns it*. A question asking about **ownership across teams in a big organisation** is a mesh question.

**ETL pipelines**
- ETL and ELT use the **same three steps** — only the order of T and L changes. ETL → warehouse. ELT → data lake.
- **Data integrity must be ensured in BOTH extract and load**, not just one. Failed API calls on the way in, failed disk writes on the way out.
- The **extract and load frequency** (batch vs near real time vs real time) is decided by **velocity + requirements**, not preference.
- Transform covers six things: **cleanse, enrich, convert format, aggregate, encode/decode, handle missing data.**
- Classic format-conversion example: **date stored as a string → real datetime type.**
- Missing data has three valid policies: **drop / impute / reject-and-report.** The right one "depends on requirements."
- Enrichment may require **combining multiple extractions**.
- Know the split: **Glue performs** the ETL; **EventBridge, Step Functions, MWAA, Lambda, Glue Workflows orchestrate** it.
- **MWAA** = managed Apache Airflow → the answer when a question mentions Airflow or complex DAG scheduling.
- **Step Functions** → multi-step workflow with branching and retries. **EventBridge** → event-driven trigger. **Lambda** → short glue code.

**Sources and formats**
- **JDBC = platform-independent, language-dependent** (Java). **ODBC = platform-dependent, language-independent.** Mirror image — don't mix them up.
- CSV's delimiter **doesn't have to be a comma** (pipes and tabs are common, to avoid commas inside the data).
- **CSV vs JSON:** both human-readable text. CSV = flat/tabular only. **JSON = semi-structured, supports different fields per record and nesting.**
- **Avro** = binary + **schema stored with the data**. Its reason to exist is **schema evolution**. If the schema never changes, Avro **wastes space**.
- **Parquet = columnar.** Cue phrases: "many columns but queries only need a few", "analytics", "efficient compression", "distributed processing".
- Columns compress better than rows because **all values in a column share the same type**.
- Associate the ecosystems: **Avro → Kafka/Spark/Flink/Hadoop.** **Parquet → Spark/Hive/Impala/Hadoop/Redshift Spectrum.**
- Human-readable → CSV, JSON. Binary → Avro, Parquet.
- Small data / interchange between any systems / editable by hand → **CSV**. APIs, configs, nested data → **JSON**.

**Amazon S3**
- **Buckets are regional**, even though the console shows them all together. The *console* is global, the *bucket* is not.
- Bucket naming: **no uppercase, no underscores, not an IP, no `xn--` prefix, no `-s3alias` suffix**, must start with a lowercase letter or number.
- Bucket names used to be **globally unique**; the newer **account-regional namespace** relaxes this (AWS adds a suffix). Older exam questions may still assume global uniqueness.
- **The key is the full path**, not the filename. Key = **prefix + object name**.
- **S3 has no real folders.** Folders in the console are an illusion — everything is a flat key containing slashes. Expect this as a trick option.
- **5 GB** = the multi-part upload threshold. **50 TB** = max object size (raised from 5 TB in Dec 2025). **10,000** = max parts. **10** = max tags per object.
- Lots of study material still says **5 TB** — that's the old limit. 10,000 parts × 5 GB = 50 TB.
- **Tags** (max 10) are used for **security and lifecycle rules** — metadata is not.
- A **version ID** exists only when **versioning is enabled**.
- For ML: S3 is the **data lake and the default storage for every SageMaker training job**. If an answer stores training data somewhere other than S3, be suspicious.

**S3 security**
- Two directions: **IAM policy = attached to the user** ("what can this user do?"). **Bucket policy = attached to the bucket** ("who can touch this bucket?").
- Access rule: **(IAM allows OR bucket policy allows) AND no explicit DENY.** An **explicit DENY always wins.**
- **Cross-account access → bucket policy.** You cannot attach an IAM policy to a user in someone else's account. This is a very common exam question.
- **EC2 → IAM role**, never an IAM user with access keys. Same for Lambda, SageMaker, etc.
- Bucket policy JSON has four fields to recognise: **Resource, Effect, Action, Principal.** `Principal: "*"` = anyone; `Resource: ".../*"` = every object.
- **ACLs (object and bucket) are the legacy mechanism and can be disabled.** Bucket policies are the modern answer.
- **Block Public Access beats a public bucket policy.** If a question says "the policy allows public access but the bucket is still not public" → Block Public Access is enabled.
- Block Public Access can be set **per bucket or for the whole account**.
- Bucket policies can also **force encryption on upload** — a common way to enforce "all objects must be encrypted".

**S3 performance**
- Memorise: **3,500 writes/sec and 5,500 reads/sec — PER PREFIX.** Prefixes are **unlimited**, so total throughput is effectively unlimited.
- Writes = **PUT / COPY / POST / DELETE**. Reads = **GET / HEAD**.
- First-byte latency = **100–200 ms**.
- "Prefix" = **everything between the bucket name and the file name**.
- Expect the arithmetic question: *n* prefixes × 5,500 reads = total. The lecture's example is 4 × 5,500 = **22,000**.
- The word **"evenly"** is load-bearing — uneven traffic on one prefix still caps at 5,500.
- Multi-part upload: **recommended >100 MB, required >5 GB.** Don't swap these two numbers.
- **Transfer Acceleration = edge location + AWS private network.** Works for **upload AND download**, and is **compatible with multi-part upload**.
- **Byte-range fetch = downloads.** Multi-part upload = uploads. Easy to confuse — one is for getting, one is for putting.
- Byte-range fetches do two jobs: **parallelise a download** *and* **fetch just part of a file** (e.g. the first 50 bytes of header).
- Byte-range also gives **better failure resilience** — retry one small range, not the whole file.

**S3 Access Points**
- The cue phrase is **"the bucket policy is becoming too large / unmanageable"** or **"manage access at scale"** → **S3 Access Points**.
- Each access point has **its own policy** and **its own DNS name**.
- Security moves **out of** the bucket policy **into** the access points; the bucket policy stays simple.
- Origin is either **internet** or **VPC**. VPC origin = **private traffic only**.
- For VPC origin you must create a **VPC endpoint**, and its policy must allow **both the access point AND the bucket** — not just one.
- With VPC origin there are **three policy layers**: VPC endpoint policy → access point policy → bucket policy.
- Access points control *who gets in which door*; they don't change where the data lives. The objects stay in the same bucket.
- Don't confuse with **Object Lambda Access Points** (transform data on retrieval) — see the next part.

**S3 Object Lambda**
- Cue phrase: **"modify the object as it's being retrieved"** / **"avoid duplicating the bucket"** → **S3 Object Lambda**.
- It **requires an S3 Access Point** underneath — Object Lambda is built on top of access points, not a standalone feature.
- The **original object in S3 is never modified.** Only what the caller receives is transformed.
- Classic use cases: **redact PII**, convert XML↔JSON, enrich with other data, resize/watermark images.
- If an answer says "create a second bucket with a redacted copy", Object Lambda is the better answer — **one copy, no sync problem**.
- Access Point = *who can get in*. **Object Lambda Access Point = what they see when they do.**
