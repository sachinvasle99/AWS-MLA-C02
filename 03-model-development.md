# 03 – Model Development (Domain 2, 26%)

Choosing a model, measuring it, and fixing it when it's bad.

---

## 1. Problem type → model family

| If the question asks for… | It's this problem | Typical answer |
|---|---|---|
| A number (price, demand, temperature) | **Regression** | Linear Learner, XGBoost, DeepAR |
| Yes/no | **Binary classification** | XGBoost, Linear Learner |
| One of N labels | **Multi-class classification** | XGBoost, Image Classification |
| Many labels at once | **Multi-label** | Neural net with sigmoid outputs |
| Group similar items, no labels | **Clustering** | K-Means |
| Find the weird ones | **Anomaly detection** | Random Cut Forest, IP Insights |
| "Customers who bought X also bought Y" | **Recommendation** | Factorization Machines, Amazon Personalize |
| Predict future values over time | **Forecasting** | DeepAR, Amazon Forecast |
| Understand/generate text | **NLP** | BlazingText, Hugging Face, Bedrock |
| Find objects in images | **Computer vision** | Object Detection, Rekognition |

---

## 2. Supervised vs unsupervised vs reinforcement

- **Supervised** — labelled data. Classification + regression.
- **Unsupervised** — no labels. Clustering, dimensionality reduction, anomaly detection, topic modelling.
- **Semi-supervised** — small labelled set + large unlabelled set (Ground Truth active learning).
- **Self-supervised** — model creates its own labels from the data (how LLMs are pre-trained).
- **Reinforcement learning** — agent + environment + reward. SageMaker RL, AWS DeepRacer. Also **RLHF** for aligning LLMs.

---

## 3. Evaluation metrics — guaranteed exam questions

### Confusion matrix

```
                 Predicted Positive   Predicted Negative
Actual Positive      TP                   FN  (missed it)
Actual Negative      FP (false alarm)     TN
```

| Metric | Formula | Plain meaning | Optimise when |
|---|---|---|---|
| **Accuracy** | (TP+TN)/all | Overall correctness | Classes are balanced. **Useless when imbalanced.** |
| **Precision** | TP/(TP+FP) | Of the ones I flagged, how many were right? | **False positives are expensive** — spam filter, blocking a legit transaction |
| **Recall / Sensitivity / TPR** | TP/(TP+FN) | Of the real positives, how many did I catch? | **False negatives are expensive** — cancer detection, fraud, safety |
| **F1 score** | harmonic mean of P and R | Balance of both | Imbalanced data, need one number |
| **Specificity / TNR** | TN/(TN+FP) | How well do I identify negatives | Medical screening pairing |
| **AUC-ROC** | area under TPR vs FPR curve | Ranking quality across all thresholds | Balanced-ish, compare models. 0.5 = random, 1.0 = perfect |
| **AUC-PR** | area under Precision-Recall curve | Same idea but **better for heavy imbalance** | Rare positives (fraud) |

**Memory hook:**
- "We must not **miss** any cases" → **Recall**.
- "We must not **falsely accuse**" → **Precision**.
- "Both matter, data is skewed" → **F1** or **AUC-PR**.

### Regression metrics

| Metric | Meaning | Note |
|---|---|---|
| **MAE** | Mean absolute error | Robust to outliers, same unit as target |
| **MSE** | Mean squared error | Punishes big errors heavily |
| **RMSE** | √MSE | Most common; same unit as target; outlier-sensitive |
| **MAPE** | Mean absolute % error | Comparable across scales; breaks when actual = 0 |
| **R²** | Variance explained (0–1) | How much better than predicting the mean |
| **RMSLE** | log version | Use when you care about relative error / skewed targets |

> "Outliers must not dominate the error" → **MAE**. "Large errors are especially bad" → **RMSE/MSE**.

### Other metrics
- **Clustering:** silhouette score, within-cluster SSE (**elbow method** to choose k).
- **Ranking/recommendation:** precision@k, recall@k, NDCG, MRR, coverage.
- **NLP:** BLEU (translation), ROUGE (summarisation), perplexity (language models).
- **Object detection:** mAP (mean Average Precision), IoU.

---

## 4. Overfitting vs underfitting — the bias/variance table

| | **Underfitting (high bias)** | **Overfitting (high variance)** |
|---|---|---|
| Symptom | Bad on train **and** test | Great on train, bad on test |
| Cause | Model too simple, too few features, over-regularised, trained too little | Model too complex, too few rows, trained too long, leakage |
| Fixes | Bigger/more complex model, more features, less regularisation, train longer, reduce dropout | **More data**, data augmentation, fewer features, **regularisation (L1/L2)**, **dropout**, **early stopping**, simpler model, **cross-validation**, ensembling, pruning |

**Regularisation:**
- **L1 (Lasso)** — drives weights to exactly zero → **does feature selection**. Use for sparse/high-dimensional data.
- **L2 (Ridge)** — shrinks weights smoothly, keeps all features. Default.
- **Elastic Net** — both.
- **Dropout** — randomly disable neurons during training (neural nets only).
- **Early stopping** — stop when validation loss stops improving. XGBoost: `early_stopping_rounds`.

> **Trap:** "Train accuracy 99%, validation accuracy 70%" → **overfitting** → more data / regularisation / early stopping / simpler model. Not "train longer".
> **Trap:** "Both train and validation accuracy are low" → **underfitting** → more complex model / more features.

---

## 5. Key hyperparameters to recognise

| Hyperparameter | Effect |
|---|---|
| **Learning rate** | Too high → loss diverges/oscillates. Too low → very slow convergence. The #1 thing to tune. |
| **Batch size** | Large → faster, smoother, more memory, may generalise slightly worse. Small → noisier but often generalises better. |
| **Epochs** | Too many → overfitting (use early stopping) |
| **max_depth** (trees) | Deeper → more overfitting |
| **n_estimators / num_round** | More trees → better until it plateaus |
| **subsample / colsample_bytree** | <1 adds randomness → reduces overfitting |
| **alpha / lambda** (XGBoost) | L1 / L2 regularisation |
| **gamma / min_child_weight** | Higher → more conservative tree, less overfitting |
| **eta** (XGBoost) | Learning rate |
| **scale_pos_weight** | Handle class imbalance in XGBoost |
| **dropout** | Neural net regularisation |

> If loss is NaN or exploding → **lower the learning rate** (or check for unscaled features / exploding gradients → gradient clipping).

---

## 6. Ensembles

| Type | Idea | Example |
|---|---|---|
| **Bagging** | Train many models on random subsets in parallel, average them → **reduces variance** | Random Forest |
| **Boosting** | Train models sequentially, each fixing the previous one's errors → **reduces bias** | **XGBoost**, LightGBM, AdaBoost |
| **Stacking** | A meta-model learns to combine other models' outputs | Autopilot ensembling mode |

> "Reduce variance/overfitting" → bagging. "Improve accuracy of a weak model" → boosting.

---

## 7. Transfer learning & fine-tuning

Take a pre-trained model, re-use it on your task with far less data and compute.

- **Feature extraction** — freeze the base, train only the new output layer. Very little data.
- **Fine-tuning** — unfreeze some/all layers and train with a **low learning rate**. More data, better accuracy.
- **SageMaker JumpStart** — hundreds of pre-trained models + solution templates, deployable/fine-tunable in a few clicks. The "least effort to get a good model" answer.

---

## 8. Low-code / no-code options

| Service | What | Choose when |
|---|---|---|
| **SageMaker Autopilot (AutoML)** | Auto data prep + algorithm selection + HPO; gives you **explainability and candidate notebooks** | "Automatically find the best model", "team has limited ML expertise", still want full code visibility |
| **SageMaker Canvas** | Visual, no-code ML for business analysts | "Business analysts with no coding skills" |
| **JumpStart** | Pre-trained model zoo + fine-tuning | "Get started quickly with a proven model" |
| **Amazon Q / Bedrock** | GenAI | Text/chat/summarisation use cases |

Autopilot modes: **Ensembling** (fast, AutoGluon, <100 MB data) vs **HPO** (slower, tries hyperparameters, larger data) vs **Auto**.

---

## 9. Responsible AI basics (expanded in file 09)

- **SageMaker Clarify** — detects **bias** before training (data bias) and after training (model bias), and gives **explainability** via SHAP values.
- Bias metrics to recognise: Class Imbalance (CI), Difference in Proportions of Labels (DPL), Disparate Impact (DI), Accuracy Difference, Recall Difference.
- **Model Cards** — documented record of a model's purpose, data, metrics, risks. Governance answer.

---

## Exam traps for this chapter

- Accuracy is the wrong metric whenever the data is imbalanced.
- Recall = catch everything (medical/fraud). Precision = don't false-alarm (spam).
- AUC-PR beats AUC-ROC on heavily imbalanced data.
- Overfitting fix list: more data, regularisation, dropout, early stopping, simpler model, cross-validation.
- L1 does feature selection; L2 does not.
- XGBoost is the safest default answer for tabular classification/regression.
- "Little data + similar public task" → **transfer learning / JumpStart**, not training from scratch.
