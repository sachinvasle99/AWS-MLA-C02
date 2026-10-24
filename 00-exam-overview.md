# 00 – Exam Overview

## The basics

| Item | Detail |
|---|---|
| Exam code | MLA-C01 (MLA-C02 is the refreshed version — same structure, more GenAI/Bedrock) |
| Level | Associate |
| Questions | 65 total — 50 scored, 15 unscored (you can't tell which) |
| Time | 130 minutes (~2 min per question) |
| Pass score | 720 / 1000 (scaled — roughly 70–75% correct) |
| Cost | $150 USD |
| Format | Multiple choice, multiple response, ordering, matching, case study |

## The 4 domains

| Domain | Weight | In plain English |
|---|---|---|
| **1. Data Preparation for ML** | 28% | Get data in, clean it, make features, store it |
| **2. ML Model Development** | 26% | Pick an algorithm, train it, tune it, measure it |
| **3. Deployment & Orchestration** | 22% | Ship it, automate it, make it scale |
| **4. Monitoring, Maintenance & Security** | 24% | Watch it, fix drift, lock it down, control cost |

Note the shape: **data + ops is 74%** of the exam. This is an *engineering* exam, not a data-science exam. You will get far more questions about "which service / which instance / which permission" than about maths.

## What this exam is NOT

- Not MLS-C01 (the old Specialty). Much less deep maths and algorithm theory.
- You do **not** need to derive gradients, remember formulas, or read Python line by line.
- You **do** need to know SageMaker's features cold — there are dozens of small named features and the exam loves them.

## The mental model that makes everything click

Think of every ML question as a pipeline with 5 stages. When you read a question, first decide **which stage it is in** — that alone eliminates half the answers.

```
 [1] INGEST        [2] PREPARE       [3] TRAIN        [4] DEPLOY       [5] OPERATE
 ---------------   ---------------   --------------   --------------   ---------------
 S3                Glue / DataBrew   SageMaker        Real-time        Model Monitor
 Kinesis           Data Wrangler       Training Job   Serverless       CloudWatch
 Firehose          EMR / Spark       Built-in algos   Async            Clarify (bias)
 Glue crawler      Athena            Script mode      Batch Transform  Pipelines
 Data Exchange     Feature Store     AMT (tuning)     Multi-model      Model Registry
 DMS               Processing Job    Spot training    Inference        A/B & shadow
                                                        Recommender
```

## Exam-day strategy

1. **Read the last line of the question first** — it tells you what's being optimised: *least operational overhead*, *lowest cost*, *lowest latency*, *minimum code change*. The "right" ML answer changes with that phrase.
2. **Qualifier → answer mapping** (memorise this, it's worth many marks):

| Phrase in question | What they want |
|---|---|
| "least operational overhead" / "minimal management" | Managed/serverless service. Pick SageMaker JumpStart, Bedrock, Serverless Inference, Glue over EMR, AI services over custom models. |
| "most cost-effective" | Spot training, Serverless/Async inference, Savings Plans, multi-model endpoints, scale-to-zero. |
| "lowest latency" / "real-time" | Real-time endpoint, provisioned concurrency, caching, Feature Store online store. |
| "large payload (>6 MB) or long processing (>60s)" | **Asynchronous Inference**. |
| "no endpoint needed / run on a schedule over a whole dataset" | **Batch Transform**. |
| "intermittent / unpredictable / idle traffic" | **Serverless Inference**. |
| "hundreds/thousands of similar models" | **Multi-model endpoint**. |
| "with minimal code changes" | Keep existing framework → SageMaker **script mode** / BYO container, not a rewrite. |
| "must not leave the VPC / no internet" | **VPC endpoints (PrivateLink)**, network isolation, VPC config on the job. |
| "need to explain the prediction" | **SageMaker Clarify** (SHAP). |
| "detect data/model drift" | **SageMaker Model Monitor**. |
| "track who did what" | **CloudTrail** (API calls) + **Model Cards** / **ML Lineage**. |

3. **Eliminate first.** Two options are usually obviously wrong (wrong service category entirely). Then pick between the remaining two using the qualifier.
4. Flag and move on if stuck past ~2.5 minutes. You will have time at the end.

## Exam traps

- "SageMaker Neo" = compile a model to run faster on a **specific hardware target** (edge/IoT). Not a training thing.
- "SageMaker Ground Truth" = **labelling** data. "Ground Truth Plus" = AWS does the labelling for you.
- "Data Wrangler" = visual data prep **inside Studio**. "Glue DataBrew" = visual data prep **outside** Studio, no code, for analysts.
- Anything with "Autopilot" or "Canvas" = low-code / no-code, used when the question says *team has no ML expertise*.
- If the question describes a problem a plain **AI service** (Comprehend, Rekognition, Textract) already solves, that's the answer — don't train a custom model. The exam rewards "don't build what AWS already built."
