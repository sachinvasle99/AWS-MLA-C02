# 02 – Feature Engineering (Domain 1, 28%)

Turning raw columns into numbers a model can learn from.

---

## 1. Missing values

| Technique | When |
|---|---|
| Drop the rows | Very few missing, lots of data |
| Drop the column | Most of the column is missing |
| **Mean/median imputation** | Numeric; median is safer when outliers exist |
| **Mode imputation** | Categorical |
| **"Missing" as its own category** | Categorical, and missingness itself is meaningful |
| **KNN / MICE imputation** | Best accuracy, more compute — "most accurate imputation" answers |
| Forward/backward fill | Time series |

> Rule: **median** beats mean when the data is skewed or has outliers.

---

## 2. Outliers

- Detect: box plot / IQR rule (outside Q1−1.5·IQR to Q3+1.5·IQR), z-score > 3, scatter plots.
- **Random Cut Forest (RCF)** — the SageMaker built-in algorithm for anomaly/outlier detection. Also available in Kinesis Analytics for streaming anomalies.
- Handle: remove, cap/clip (winsorize), or log-transform.

---

## 3. Encoding categorical data

| Technique | Use when | Note |
|---|---|---|
| **One-hot encoding** | Nominal (no order): colour, city, country | Blows up dimensions if too many categories |
| **Label / ordinal encoding** | Ordinal (has order): small < medium < large | Don't use for nominal — model invents fake order |
| **Binary / hashing encoding** | Very high cardinality (e.g. 100k user IDs) | Fewer columns than one-hot |
| **Target / mean encoding** | High cardinality, tree models | Risk of leakage — compute on train folds only |
| **Embeddings** | Text, very high cardinality, deep learning | Learned dense vectors |

> **Trap:** "a column with thousands of unique values, one-hot creates too many features" → **feature hashing** or **embeddings**.

---

## 4. Scaling numeric data

| Technique | Formula idea | Use when |
|---|---|---|
| **Normalisation (Min-Max)** | squeeze to 0–1 | Bounded range needed, neural nets, image pixels |
| **Standardisation (z-score)** | mean 0, std 1 | Default; needed when outliers exist; required by PCA, SVM, k-means, linear models |
| **Robust scaler** | uses median & IQR | Heavy outliers |
| **Log transform** | log(x) | Right-skewed data (income, price, counts) |

**Which algorithms need scaling?**
- **Need it:** anything distance- or gradient-based — k-means, k-NN, SVM, PCA, linear/logistic regression, neural networks.
- **Don't need it:** tree-based — Decision Tree, Random Forest, **XGBoost**.

> **Trap:** "XGBoost model performs poorly — should we normalise features?" → No. Trees are scale-invariant.

---

## 5. Other common transforms

- **Binning / discretisation** — turn age into age-groups. Reduces noise, captures non-linearity.
- **Polynomial / interaction features** — multiply features together to capture relationships.
- **Date-time expansion** — split timestamp into hour, day-of-week, month, is_weekend. Very common correct answer for time-based data.
- **Text:** tokenise → stop-word removal → stemming/lemmatisation → **Bag of Words / TF-IDF / n-grams / word embeddings (Word2Vec, BlazingText)**.
  - **TF-IDF** = weights a word by how rare it is across documents.
  - **n-grams** = keep word order context ("not good" vs "good").
- **Images:** resize, normalise pixels, and **data augmentation** (flip, rotate, crop, colour jitter) to grow a small dataset.

---

## 6. Dimensionality reduction

| Technique | Note |
|---|---|
| **PCA** | Linear, unsupervised; SageMaker has a built-in PCA. Must scale first. |
| **t-SNE / UMAP** | Visualisation only, not for production features |
| **Feature selection** | Drop low-variance, highly correlated, or low-importance features |
| **Autoencoders** | Neural compression of features |

"Curse of dimensionality" = too many features vs too few rows → overfitting, slow training. Fix with PCA or feature selection.

---

## 7. Class imbalance — high-yield topic

Problem: 99% "not fraud", 1% "fraud". A model predicting "never fraud" gets 99% accuracy and is useless.

| Fix | Detail |
|---|---|
| **Oversample the minority** | Duplicate minority rows |
| **SMOTE** | Generate *synthetic* minority samples — the named technique the exam loves |
| **Undersample the majority** | Throw away majority rows (loses information) |
| **Class weights** | XGBoost `scale_pos_weight`, or `class_weight` in sklearn — no data change needed |
| **Change the metric** | Stop using accuracy → use **F1, Precision/Recall, AUC-PR** |
| **Adjust the decision threshold** | Move from 0.5 to favour recall |
| **Anomaly detection framing** | If minority is ultra-rare, treat it as anomaly detection (RCF) |

> **Trap:** "Model has 99% accuracy but never catches fraud." → Imbalance. The answer is almost always *SMOTE / class weights / use AUC-PR or recall*, not "train longer".

---

## 8. Data leakage — know this cold

Leakage = information from the future or from the target sneaks into training features. Symptom: **suspiciously high validation accuracy that collapses in production.**

Causes and fixes:
- Scaling/imputing **before** the train/test split → fit the scaler on **train only**, then apply to test.
- A feature that is a proxy for the label (e.g. `discharge_date` when predicting admission).
- Target encoding computed over the whole dataset.
- Random split on **time-series** data → must use a **chronological split** (train on past, test on future).

---

## 9. Splitting data

| Split | Typical |
|---|---|
| Train / Validation / Test | 70 / 15 / 15 or 80 / 10 / 10 |
| **Stratified split** | Keeps class proportions — use for imbalanced classification |
| **k-fold cross-validation** | Small datasets; average score over k folds; more reliable estimate |
| **Time-based split** | Time series — never shuffle |

---

## 10. SageMaker Feature Store

A managed place to store, share, and reuse features.

| Part | What it does |
|---|---|
| **Online store** | Low-latency (ms) single-record lookup — for **real-time inference** |
| **Offline store** | S3 + Parquet + Glue catalog — for **training and batch**, queryable with Athena |
| **Feature group** | Table of features, with a record ID and event-time column |
| **Time travel** | Query feature values as of a past timestamp — reproduces old training sets |

**Why it exists (exam answer):**
1. Stops **training/serving skew** — the same feature code produces both training and inference values.
2. Lets teams **reuse and share** features instead of re-computing them.
3. Gives point-in-time correctness.

> **Trap:** "Features computed differently in training and production cause bad predictions" → **SageMaker Feature Store**.
> **Trap:** "Need features at <10 ms for a real-time endpoint" → **Feature Store online store** (or ElastiCache/DynamoDB if Feature Store isn't an option).

---

## Exam traps for this chapter

- Trees don't need scaling; distance/gradient models do.
- Ordinal encoding only for genuinely ordered categories.
- SMOTE = synthetic minority oversampling. Memorise the name.
- Fit transformations on **training data only**.
- Time series → chronological split, never random.
- Feature Store = online (real-time) + offline (training), solves skew.
- Data Wrangler can **export directly to Feature Store or a SageMaker Pipeline** — a common "operationalise this prep" answer.
