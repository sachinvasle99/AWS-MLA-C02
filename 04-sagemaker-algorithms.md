# 04 – SageMaker Built-in Algorithms (Domain 2)

You don't need the maths. You need: **what it's for, what data format, GPU or CPU**.

---

## The master table

| Algorithm | Problem type | One-line use case |
|---|---|---|
| **XGBoost** | Supervised: classification + regression | **The default for tabular data.** Gradient-boosted trees. |
| **Linear Learner** | Classification + regression | Fast linear baseline; trains many models in parallel and picks the best |
| **K-Means** | Clustering (unsupervised) | Customer segmentation |
| **KNN** | Classification + regression | Predict from nearest neighbours; also index/search |
| **PCA** | Dimensionality reduction | Compress many features |
| **Factorization Machines** | Classification/regression on **sparse** data | **Recommendations**, click prediction, high-dimensional sparse |
| **Random Cut Forest (RCF)** | Anomaly detection | Find outliers in metrics/streams |
| **IP Insights** | Anomaly detection on (entity, IPv4) pairs | Detect suspicious logins from unusual IPs |
| **DeepAR** | Time-series forecasting | Forecast **many related time series** at once (e.g. demand per store) |
| **BlazingText** | NLP | Fast text classification (supervised) and **Word2Vec embeddings** (unsupervised) |
| **Object2Vec** | Embeddings for arbitrary objects/pairs | Learn similarity between sentences, products, users |
| **Seq2Seq** | Sequence → sequence | Machine translation, summarisation, speech-to-text |
| **LDA** | Topic modelling (unsupervised) | Discover topics in documents; CPU-only |
| **Neural Topic Model (NTM)** | Topic modelling | Neural alternative to LDA, GPU-capable |
| **Image Classification** | CV | Label the whole image (ResNet) |
| **Object Detection** | CV | Find objects + bounding boxes |
| **Semantic Segmentation** | CV | Classify **every pixel** (self-driving, medical imaging) |
| **Text Classification – TensorFlow** | NLP | Transfer learning text classification |

---

## Grouped the way questions are asked

### Tabular
- **XGBoost** — almost always the right answer. Handles missing values, no scaling needed, great accuracy, built-in `scale_pos_weight` for imbalance.
- **Linear Learner** — when you need a simple, explainable, fast baseline.
- **KNN** — small data, simple, lazy learner.

### Time series
- **DeepAR** — the SageMaker answer. Key advantage: trains **one global model over many related series**, works for **cold-start** items (new product with no history), supports probabilistic forecasts (quantiles).
- **Amazon Forecast** — the fully-managed service version (less overhead, no code). If the question says "least operational overhead for forecasting", prefer Forecast.
- Classical: ARIMA/ETS/Prophet — single series only.

### NLP
- **BlazingText** — fast, cheap text classification; Word2Vec embeddings.
- **Seq2Seq** — translation/summarisation.
- **LDA / NTM** — topic modelling (no labels, "what are these documents about?").
- **Object2Vec** — similarity/embeddings for pairs.
- **Hugging Face containers / JumpStart / Bedrock** — modern transformer answers.

### Computer vision — know the difference

| Task | Output |
|---|---|
| **Image Classification** | One label for the whole image: "cat" |
| **Object Detection** | Boxes + labels: "cat at (x,y,w,h), dog at (…)" |
| **Semantic Segmentation** | A class for **every pixel** — pixel-level mask |

### Anomaly detection
- **Random Cut Forest** — numeric/streaming anomalies. Also in Kinesis Data Analytics.
- **IP Insights** — specifically (user, IP) pairs → account-takeover detection.

### Recommendations
- **Factorization Machines** — sparse user×item matrices.
- **Object2Vec** — embedding similarity.
- **Amazon Personalize** — the managed service; pick it for "least operational overhead" recommendation questions.

---

## Data formats and modes

| Algorithm group | Preferred format |
|---|---|
| Most built-ins | **RecordIO-protobuf** (fastest) or CSV |
| XGBoost | CSV, libsvm, Parquet, RecordIO |
| Image algorithms | RecordIO or raw images + `.lst` / augmented manifest |
| BlazingText | Single preprocessed text file, one sentence per line (`__label__x text`) |

**CSV rule for built-ins:** the **target must be the first column** and there must be **no header row**.

**Input modes:**
- **File mode** — download all data to the instance first. Simple, needs disk.
- **Pipe mode** — stream from S3 directly → faster start, less disk, works best with RecordIO.
- **FastFile mode** — stream but presented as a POSIX file system. Modern choice for large data without converting formats.

---

## CPU vs GPU

| Use GPU | Use CPU |
|---|---|
| Image Classification, Object Detection, Semantic Segmentation | XGBoost (traditionally CPU; GPU supported in newer versions) |
| Seq2Seq (GPU required), NTM, BlazingText Word2Vec | Linear Learner, K-Means, PCA, RCF, **LDA (CPU only)** |
| Deep learning in general | Classical ML |

> **Trap:** "Seq2Seq training on CPU is slow" → it **requires GPU**.
> **Trap:** "LDA on GPU" → LDA is **CPU-only**; use **NTM** if you need GPU topic modelling.

---

## Bringing your own

| Option | When |
|---|---|
| **Built-in algorithm** | AWS has one that fits — least effort |
| **Script mode** | You have existing TensorFlow/PyTorch/sklearn/HF code → run it in an AWS-managed framework container with minimal changes |
| **Bring your own container (BYOC)** | Custom framework/dependencies → push a Docker image to **ECR** |
| **Marketplace model** | Buy a model/algorithm |

For BYOC the container must respond to `/ping` (health) and `/invocations` (inference) on port 8080, and training reads `/opt/ml/input/data/...` and writes to `/opt/ml/model`.

**Key container paths (worth memorising):**
```
/opt/ml/input/data/<channel>/   training data
/opt/ml/input/config/           hyperparameters.json, resourceConfig.json
/opt/ml/model/                  save your model here → gets tar.gz'd to S3
/opt/ml/output/failure          write error message here if the job fails
/opt/ml/code/                   your script
```

---

## Exam traps for this chapter

- Tabular → **XGBoost**. Many related time series → **DeepAR**. Sparse recommendations → **Factorization Machines**.
- Pixel-level → **Semantic Segmentation**; boxes → **Object Detection**; one label → **Image Classification**.
- CSV for built-ins: **no header, label first**.
- "Minimal code change to my existing PyTorch script" → **script mode**, not BYOC.
- "Custom library not in AWS containers" → **BYOC in ECR**.
- Managed-service alternatives beat built-in algorithms when the question says *least operational overhead*: Forecast > DeepAR, Personalize > FM, Comprehend > BlazingText.
