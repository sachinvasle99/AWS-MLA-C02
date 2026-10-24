# AWS Certified Machine Learning Engineer – Associate (MLA-C01 / MLA-C02)

My personal study notes. Written in simple language — no heavy maths, lots of "when do I pick what" tables.

> These notes are personal study material only. Nothing here is run against any work/Krista AWS account.
> If you want to practice hands-on, use your **own personal AWS account** with a billing alarm set at $5.

## How to use these notes

1. Read `00-exam-overview.md` first — know what the exam actually tests.
2. Go file by file, 1–2 per day.
3. Every file ends with **Exam traps** — those are the lines that actually win you marks.
4. In the last week, only revise `12-cheatsheet.md` + `13-practice-questions.md`.

## Index

| # | File | Topic | Exam weight |
|---|---|---|---|
| 00 | [00-exam-overview.md](00-exam-overview.md) | Exam format, domains, strategy | — |
| 01 | [01-data-preparation.md](01-data-preparation.md) | Ingestion, storage, formats, ETL | Domain 1 (28%) |
| 02 | [02-feature-engineering.md](02-feature-engineering.md) | Cleaning, encoding, scaling, imbalance, Feature Store | Domain 1 (28%) |
| 03 | [03-model-development.md](03-model-development.md) | Choosing a model, metrics, overfitting, bias | Domain 2 (26%) |
| 04 | [04-sagemaker-algorithms.md](04-sagemaker-algorithms.md) | Built-in algorithms cheat sheet | Domain 2 (26%) |
| 05 | [05-training-and-tuning.md](05-training-and-tuning.md) | Training jobs, HPO, distributed, cost saving | Domain 2 (26%) |
| 06 | [06-deployment-inference.md](06-deployment-inference.md) | 4 inference types, endpoints, A/B, autoscaling | Domain 3 (22%) |
| 07 | [07-mlops-orchestration.md](07-mlops-orchestration.md) | Pipelines, Step Functions, CI/CD, registry, IaC | Domain 3 (22%) |
| 08 | [08-monitoring-maintenance.md](08-monitoring-maintenance.md) | Model Monitor, drift, CloudWatch, cost | Domain 4 (24%) |
| 09 | [09-security-governance.md](09-security-governance.md) | IAM, VPC, KMS, Model Cards, Clarify | Domain 4 (24%) |
| 10 | [10-bedrock-genai.md](10-bedrock-genai.md) | Bedrock, RAG, prompting, fine-tuning, Guardrails | Domain 2 + GenAI |
| 11 | [11-ai-services.md](11-ai-services.md) | Comprehend, Textract, Rekognition, Transcribe… | Across domains |
| 12 | [12-cheatsheet.md](12-cheatsheet.md) | One-page "if you see X, answer Y" | Final revision |
| 13 | [13-practice-questions.md](13-practice-questions.md) | Q&A with explanations | Final revision |

## Udemy course notes (section by section)

**One file per course section.** Each new topic gets appended as a new part inside that section's file.

| Section | Title | Parts covered | File |
|---|---|---|---|
| 2 | Data Engineering Fundamentals | Types of Data · The 3 V's | [section-02.md](section-02.md) |

## Suggested 4-week plan

| Week | Do this |
|---|---|
| 1 | Files 00, 01, 02 + hands-on: upload data to S3, run a Glue DataBrew / Data Wrangler job |
| 2 | Files 03, 04, 05 + hands-on: train XGBoost in SageMaker on a small CSV |
| 3 | Files 06, 07 + hands-on: deploy a real-time endpoint, then delete it immediately |
| 4 | Files 08, 09, 10, 11 + revise 12, 13 |

## Hands-on safety rules (personal account)

- Set an AWS Budget alert at **$5** before touching anything.
- **Always delete endpoints** after practice — a running `ml.m5.xlarge` endpoint costs ~$140/month.
- Use `ml.t3.medium` / `ml.m5.large` for practice, never GPU instances.
- SageMaker Studio apps keep running even when the browser closes — shut them down.
- Prefer **SageMaker Serverless Inference** for practice: you pay only per request.
