# Unit VI — Phase 11: Model Evaluation and Performance Metrics

> **Continuing from Phase 10.** We have six algorithms. How do we know whether a model is *any good*, and which of two models is *better*? This phase covers the syllabus topic **Evaluation and performance metrics: Confusion matrix, R², RMSE, MAE, Correlation heat map**. We build every metric from its definition, compute it by hand, and confirm it against Spark. We also cover ROC/AUC, multiclass metrics, cross-validation and hyperparameter tuning.
>
> All outputs are real. In every case the hand calculation matches Spark's evaluator.

---

## Phase 11 — Roadmap

| # | Topic |
| --- | --- |
| 1 | Why evaluation matters |
| 2 | The map of metrics |
| **Part A** | **Classification metrics** |
| 3 | The confusion matrix |
| 4 | Accuracy, precision, recall, specificity, F1 |
| 5 | A hand-worked example |
| 6 | The accuracy paradox |
| 7 | Confusion matrix in Spark (students) |
| 8 | The confusion matrix heat map |
| 9 | The precision-recall trade-off and the threshold |
| 10 | ROC curve and AUC |
| 11 | Multiclass metrics: macro vs weighted |
| 12 | Spark evaluators: all `metricName` values |
| **Part B** | **Regression metrics** |
| 13 | MSE, RMSE, MAE, R², adjusted R² |
| 14 | Hand calculation and Spark |
| 15 | Choosing among them |
| **Part C** | **Clustering metrics** |
| 16 | WSSSE and silhouette |
| **Part D** | **Validation and tuning** |
| 17 | Cross-validation |
| 18 | Hyperparameter tuning with `CrossValidator` |
| 19 | `TrainValidationSplit` |
| 20 | Choosing the right metric |
| 21 | Common errors |
| 22 | Practical scenarios |
| 23 | Viva questions |
| 24 | Master revision |

---

### 1. Why evaluation matters

> **Training error tells us how well the model memorised; evaluation on unseen data tells us how well it will perform in real life.**

Three rules (Phase 2) that apply to every metric:

1. Evaluate on **test data** the model never saw.
2. Compare against a **baseline** (for example "always predict the majority class" or "always predict the mean").
3. Pick the metric that reflects the **real cost of mistakes**.

---

### 2. The map of metrics

| Task | Metrics | Spark tool |
| --- | --- | --- |
| **Classification** | Confusion matrix, accuracy, precision, recall, F1, specificity, ROC-AUC, log-loss | `MulticlassClassificationEvaluator`, `BinaryClassificationEvaluator`, `MulticlassMetrics` |
| **Regression** | MSE, **RMSE**, **MAE**, **R²**, adjusted R² | `RegressionEvaluator` |
| **Clustering** | WSSSE (cost), silhouette | `ClusteringEvaluator`, `summary.trainingCost` |
| **Relationships (data)** | **Correlation heat map** (Phase 4) | `Correlation` + seaborn |

---

## PART A — CLASSIFICATION METRICS

### 3. The confusion matrix

> **A confusion matrix is a table that compares the model's predictions with the actual classes, counting each kind of right and wrong answer.**

For **binary** classification with positive = 1 (pass) and negative = 0 (fail):

```text
                         PREDICTED
                    Negative (0)   Positive (1)
                  ┌──────────────┬──────────────┐
 ACTUAL Negative  │      TN      │      FP      │
        (0)       │ true negative│false positive│
                  ├──────────────┼──────────────┤
 ACTUAL Positive  │      FN      │      TP      │
        (1)       │false negative│ true positive│
                  └──────────────┴──────────────┘
```

| Cell | Name | Meaning | Example (disease test) |
| --- | --- | --- | --- |
| **TP** | True Positive | Predicted positive, **actually** positive ✔ | Sick person correctly flagged |
| **TN** | True Negative | Predicted negative, actually negative ✔ | Healthy person correctly cleared |
| **FP** | False Positive (**Type I error**) | Predicted positive, actually negative ✘ | Healthy person told they are sick |
| **FN** | False Negative (**Type II error**) | Predicted negative, actually positive ✘ | Sick person told they are healthy |

**Memory trick:** the **second word** (Positive/Negative) is **what the model predicted**; the **first word** (True/False) says **whether that prediction was correct**.

```text
 TP + TN = correct predictions          FP + FN = mistakes          TP + TN + FP + FN = all samples
```

**Which mistake is worse?** It depends on the problem:

| Problem | Worse error | Why |
| --- | --- | --- |
| Cancer screening | **FN** (missed cancer) | A missed case can be fatal |
| Spam filter | **FP** (real email deleted) | You lose important messages |
| Fraud detection | **FN** (missed fraud) | Money is lost |

---

### 4. Accuracy, precision, recall, specificity, F1

All are built from the four counts.

| Metric | Formula | Question it answers |
| --- | --- | --- |
| **Accuracy** | `(TP + TN) / (TP + TN + FP + FN)` | What fraction of **all** predictions were correct? |
| **Precision** | `TP / (TP + FP)` | Of everything **predicted positive**, how much really was positive? *(How trustworthy is a "positive"?)* |
| **Recall** (Sensitivity, TPR) | `TP / (TP + FN)` | Of all **actual positives**, how many did we **find**? *(How complete are we?)* |
| **Specificity** (TNR) | `TN / (TN + FP)` | Of all **actual negatives**, how many did we correctly clear? |
| **False Positive Rate** | `FP / (FP + TN)` = 1 − specificity | Of all actual negatives, how many did we wrongly flag? |
| **F1-score** | `2 · Precision · Recall / (Precision + Recall)` | A single balance of precision and recall |

**Why F1 uses the harmonic mean.** The ordinary average of precision 1.0 and recall 0.0 would be 0.5, which hides a useless model. The harmonic mean is dragged towards the **lower** value, so F1 is high only when **both** are high.

```text
 Precision = 1.0, Recall = 0.0 → average = 0.50  but  F1 = 0.0 (honest)
```

**F-beta** generalizes F1: `Fβ = (1 + β²)·P·R / (β²·P + R)`. Use β = 2 to weight recall more; β = 0.5 to weight precision more.

**Easy memory**

```text
 Precision: "when I say POSITIVE, how often am I right?"          (denominator: predicted positives)
 Recall   : "of the REAL positives, how many did I catch?"        (denominator: actual positives)
```

---

### 5. A hand-worked example

A disease test on **100 patients**: the results are

| | Predicted sick | Predicted healthy |
| --- | --- | --- |
| **Actually sick (45)** | TP = 40 | FN = 5 |
| **Actually healthy (55)** | FP = 10 | TN = 45 |

```text
 Accuracy    = (40 + 45) / 100          = 0.85
 Precision   = 40 / (40 + 10) = 40/50   = 0.80      (80% of "sick" verdicts were right)
 Recall      = 40 / (40 + 5)  = 40/45   = 0.8889    (found 88.9% of the sick people)
 Specificity = 45 / (45 + 10) = 45/55   = 0.8182
 FPR         = 10 / 55                  = 0.1818
 F1          = 2 × 0.80 × 0.8889 / (0.80 + 0.8889) = 1.4222 / 1.6889 = 0.8421
```

**Interpretation:** the test finds most sick people (recall 0.89) but 1 in 5 positive verdicts is a false alarm (precision 0.80).

---

### 6. The accuracy paradox

**Scenario:** 1000 card transactions, of which only **10 are fraud** (1%). A lazy model predicts **"genuine" for everything**.

| | Predicted fraud | Predicted genuine |
| --- | --- | --- |
| Actual fraud (10) | TP = 0 | FN = 10 |
| Actual genuine (990) | FP = 0 | TN = 990 |

```text
 Accuracy = (0 + 990) / 1000 = 0.99   ← looks excellent!
 Recall   = 0 / (0 + 10)     = 0.0    ← catches NO fraud at all
```

> **Accuracy is misleading on imbalanced data.** The useless model has 99% accuracy. Use **recall, precision, F1 and AUC** instead. This is why we always look at the confusion matrix, not accuracy alone.

---

### 7. Confusion matrix in Spark (students)

Set-up: predict `passed` with logistic regression (Phase 6) on a 70:30 split.

```python
import sys
sys.path.append(r"C:\sparklabs")
from make_data import make_students
from pyspark.ml.feature import VectorAssembler
from pyspark.ml.classification import LogisticRegression
from pyspark.sql import functions as F

df = spark.createDataFrame(make_students(),
        ["id", "hours", "attendance", "prev_marks", "final_marks", "passed"])
va = VectorAssembler(inputCols=["hours", "attendance", "prev_marks"], outputCol="features")
data = va.transform(df).select("features", F.col("passed").alias("label"))

train, test = data.randomSplit([0.7, 0.3], seed=7)
print(train.count(), test.count())

model = LogisticRegression(maxIter=100).fit(train)
pred = model.transform(test)
```

```text
133 67
```

The test set has **35 fails and 32 passes**.

**Method 1: count each (label, prediction) pair**

```python
pred.groupBy("label", "prediction").count().orderBy("label", "prediction").show()
```

```text
+-----+----------+-----+
|label|prediction|count|
+-----+----------+-----+
|    0|       0.0|   33|
|    0|       1.0|    2|
|    1|       0.0|    1|
|    1|       1.0|   31|
+-----+----------+-----+
```

**Method 2: `MulticlassMetrics` (RDD-based helper)**

```python
from pyspark.mllib.evaluation import MulticlassMetrics

rdd = pred.select("prediction", "label").rdd.map(lambda r: (float(r[0]), float(r[1])))
mm = MulticlassMetrics(rdd)
print(mm.confusionMatrix().toArray().astype(int))
```

```text
[[33  2]
 [ 1 31]]
```

(Rows = actual class, columns = predicted class.)

**Fill in the matrix**

| | Predicted fail (0) | Predicted pass (1) |
| --- | --- | --- |
| **Actual fail (0)** | **TN = 33** | **FP = 2** |
| **Actual pass (1)** | **FN = 1** | **TP = 31** |

**Hand-calculated metrics (positive class = pass)**

```text
 Accuracy    = (31 + 33) / 67 = 64/67   = 0.9552
 Precision   = 31 / (31 + 2)  = 31/33   = 0.9394
 Recall      = 31 / (31 + 1)  = 31/32   = 0.9688
 Specificity = 33 / (33 + 2)  = 33/35   = 0.9429
 F1          = 2×0.9394×0.9688 / (0.9394 + 0.9688) = 0.9538
```

**Spark's own numbers**

```python
from pyspark.ml.evaluation import MulticlassClassificationEvaluator as MCE

print("accuracy          :", round(MCE(metricName="accuracy").evaluate(pred), 4))
print("precision (pass)  :", round(MCE(metricName="precisionByLabel", metricLabel=1.0).evaluate(pred), 4))
print("recall (pass)     :", round(MCE(metricName="recallByLabel", metricLabel=1.0).evaluate(pred), 4))
print("F1 (pass)         :", round(MCE(metricName="fMeasureByLabel", metricLabel=1.0).evaluate(pred), 4))
print("weighted F1       :", round(MCE(metricName="f1").evaluate(pred), 4))
```

```text
accuracy          : 0.9552
precision (pass)  : 0.9394
recall (pass)     : 0.9688
F1 (pass)         : 0.9538
weighted F1       : 0.9552
```

✔ **Every hand value matches Spark.**

> **Watch out:** Spark's plain `metricName="f1"` is the **weighted average over both classes** (0.9552 here), *not* the F1 of the positive class (0.9538). For the positive class, use `fMeasureByLabel` with `metricLabel=1.0`.

---

### 8. The confusion matrix heat map

A heat map makes the matrix visual: dark diagonal = good.

```python
import numpy as np
import matplotlib.pyplot as plt
import seaborn as sns

cm = np.array([[33, 2],
               [1, 31]])

plt.figure(figsize=(5, 4))
sns.heatmap(cm, annot=True, fmt="d", cmap="Blues",
            xticklabels=["Pred fail (0)", "Pred pass (1)"],
            yticklabels=["Actual fail (0)", "Actual pass (1)"])
plt.xlabel("Predicted")
plt.ylabel("Actual")
plt.title("Confusion matrix")
plt.tight_layout()
plt.savefig("C:/sparklabs/confusion_matrix.png")
plt.show()
```

```text
                 Pred fail   Pred pass
 Actual fail   ┌   33 ████    2 ·     ┐
 Actual pass   └    1 ·      31 ████   ┘      (dark cells on the diagonal = correct predictions)
```

| Argument | Meaning |
| --- | --- |
| `annot=True`, `fmt="d"` | Print each count as an integer in its cell |
| `cmap="Blues"` | Darker = larger count |
| `xticklabels`, `yticklabels` | Label the columns (predicted) and rows (actual) |

**Reading it:** the heat is concentrated on the **diagonal** (33 and 31), which is where correct predictions sit. The faint off-diagonal cells (2 and 1) are the errors.

*(Phase 4 used a heat map for correlations; this one is the same idea applied to counts. Both are in your syllabus list.)*

---

### 9. The precision-recall trade-off and the threshold

A classifier gives a **probability**; we turn it into a class with a **threshold** (default 0.5). Moving the threshold shifts errors between FP and FN:

```python
for t in (0.3, 0.5, 0.7, 0.9):
    p = LogisticRegression(maxIter=100, threshold=t).fit(train).transform(test)
    # compute precision and recall (class 1) from the counts
```

```text
 threshold   precision(pass)   recall(pass)
    0.3          0.912             0.969
    0.5          0.939             0.969
    0.7          0.967             0.906
    0.9          0.963             0.812
```

```text
 Lower threshold  → predict "positive" more easily → FEWER false negatives (recall ↑)
                                                    MORE false positives  (precision ↓)
 Higher threshold → predict "positive" only when very sure → precision ↑ , recall ↓
```

| Goal | Choose |
| --- | --- |
| Never miss a positive (cancer, fraud) | **Low threshold** → high recall |
| Avoid false alarms (spam filter) | **High threshold** → high precision |
| Balance | Maximize **F1** |

**Precision and recall cannot both be maximized freely**: improving one usually costs the other.

---

### 10. ROC curve and AUC

Instead of fixing one threshold, the **ROC curve** shows performance at **all** thresholds.

**ROC = Receiver Operating Characteristic curve:** plot **TPR (recall)** on the y-axis against **FPR** on the x-axis, one point per threshold.

```text
 TPR
 1.0 │        ┌────────●───●          ← a good model hugs the top-left corner
     │      ●─┘
     │    ╱
     │  ╱             ╱ ← diagonal = random guessing (AUC 0.5)
     │╱         ╱
 0.0 └─────────────────── FPR
     0.0               1.0
```

**AUC = Area Under the ROC Curve**, from 0 to 1:

| AUC | Meaning |
| --- | --- |
| **1.0** | Perfect separation |
| 0.9 to 1.0 | Excellent |
| 0.8 to 0.9 | Good |
| 0.7 to 0.8 | Fair |
| **0.5** | No better than random |
| < 0.5 | Worse than random (predictions inverted) |

**Meaning of AUC (very useful for viva):** *AUC is the probability that the model gives a randomly chosen positive example a higher score than a randomly chosen negative example.*

**Hand-worked example.** Six samples with model scores: positives scored **0.9, 0.8, 0.4**; negatives scored **0.7, 0.3, 0.2**.

*ROC points* (lower the threshold step by step):

| Threshold | TPR (positives found ÷ 3) | FPR (negatives flagged ÷ 3) |
| --- | --- | --- |
| 1.0 | 0 | 0 |
| 0.9 | 0.333 | 0 |
| 0.8 | 0.667 | 0 |
| 0.7 | 0.667 | 0.333 |
| 0.4 | 1.0 | 0.333 |
| 0.3 | 1.0 | 0.667 |
| 0.2 | 1.0 | 1.0 |

*AUC by pairs:* compare every positive with every negative (3 × 3 = 9 pairs) and count how often the positive scores higher:

```text
 0.9 beats 0.7, 0.3, 0.2   → 3
 0.8 beats 0.7, 0.3, 0.2   → 3
 0.4 beats 0.3, 0.2 (not 0.7) → 2
 AUC = 8 / 9 = 0.8889
```

**Spark:**

```python
from pyspark.ml.evaluation import BinaryClassificationEvaluator
# the evaluator reads the score from the "rawPrediction" column
print(BinaryClassificationEvaluator(metricName="areaUnderROC").evaluate(scored_df))
```

```text
0.8888888888888888   (our pairs-counting result 8/9 ✔)
```

**AUC for our students**

```python
auc = BinaryClassificationEvaluator(labelCol="label", metricName="areaUnderROC").evaluate(pred)
print(round(auc, 4))
```

```text
0.9866
```

**AUC 0.987:** a randomly chosen passing student gets a higher pass-score than a randomly chosen failing student 98.7% of the time. The ROC curve has 135 points; the first are (0.0, 0.0), (0.0, 0.015), (0.0, 0.029): it climbs straight up the TPR axis (finding positives with **no** false alarms) before bending right.

```python
roc = model.summary.roc          # DataFrame of (FPR, TPR) points  (training data)
```

| ROC-AUC | Precision-Recall AUC |
| --- | --- |
| Threshold-independent, good general measure | More informative when positives are **rare** |
| `metricName="areaUnderROC"` | `metricName="areaUnderPR"` |

> **Important:** `BinaryClassificationEvaluator` should receive the **score** (`rawPrediction` or `probability`), not the 0/1 `prediction`. If you give it hard predictions, AUC collapses to a single threshold and looks worse.

---

### 11. Multiclass metrics: macro vs weighted

For **3 or more classes** the confusion matrix is K × K, and precision/recall are computed **per class** ("one class vs the rest").

**Example:** convert marks into three grade classes: **0 = below 60, 1 = 60 to 74, 2 = 75 and above** (counts 56, 85, 59 in our 200 students). Train multinomial logistic regression on a 70:30 split (`seed=11`, 56 test rows).

**Confusion matrix** (rows = actual, columns = predicted):

```text
                  Pred 0   Pred 1   Pred 2
 Actual 0 (<60)  │  16        4        0   │  = 20
 Actual 1 (60-74)│   1       18        3   │  = 22
 Actual 2 (≥75)  │   0        1       13   │  = 14
                   17       23       16        total 56
```

**Per-class precision and recall.** For class k: `precision = diagonal / column sum`, `recall = diagonal / row sum`.

| Class | Precision | Recall | F1 | Support (actual count) |
| --- | --- | --- | --- | --- |
| 0 | 16/17 = **0.9412** | 16/20 = **0.8000** | 0.8649 | 20 |
| 1 | 18/23 = **0.7826** | 18/22 = **0.8182** | 0.8000 | 22 |
| 2 | 13/16 = **0.8125** | 13/14 = **0.9286** | 0.8667 | 14 |

**Overall accuracy** = diagonal sum / total = (16 + 18 + 13) / 56 = 47/56 = **0.8393**.

**Two ways to average the per-class scores**

| Average | How | Our result (precision / recall / F1) |
| --- | --- | --- |
| **Macro** | Plain average of the per-class scores (all classes count equally) | 0.8454 / 0.8489 / 0.8438 |
| **Weighted** | Average **weighted by class size (support)** | 0.8467 / 0.8393 / 0.8398 |
| **Micro** | Pool all TP, FP, FN, then compute (equals accuracy for single-label problems) | 0.8393 |

```text
 Macro precision    = (0.9412 + 0.7826 + 0.8125) / 3 = 0.8454
 Weighted precision = (20×0.9412 + 22×0.7826 + 14×0.8125) / 56 = 0.8467
```

**Spark** (`MulticlassClassificationEvaluator`):

```text
 accuracy 0.8393    f1 (weighted) 0.8398    weightedPrecision 0.8467    weightedRecall 0.8393
 hammingLoss 0.1607    logLoss 0.3303
```

✔ All agree with the hand values. (`hammingLoss` = 1 − accuracy = 0.1607 for single-label problems.)

**When to use which average:** *macro* when every class matters equally (including rare ones); *weighted* when class sizes reflect importance.

---

### 12. Spark evaluators: all `metricName` values

| Evaluator | Task | `metricName` options (default first) |
| --- | --- | --- |
| **`BinaryClassificationEvaluator`** | Binary | `areaUnderROC`, `areaUnderPR` |
| **`MulticlassClassificationEvaluator`** | Binary or multiclass | `f1`, `accuracy`, `weightedPrecision`, `weightedRecall`, `weightedTruePositiveRate`, `weightedFalsePositiveRate`, `weightedFMeasure`, `truePositiveRateByLabel`, `falsePositiveRateByLabel`, `precisionByLabel`, `recallByLabel`, `fMeasureByLabel`, `logLoss`, `hammingLoss` |
| **`RegressionEvaluator`** | Regression | `rmse`, `mse`, `r2`, `mae`, `var` |
| **`ClusteringEvaluator`** | Clustering | `silhouette` |

```python
evaluator = MCE(labelCol="label", predictionCol="prediction", metricName="accuracy")
score = evaluator.evaluate(pred)
```

Use `metricLabel=1.0` together with the `...ByLabel` metrics.

---

## PART B — REGRESSION METRICS

### 13. MSE, RMSE, MAE, R², adjusted R²

For `n` samples with actual `yᵢ`, predicted `ŷᵢ` and mean `ȳ`:

| Metric | Formula | Meaning | Better |
| --- | --- | --- | --- |
| **MSE** | `Σ (yᵢ − ŷᵢ)² / n` | Mean squared error | lower |
| **RMSE** | `√MSE` | Typical error **in label units**; punishes large errors | lower |
| **MAE** | `Σ |yᵢ − ŷᵢ| / n` | Mean absolute error; every error counts proportionally | lower |
| **R²** | `1 − SSE/SST` | Fraction of variance explained | closer to 1 |
| **Adjusted R²** | `1 − (1 − R²)(n − 1)/(n − k − 1)` | R² penalised for each extra feature (k features) | closer to 1 |

with

```text
 SSE = Σ (yᵢ − ŷᵢ)²          (unexplained: error of the model)
 SST = Σ (yᵢ − ȳ)²           (total: error of always predicting the mean ȳ)
```

**Understanding R² through SST and SSE:**

```text
 R² = 1 − SSE/SST

 R² = 1    → SSE = 0           → perfect predictions
 R² = 0    → SSE = SST         → no better than predicting the mean
 R² < 0    → SSE > SST         → WORSE than predicting the mean
```

**Properties to remember**

| Fact | Detail |
| --- | --- |
| RMSE ≥ MAE | Always. A big gap means a few large errors |
| RMSE and MAE share the label's **units** | RMSE = 2.8 marks means about 2.8 marks of error |
| **R² has no units** | Compare across datasets with different scales |
| Adding any feature never lowers (train) R² | That is why adjusted R² exists |
| R² can be negative on test data | A model worse than the mean baseline |

---

### 14. Hand calculation and Spark

**Tiny example** (from Phase 5): actual `[40, 50, 55, 65, 75]`, predicted `[40, 48.5, 57, 65.5, 74]`.

| y | ŷ | y − ŷ | (y − ŷ)² | \|y − ŷ\| | (y − ȳ)² (ȳ = 57) |
| --- | --- | --- | --- | --- | --- |
| 40 | 40 | 0 | 0 | 0 | 289 |
| 50 | 48.5 | 1.5 | 2.25 | 1.5 | 49 |
| 55 | 57 | −2 | 4 | 2 | 4 |
| 65 | 65.5 | −0.5 | 0.25 | 0.5 | 64 |
| 75 | 74 | 1 | 1 | 1 | 324 |
| | | | **SSE = 7.5** | **Σ = 5.0** | **SST = 730** |

```text
 MSE  = 7.5 / 5     = 1.5
 RMSE = √1.5        = 1.2247
 MAE  = 5.0 / 5     = 1.0
 R²   = 1 − 7.5/730 = 0.9897
```

**Students: test set of 35** (linear regression, Phase 5). Hand-compute with numpy and compare with Spark:

```python
import numpy as np
from pyspark.ml.regression import LinearRegression
from pyspark.ml.evaluation import RegressionEvaluator

rdata = va.transform(df).select("features", F.col("final_marks").alias("label"))
rtrain, rtest = rdata.randomSplit([0.8, 0.2], seed=42)

lrm = LinearRegression().fit(rtrain)
pr = lrm.transform(rtest)

# --- by hand, with numpy ---
y  = np.array([r[0] for r in pr.select("label").collect()])
yh = np.array([r[0] for r in pr.select("prediction").collect()])
mse  = np.mean((y - yh) ** 2)
rmse = mse ** 0.5
mae  = np.mean(np.abs(y - yh))
r2   = 1 - np.sum((y - yh) ** 2) / np.sum((y - y.mean()) ** 2)
print("HAND  mse %.4f rmse %.4f mae %.4f r2 %.4f" % (mse, rmse, mae, r2))

# --- with Spark ---
ev = RegressionEvaluator(labelCol="label")
print("SPARK", {m: round(ev.setMetricName(m).evaluate(pr), 4) for m in ("mse", "rmse", "mae", "r2", "var")})
```

```text
HAND  mse 8.0730 rmse 2.8413 mae 2.2595 r2 0.9291
SPARK {'mse': 8.073, 'rmse': 2.8413, 'mae': 2.2595, 'r2': 0.9291, 'var': 101.4623}
```

✔ **Hand and Spark agree exactly.** (`var` is the *explained variance*, an extra metric.)

**Interpretation:** predictions are off by about **2.84 marks** (RMSE, slightly more than the average miss of 2.26 marks, so a few larger errors exist) and the model explains **92.9%** of the variance of the final marks.

**Adjusted R² (training summary)**

```python
s = lrm.summary
print(round(s.r2, 4), round(s.r2adj, 4))
```

```text
0.9442 0.9432
```

With n = 165 and k = 3: `1 − (1 − 0.9442)(165 − 1)/(165 − 3 − 1) = 1 − 0.0558 × 164/161 = 0.9432` ✔. The adjustment is tiny because the three features are all useful and n is large.

**Comparing models on the same test set** (final marks)

| Model | RMSE | R² |
| --- | --- | --- |
| **Linear Regression** | **2.84** | **0.929** |
| Decision Tree Regressor (Phase 7) | 6.30 | 0.652 |

Lower RMSE and higher R² both say linear regression is the better model for this data.

---

### 15. Choosing among them

| Situation | Prefer |
| --- | --- |
| Large errors are especially costly | **RMSE** (squares errors) |
| Outliers should not dominate | **MAE** (robust) |
| Compare across different label scales | **R²** (unit-free) |
| Report "typical error" to a non-technical audience | **MAE** or **RMSE** in original units |
| Many features | **Adjusted R²** |
| Tuning (choose a model) | The same metric you care about on a **validation set** |

---

## PART C — CLUSTERING METRICS

### 16. WSSSE and silhouette

Clustering has **no labels**, so there is no "accuracy". We use **internal** measures (Phase 8):

| Metric | Meaning | Better |
| --- | --- | --- |
| **WSSSE / WCSS / inertia** (`model.summary.trainingCost`) | Sum of squared distances to the cluster centroids | Lower (but always falls as K grows) |
| **Silhouette** (`ClusteringEvaluator`) | `(b − a) / max(a, b)` averaged over points, range −1 to +1 | **Higher** |

Recap of our results: 7-point example silhouette **0.8315** (WSSSE 8.525); customers with K = 4: WSSSE **271.8**, silhouette **0.988**. Use the elbow of WSSSE and the peak of silhouette to choose K.

---

## PART D — VALIDATION AND TUNING

### 17. Cross-validation

One train/test split can be lucky or unlucky (our two splits gave test accuracies of 0.914 and 0.955 for the *same* model!). **k-fold cross-validation** averages over several splits.

```text
 3-fold example   [ F1 | F2 | F3 ]

 Round 1: train F2+F3 , validate on F1 → score₁
 Round 2: train F1+F3 , validate on F2 → score₂
 Round 3: train F1+F2 , validate on F3 → score₃

 CV score = (score₁ + score₂ + score₃) / 3
```

**Three data roles**

| Set | Purpose | Used |
| --- | --- | --- |
| **Training** | Fit the model's parameters | Many times |
| **Validation** (folds in CV) | **Choose hyperparameters** | Many times |
| **Test** | Final, honest performance estimate | **Once** |

> The test set must stay **untouched** during tuning. If you tune on it, you leak information and the final score is optimistic.

---

### 18. Hyperparameter tuning with `CrossValidator`

**Goal:** choose `regParam` and `elasticNetParam` for logistic regression, using a **Pipeline** (scaler + model) and 3-fold CV on the **training** data.

```python
from pyspark.ml import Pipeline
from pyspark.ml.feature import StandardScaler
from pyspark.ml.classification import LogisticRegression
from pyspark.ml.evaluation import BinaryClassificationEvaluator
from pyspark.ml.tuning import ParamGridBuilder, CrossValidator

train, test = data.randomSplit([0.8, 0.2], seed=42)

scaler = StandardScaler(inputCol="features", outputCol="sf")
lr = LogisticRegression(featuresCol="sf", maxIter=100)
pipe = Pipeline(stages=[scaler, lr])

grid = (ParamGridBuilder()
        .addGrid(lr.regParam, [0.0, 0.01, 0.1])
        .addGrid(lr.elasticNetParam, [0.0, 0.5])
        .build())
print("combinations:", len(grid))

cv = CrossValidator(estimator=pipe,
                    estimatorParamMaps=grid,
                    evaluator=BinaryClassificationEvaluator(metricName="areaUnderROC"),
                    numFolds=3, seed=42)

cv_model = cv.fit(train)
```

```text
combinations: 6
```

**Results: the average AUC of each combination**

```python
for params, score in zip(grid, cv_model.avgMetrics):
    print(round(score, 4))
```

| # | `regParam` | `elasticNetParam` | Average CV AUC |
| --- | --- | --- | --- |
| 1 | 0.0 | 0.0 | 0.9933 |
| 2 | 0.0 | 0.5 | 0.9933 |
| **3** | **0.01** | **0.0** | **0.9951** ← best |
| **4** | 0.01 | 0.5 | 0.9951 |
| 5 | 0.1 | 0.0 | 0.9947 |
| 6 | 0.1 | 0.5 | 0.9878 |

(`elasticNetParam` has no effect when `regParam = 0`, so rows 1 and 2 tie.)

**Pick the winner and test it once**

```python
best = cv_model.bestModel.stages[1]          # the LogisticRegressionModel inside the best PipelineModel
print("best regParam:", best.getRegParam(), " elasticNet:", best.getElasticNetParam())

test_auc = BinaryClassificationEvaluator(metricName="areaUnderROC").evaluate(cv_model.transform(test))
print("test AUC:", round(test_auc, 4))
```

```text
best regParam: 0.01  elasticNet: 0.0
test AUC: 0.984
```

**The cost of tuning:** 6 combinations × 3 folds = **18 model fits**, plus 1 final refit on all of the training data. Keep grids small on big data (and use `parallelism=` in `CrossValidator` to train several models at once).

**Save the tuned model**

```python
cv_model.write().overwrite().save("C:/sparklabs/models/cv_logreg")
from pyspark.ml.tuning import CrossValidatorModel
same = CrossValidatorModel.load("C:/sparklabs/models/cv_logreg")
```

| `CrossValidatorModel` attribute | Meaning |
| --- | --- |
| `bestModel` | The fitted best model (refit on all training data) |
| `avgMetrics` | Average metric for each parameter combination |
| `transform(df)` | Predict with the best model |

---

### 19. `TrainValidationSplit`

A cheaper alternative that splits the training data **once** (for example 75% / 25%) instead of k times.

```python
from pyspark.ml.tuning import TrainValidationSplit

tvs = TrainValidationSplit(estimator=pipe, estimatorParamMaps=grid,
                           evaluator=BinaryClassificationEvaluator(),
                           trainRatio=0.75, seed=42)
tvs_model = tvs.fit(train)
print([round(x, 3) for x in tvs_model.validationMetrics])
```

```text
[0.97, 0.97, 0.984, 0.984, 0.984, 0.964]
```

It also prefers `regParam = 0.01` and `0.1` (with elastic net 0), agreeing with cross-validation, but only needs 6 fits instead of 18.

| `CrossValidator` | `TrainValidationSplit` |
| --- | --- |
| k folds | One split |
| More reliable | Less reliable (one lucky/unlucky split) |
| k times slower | Fastest |
| Small/medium data | Very large data |

---

### 20. Choosing the right metric

| Scenario | Primary metric | Why |
| --- | --- | --- |
| Predict house price | RMSE / MAE | Error in rupees |
| Balanced pass/fail | Accuracy + F1 | Classes equal |
| Fraud (1% positives) | **Recall**, precision, **PR-AUC** | Accuracy misleading |
| Cancer screening | **Recall** | Missing a case is worst |
| Spam filter | **Precision** | Losing real email is worst |
| Ranking quality of scores | **ROC-AUC** | Threshold independent |
| Rare-class multiclass | **Macro F1** | Treats classes equally |
| Customer segments | **Silhouette** + business sense | No labels |

---

### 21. Common errors

| Symptom | Cause | Fix |
| --- | --- | --- |
| Accuracy 99% but model is useless | Imbalanced data | Look at the confusion matrix, recall, F1 |
| AUC unexpectedly low | Passed `prediction` (0/1) instead of the score | Use `rawPrediction` / `probability` |
| Mixed-up TP/FP | Confusing rows/columns | Rows = actual, columns = predicted |
| `f1` metric disagrees with hand F1 | Spark's `f1` is **weighted** over classes | `fMeasureByLabel` with `metricLabel` |
| Evaluator cannot find `label` | Different column names | Set `labelCol` and `predictionCol` |
| Evaluating on the **training** set | Over-optimistic | Evaluate on test data |
| Tuning on the test set | Leakage | Tune with CV on the training set only |
| Comparing RMSE across different label scales | RMSE has units | Use R² or normalised error |
| Large negative R² | Model worse than the mean | Check features, leakage and scaling |
| Huge grid is very slow | Combinations × folds | Smaller grid, `parallelism`, `TrainValidationSplit` |
| `bestModel` coefficients not found | It is a `PipelineModel` | Use `bestModel.stages[i]` |
| Different results every run | Missing seeds | `seed=42` in splits and CV |

---

### 22. Phase 11 — practical scenarios

1. From TP = 40, FP = 10, FN = 5, TN = 45, compute **accuracy, precision, recall, specificity and F1**.
2. Show with the **fraud example** why 99% accuracy can be worthless.
3. Build the **confusion matrix** for the student classifier; compute the metrics by hand and with Spark.
4. Draw the **confusion-matrix heat map** with seaborn.
5. Sweep the **threshold** (0.3, 0.5, 0.7, 0.9) and tabulate precision and recall.
6. By hand, compute the **AUC for the six-score example** (8/9) and list the ROC points.
7. Compute **AUC** for the students with `BinaryClassificationEvaluator`.
8. For the **3-class confusion matrix**, compute per-class precision/recall and the macro and weighted averages.
9. Compute **MSE, RMSE, MAE and R²** by hand for the 5-point regression and confirm with Spark.
10. Compute **adjusted R²** for n = 165, k = 3, R² = 0.9442.
11. Run a **CrossValidator** with a 3 × 2 grid; report the best parameters and test AUC.
12. Compare **`CrossValidator` and `TrainValidationSplit`** on speed and reliability.

---

### 23. Phase 11 — viva questions and answers

#### Q1. What is a confusion matrix?

A table comparing actual and predicted classes, counting true positives, true negatives, false positives and false negatives.

#### Q2. Define TP, TN, FP and FN.

TP: predicted positive and actually positive; TN: predicted negative and actually negative; FP: predicted positive but actually negative (Type I error); FN: predicted negative but actually positive (Type II error).

#### Q3. Give the formulas for accuracy, precision and recall.

Accuracy = (TP + TN)/(TP + TN + FP + FN); Precision = TP/(TP + FP); Recall = TP/(TP + FN).

#### Q4. What is the F1-score and why the harmonic mean?

`2PR/(P + R)`; the harmonic mean is pulled towards the smaller value, so F1 is high only if both precision and recall are high.

#### Q5. What is specificity?

TN/(TN + FP), the fraction of actual negatives correctly identified (1 − FPR).

#### Q6. Why is accuracy misleading on imbalanced data?

A model that always predicts the majority class gets high accuracy while missing every minority case.

#### Q7. When do you favour recall over precision, and vice versa?

Recall when missing a positive is costly (disease, fraud); precision when false alarms are costly (spam filtering).

#### Q8. What does the threshold do?

It converts probabilities into classes; lowering it raises recall and lowers precision, raising it does the reverse.

#### Q9. What is the ROC curve?

A plot of true positive rate against false positive rate across all thresholds.

#### Q10. What is AUC and what does it mean?

The area under the ROC curve; it equals the probability that a random positive receives a higher score than a random negative. 0.5 is random, 1.0 is perfect.

#### Q11. Which is better for imbalanced data, ROC-AUC or PR-AUC?

PR-AUC is usually more informative when positives are rare.

#### Q12. How are precision and recall computed for multiclass problems?

Per class (that class vs the rest), then averaged as macro (equal weight), weighted (by support) or micro (pooled).

#### Q13. Define MSE, RMSE and MAE.

MSE: mean squared error; RMSE: its square root in label units; MAE: mean absolute error.

#### Q14. Why prefer RMSE or MAE in different situations?

RMSE penalises large errors more; MAE is more robust to outliers and easier to interpret.

#### Q15. Define R² and what values mean.

`R² = 1 − SSE/SST`, the proportion of variance explained: 1 is perfect, 0 equals predicting the mean, negative is worse than the mean.

#### Q16. What is adjusted R²?

R² adjusted for the number of predictors, `1 − (1 − R²)(n − 1)/(n − k − 1)`, so useless features do not inflate it.

#### Q17. Can RMSE be smaller than MAE?

No. RMSE ≥ MAE always.

#### Q18. What is a correlation heat map and a confusion-matrix heat map?

Colour-coded matrices: the first shows pairwise feature correlations, the second shows counts of actual vs predicted classes.

#### Q19. What is k-fold cross-validation?

Splitting data into k folds, training on k − 1 and validating on the remaining one in turn, then averaging the k scores.

#### Q20. What is the difference between validation and test sets?

The validation set (or CV folds) is used to tune hyperparameters; the test set is kept untouched for one final unbiased evaluation.

#### Q21. How does `CrossValidator` work in Spark?

For each parameter combination from a `ParamGridBuilder`, it runs k-fold CV with the evaluator, picks the best average metric and refits the best model on all training data.

#### Q22. How many models does a 3-fold CV over a 6-combination grid train?

6 × 3 = 18, plus one final refit of the best model.

---

### 24. Phase 11 — final master recap

```text
CONFUSION MATRIX      rows = actual, columns = predicted      [[TN FP],[FN TP]]
ACCURACY   (TP+TN)/all          PRECISION  TP/(TP+FP)          RECALL (TPR)  TP/(TP+FN)
SPECIFICITY TN/(TN+FP)          FPR  FP/(FP+TN)               F1 = 2PR/(P+R)  (harmonic mean)
HAND EXAMPLE   TP40 FP10 FN5 TN45 → acc 0.85 · prec 0.80 · rec 0.8889 · spec 0.8182 · F1 0.8421
STUDENTS       [[33,2],[1,31]] → acc 0.9552 · prec 0.9394 · rec 0.9688 · F1 0.9538 · AUC 0.9866
THRESHOLD      lower → recall ↑ precision ↓ ; higher → precision ↑ recall ↓
ROC / AUC      TPR vs FPR ; AUC = P(score(pos) > score(neg)) ; example 8/9 = 0.8889
MULTICLASS     macro (equal) · weighted (by support) · micro (= accuracy)
REGRESSION     MSE = Σ(y−ŷ)²/n · RMSE = √MSE · MAE = Σ|y−ŷ|/n · R² = 1 − SSE/SST · adjR² = 1−(1−R²)(n−1)/(n−k−1)
STUDENTS REG   MSE 8.073 · RMSE 2.8413 · MAE 2.2595 · R² 0.9291
CLUSTERING     WSSSE (lower) · silhouette (higher)
TUNING         k-fold CV · ParamGridBuilder · CrossValidator (6×3=18 fits) · TrainValidationSplit
HEAT MAPS      correlation (Phase 4) · confusion matrix (this phase) via sns.heatmap
```

**Seven facts to memorise**

1. Confusion matrix: **second word = prediction**, first word = correct or not.
2. **Precision** = of predicted positives; **recall** = of actual positives.
3. **Accuracy lies** on imbalanced data; use recall/precision/F1/AUC.
4. **AUC** = chance a random positive outranks a random negative.
5. **RMSE ≥ MAE**; **R²** is unit-free and can be negative.
6. Spark's `f1` is **weighted**; use `fMeasureByLabel` for one class.
7. **Tune on validation (CV), test once.**

### Phase 11 completed

---

> **Next → Phase 12: Complete Practical Integration** (end-to-end regression, classification and clustering projects with Pipelines, a head-to-head algorithm comparison, the "question → algorithm" method, ten exam-style scenarios, saving models, and troubleshooting). File: `12_Unit6_Complete_Practical_Integration.md`
