# Unit VI — Phase 12: Complete Practical Integration

> **Continuing from Phase 11.** Every tool is now in your hands: data understanding, six algorithms, the metrics and tuning. In this phase we **combine everything** into complete, exam-ready projects built with **Pipelines**. We compare all the classifiers head to head (with a baseline), tune a model with cross-validation, segment customers, save and reuse models, and train the *question → algorithm* method for unseen practicals.
>
> **Everything in this phase was run for real** in PySpark 3.5 (Python). The tables are actual results. They also teach an honest lesson: on a tiny test set, rankings are noisy, so we always look at *several* metrics and a baseline.

---

## Phase 12 — Roadmap

| # | Topic |
| --- | --- |
| 1 | The end-to-end ML blueprint |
| 2 | Lab setup checklist |
| 3 | Project A: Predict final marks (regression pipeline) |
| 4 | Project B: Pass/fail: compare all classifiers |
| 5 | Project B2: tune a decision tree with cross-validation |
| 6 | Project C: Customer segmentation (clustering pipeline) |
| 7 | Project D: Spam filter (text pipeline) |
| 8 | Saving, loading and using a model |
| 9 | Using a saved model on streaming data (concept) |
| 10 | Algorithm selection guide |
| 11 | The "question → algorithm" method |
| 12 | Ten exam-style scenarios |
| 13 | Troubleshooting master table |
| 14 | Master template |

---

### 1. The end-to-end ML blueprint

```text
  1 LOAD            spark.createDataFrame(...) / spark.read.csv(...)
  2 UNDERSTAND      describe · null counts · groupBy · correlation heat map          (Phase 4)
  3 PREPARE         Imputer → StringIndexer → OneHotEncoder → VectorAssembler → StandardScaler   (Phase 3)
  4 SPLIT           train / test  (randomSplit, seed)                                 (Phase 2)
  5 BUILD PIPELINE  stages = [prep stages..., algorithm]
  6 FIT             pipeline.fit(train)
  7 PREDICT         model.transform(test)
  8 EVALUATE        confusion matrix · accuracy/F1/AUC · RMSE/MAE/R²                  (Phase 11)
  9 TUNE            ParamGridBuilder + CrossValidator
 10 SAVE & USE      model.write().overwrite().save(path) ; PipelineModel.load(path) ; predict new data
```

**Golden rules**

1. Fit on **train**, evaluate on **test**. Tune with CV on train only.
2. Always compare with a **baseline** (majority class / mean).
3. Look at **more than one metric**.
4. Set **seeds** (`seed=42`) so results repeat.
5. Keep preprocessing **inside the Pipeline**, so it is learned from training data only and reused for new data.

---

### 2. Lab setup checklist

| ✔ | Step | Command / check |
| --- | --- | --- |
| ☐ | Python and Spark OK | `python --version` → 3.10.11 ; `spark.version` → 3.5.9 |
| ☐ | Libraries installed | `pip install numpy pandas matplotlib seaborn` |
| ☐ | Work folder | `C:\sparklabs` containing `make_data.py` |
| ☐ | Start PySpark | `pyspark` (or a script with `spark-submit`) |
| ☐ | Import the dataset | `sys.path.append(r"C:\sparklabs")` then `from make_data import make_students` |
| ☐ | Reduce noise | `spark.sparkContext.setLogLevel("ERROR")` |
| ☐ | Faster on a laptop | `spark.conf.set("spark.sql.shuffle.partitions", "4")` |

**Standard imports used in this phase**

```python
import sys
sys.path.append(r"C:\sparklabs")
from make_data import make_students

from pyspark.ml import Pipeline, PipelineModel
from pyspark.ml.feature import VectorAssembler, StandardScaler, Tokenizer, CountVectorizer
from pyspark.ml.regression import LinearRegression
from pyspark.ml.classification import (LogisticRegression, DecisionTreeClassifier, LinearSVC,
                                       NaiveBayes, RandomForestClassifier)
from pyspark.ml.clustering import KMeans
from pyspark.ml.evaluation import (RegressionEvaluator, MulticlassClassificationEvaluator,
                                   BinaryClassificationEvaluator, ClusteringEvaluator)
from pyspark.ml.tuning import ParamGridBuilder, CrossValidator
from pyspark.sql import functions as F

df = spark.createDataFrame(make_students(),
        ["id", "hours", "attendance", "prev_marks", "final_marks", "passed"])
feats = ["hours", "attendance", "prev_marks"]
```

---

### 3. Project A: Predict final marks (regression pipeline)

**Goal:** predict `final_marks` from study hours, attendance and previous marks, with **assemble → scale → linear regression** in one Pipeline.

```python
d = df.select(*feats, F.col("final_marks").alias("label"))
train, test = d.randomSplit([0.8, 0.2], seed=42)

pipeline = Pipeline(stages=[
    VectorAssembler(inputCols=feats, outputCol="features"),
    StandardScaler(inputCol="features", outputCol="scaled", withMean=True, withStd=True),
    LinearRegression(featuresCol="scaled", labelCol="label")
])

model = pipeline.fit(train)          # learns scaler statistics AND regression weights from train only
pred = model.transform(test)

ev = RegressionEvaluator(labelCol="label")
for metric in ("rmse", "mae", "r2"):
    print(metric, round(ev.setMetricName(metric).evaluate(pred), 4))
```

```text
rmse 2.8413
mae 2.2595
r2 0.9291
```

✔ **Identical to Phase 5** (RMSE 2.8413, R² 0.9291): scaling does not change a linear model's predictions, but it makes the coefficients **comparable**.

**Standardized coefficients: which feature matters most?**

```python
lr_model = model.stages[-1]
print([round(float(c), 4) for c in lr_model.coefficients], round(float(lr_model.intercept), 4))
```

```text
[10.8374, 3.7273, 5.1498]   69.0255
```

| Feature | Coefficient per **1 standard deviation** | Meaning |
| --- | --- | --- |
| hours | **10.84** | A 1-SD increase in study hours adds ~10.8 marks |
| prev_marks | 5.15 | ~5.1 marks |
| attendance | 3.73 | ~3.7 marks |

Now that all features are on the same scale, the comparison is fair: **hours matters most**, then previous marks, then attendance. (The intercept 69.03 is the predicted marks for an *average* student, since every feature is centred at 0.)

**Predict for new students**

```python
new = spark.createDataFrame([(6.0, 80.0, 60.0), (2.0, 60.0, 45.0), (9.0, 95.0, 85.0)], feats)
model.transform(new).select(*feats, "prediction").show()
```

```text
+-----+----------+----------+------------------+
|hours|attendance|prev_marks|        prediction|
+-----+----------+----------+------------------+
|  6.0|      80.0|      60.0| 70.47            |
|  2.0|      60.0|      45.0| 44.16            |
|  9.0|      95.0|      85.0| 94.87            |
+-----+----------+----------+------------------+
```

(Predictions rounded to 2 decimals.) Note that the Pipeline accepted **raw columns**: it assembles and scales them itself. This is the main benefit of Pipelines.

**Save and reload**

```python
model.write().overwrite().save("C:/sparklabs/models/marks_pipeline")
loaded = PipelineModel.load("C:/sparklabs/models/marks_pipeline")
loaded.transform(new).select("prediction").show()      # same predictions as above
```

---

### 4. Project B: Pass/fail: compare all classifiers

**Goal:** predict `passed` and find which algorithm works best. We use the **same split** (`seed=42`) for all models.

```python
c = df.select(*feats, F.col("passed").alias("label"))
ctrain, ctest = c.randomSplit([0.8, 0.2], seed=42)

va = VectorAssembler(inputCols=feats, outputCol="features")
ctrain_f, ctest_f = va.transform(ctrain), va.transform(ctest)

models = {
    "Logistic Regression":      LogisticRegression(maxIter=100),
    "Decision Tree (depth 4)":  DecisionTreeClassifier(maxDepth=4, seed=42),
    "Linear SVM":               LinearSVC(maxIter=100, regParam=0.1),
    "Naive Bayes (Gaussian)":   NaiveBayes(modelType="gaussian"),
    "Random Forest (20 trees)": RandomForestClassifier(numTrees=20, maxDepth=4, seed=42),
}

acc = MulticlassClassificationEvaluator(metricName="accuracy")
f1  = MulticlassClassificationEvaluator(metricName="f1")
auc = BinaryClassificationEvaluator(metricName="areaUnderROC")

for name, estimator in models.items():
    m = estimator.fit(ctrain_f)
    p = m.transform(ctest_f)
    print(name, "| test acc", round(acc.evaluate(p), 4), "| F1", round(f1.evaluate(p), 4),
          "| AUC", round(auc.evaluate(p), 4), "| train acc", round(acc.evaluate(m.transform(ctrain_f)), 4))
```

**Results** (35 test students: 25 fail, 10 pass)

| Model | Train acc | **Test acc** | Test F1 | Test AUC |
| --- | --- | --- | --- | --- |
| **Linear SVM** (regParam 0.1) | 0.9515 | **0.9429** | 0.9443 | **0.988** |
| **Logistic Regression** | 0.9636 | 0.9143 | 0.9171 | **0.988** |
| Decision Tree (depth 4) | 0.9879 | 0.8857 | 0.8885 | 0.880 |
| Naive Bayes (Gaussian) | 0.8909 | 0.8286 | 0.8354 | 0.808 |
| Random Forest (20 trees, depth 4) | 0.9879 | 0.8000 | 0.8096 | 0.976 |
| *Baseline: always "fail"* | – | *0.7143* | – | *0.5* |

**Reading the table (the lessons)**

1. **Baseline first.** Predicting "fail" for everyone already gets **71.4%** (25 of 35). A model must beat this. Naive Bayes (82.9%) and Random Forest (80%) beat it, but not by much.
2. **The linear models win**, because our data was generated from a **linear** formula plus noise. Logistic regression and SVM match the structure; they also reach the best AUC (0.988).
3. **Overfitting is visible.** The tree and the forest have **train accuracy 0.988** but lower test accuracy (0.886 and 0.800). The gap (train − test) is the signature of overfitting (Phase 2).
4. **AUC and accuracy can disagree.** Random Forest has a high AUC (0.976) but low accuracy (0.80): it *ranks* students well but its default 0.5 threshold is poorly placed for this split. This is why we check several metrics.
5. **The test set is tiny (35 rows).** One student changes accuracy by 2.9 points. SVM's 0.9429 versus logistic regression's 0.9143 is a difference of **one student**, not a meaningful gap. With more data or cross-validation, the two would be considered equal.

> **Exam-ready conclusion:** *"Logistic regression and linear SVM performed best (AUC 0.988), consistent with the linear relationship in the data. The tree and the forest overfit. All models beat the 71% majority-class baseline."*

---

### 5. Project B2: tune a decision tree with cross-validation

The decision tree has two key hyperparameters: `maxDepth` and `minInstancesPerNode` (Phase 7). We search a grid with 3-fold CV on the **training set**.

```python
dt = DecisionTreeClassifier(seed=42)

grid = (ParamGridBuilder()
        .addGrid(dt.maxDepth, [2, 3, 4, 5, 6])
        .addGrid(dt.minInstancesPerNode, [1, 5])
        .build())                                   # 5 × 2 = 10 combinations

cv = CrossValidator(estimator=dt, estimatorParamMaps=grid,
                    evaluator=BinaryClassificationEvaluator(metricName="areaUnderROC"),
                    numFolds=3, seed=42, parallelism=2)

cv_model = cv.fit(ctrain_f)
```

**Cross-validated AUC for each combination**

| maxDepth | minInstancesPerNode = 1 | minInstancesPerNode = 5 |
| --- | --- | --- |
| 2 | 0.8150 | 0.8283 |
| 3 | 0.8399 | 0.8532 |
| 4 | 0.8870 | 0.8532 |
| 5 | 0.8925 | 0.8532 |
| **6** | **0.8927** | 0.8532 |

```python
best = cv_model.bestModel
print("best maxDepth:", best.getMaxDepth(), " minInstances:", best.getMinInstancesPerNode(), " nodes:", best.numNodes)
print("test acc:", round(acc.evaluate(cv_model.transform(ctest_f)), 4),
      " test AUC:", round(auc.evaluate(cv_model.transform(ctest_f)), 4))
```

```text
best maxDepth: 6  minInstances: 1  nodes: 27
test acc: 0.8286  test AUC: 0.888
```

**What this teaches**

1. CV picks **depth 6, minInstances 1** (AUC 0.8927), but depths 5 and 6 are almost tied (0.8925 vs 0.8927): the curve is **flat**, so the choice barely matters.
2. A larger `minInstancesPerNode = 5` caps AUC at 0.853: forcing leaves to be bigger prevents the tree from splitting enough.
3. The tuned tree's **test accuracy (0.829)** is *not* better than our hand-picked depth-4 tree (0.886). With only 35 test rows that difference is **noise** (2 students). Tuning improves the **expected** performance; it cannot guarantee a better score on one small test set.
4. Compare with the **linear models** (CV-tuned logistic regression reached CV AUC 0.995 in Phase 11): the *model family* matters more than fine-tuning a poorly matched one.

> **Lesson:** choose the algorithm family to match the data structure first; tune second.

---

### 6. Project C: Customer segmentation (clustering pipeline)

```python
cust = [(1, 15, 20), (2, 18, 25), (3, 20, 15), (4, 22, 22),
        (5, 18, 80), (6, 20, 85), (7, 24, 78), (8, 22, 82),
        (9, 80, 18), (10, 85, 22), (11, 88, 15), (12, 90, 20),
        (13, 82, 80), (14, 86, 85), (15, 90, 78), (16, 88, 82)]
cdf = spark.createDataFrame(cust, ["id", "income", "score"])

pipeline = Pipeline(stages=[
    VectorAssembler(inputCols=["income", "score"], outputCol="features"),
    StandardScaler(inputCol="features", outputCol="scaled", withMean=True, withStd=True),
    KMeans(featuresCol="scaled", k=4, seed=42)
])

model = pipeline.fit(cdf)
out = model.transform(cdf)

km = model.stages[-1]
print("sizes:", km.summary.clusterSizes, " cost:", round(km.summary.trainingCost, 3))
print("silhouette:", round(ClusteringEvaluator(featuresCol="scaled").evaluate(out), 4))
```

```text
sizes: [4, 4, 4, 4]  cost: 0.248
silhouette: 0.9882
```

(The cost is small, 0.248, because it is measured on the **scaled** features.)

**Profile each segment in the original units**

```python
(out.groupBy("prediction")
    .agg(F.round(F.avg("income"), 1).alias("avg_income"),
         F.round(F.avg("score"), 1).alias("avg_score"),
         F.count("*").alias("customers"))
    .orderBy("prediction").show())
```

```text
+----------+----------+---------+---------+
|prediction|avg_income|avg_score|customers|
+----------+----------+---------+---------+
|         0|      18.8|     20.5|        4|
|         1|      85.8|     18.8|        4|
|         2|      21.0|     81.3|        4|
|         3|      86.5|     81.3|        4|
+----------+----------+---------+---------+
```

| Segment | Profile | Name |
| --- | --- | --- |
| 0 | Low income, low spend | **Budget savers** |
| 1 | High income, low spend | **Cautious high earners** |
| 2 | Low income, high spend | **Enthusiastic spenders** |
| 3 | High income, high spend | **Premium customers** |

The Pipeline did the scaling for us (the very mistake that ruined Phase 8's unscaled demo cannot happen here).

---

### 7. Project D: Spam filter (text pipeline)

```python
docs = [("win free money now", 1), ("free money offer win", 1), ("win prize free", 1),
        ("meeting report today", 0), ("project report meeting", 0), ("lunch meeting today", 0)]
train = spark.createDataFrame(docs, ["text", "label"])

spam_pipeline = Pipeline(stages=[
    Tokenizer(inputCol="text", outputCol="words"),
    CountVectorizer(inputCol="words", outputCol="features", vocabSize=20),
    NaiveBayes(smoothing=1.0, modelType="multinomial")
])

spam_model = spam_pipeline.fit(train)

new = spark.createDataFrame([("free money meeting",), ("report today",), ("win free prize",)], ["text"])
spam_model.transform(new).select("text", "probability", "prediction").show(truncate=False)
```

```text
+-------------------+----------------------------------------+----------+
|text               |probability                             |prediction|
+-------------------+----------------------------------------+----------+
|free money meeting |[0.3073, 0.6927]                        |1.0       |
|report today       |[0.9159, 0.0841]                        |0.0       |
|win free prize     |[0.0399, 0.9601]                        |1.0       |
+-------------------+----------------------------------------+----------+
```

✔ The same results as Phase 10's step-by-step version (and its hand calculation of 0.6927), now in **one reusable object** that takes raw text.

---

### 8. Saving, loading and using a model

```python
# save a fitted PipelineModel (any of the projects above)
model.write().overwrite().save("C:/sparklabs/models/marks_pipeline")

# later, possibly in a different program or session
from pyspark.ml import PipelineModel
loaded = PipelineModel.load("C:/sparklabs/models/marks_pipeline")

new_students = spark.createDataFrame([(6.0, 80.0, 60.0)], ["hours", "attendance", "prev_marks"])
loaded.transform(new_students).select("prediction").show()
```

| Task | Code |
| --- | --- |
| Save | `model.write().overwrite().save(path)` |
| Load a pipeline | `PipelineModel.load(path)` |
| Load a single model | `LinearRegressionModel.load(path)` etc. |
| Predict | `loaded.transform(new_df)` |
| Tuned model | `cv_model.bestModel.write().overwrite().save(path)` |

**Rules for using a saved model**

1. New data must have the **same column names and types** that the pipeline's first stage expects.
2. Use the **same Spark and Python versions** (or compatible) for saving and loading.
3. The saved folder contains the **entire** pipeline (assembler, scaler statistics, model weights).

---

### 9. Using a saved model on streaming data (concept)

Units V and VI connect here. A saved `PipelineModel` can **score a stream** because its stages are ordinary transformers.

```text
 Kafka topic "students" ─▶ readStream ─▶ parse columns ─▶ model.transform(stream_df) ─▶ writeStream (console / Kafka)
```

```python
# CONCEPT SKETCH (not executed here: requires the Kafka setup from Unit V)
loaded = PipelineModel.load("C:/sparklabs/models/marks_pipeline")

stream_df = ...            # streaming DataFrame with columns hours, attendance, prev_marks (from Kafka)

scored = loaded.transform(stream_df).select("hours", "attendance", "prev_marks", "prediction")
query = scored.writeStream.format("console").outputMode("append").start()
```

The model is **trained offline** (batch) and **applied online** (streaming): the standard way to deploy ML in a real-time pipeline.

---

### 10. Algorithm selection guide

```text
 START: What do you want to do?
   │
   ├── Predict a NUMBER ─────────────▶ Linear Regression   (baseline)
   │                                   Decision Tree / Random Forest Regressor (non-linear)
   │
   ├── Predict a CATEGORY ──┬─ need PROBABILITIES / interpretable odds ─▶ Logistic Regression
   │                        ├─ need readable RULES, mixed feature types ─▶ Decision Tree
   │                        ├─ clear margin, many features, binary      ─▶ Linear SVM
   │                        ├─ TEXT / counts, tiny data, fast           ─▶ Naive Bayes
   │                        └─ non-linear, want accuracy                ─▶ Random Forest / GBT
   │
   └── GROUP unlabeled data ─────────▶ K-means   (choose K: elbow + silhouette; scale first)
```

| Algorithm | Task | Needs scaling? | Gives probabilities? | Interpretable? | Notes |
| --- | --- | --- | --- | --- | --- |
| **Linear Regression** | Regression | Helpful | – | ✅ coefficients | Linear only; sensitive to outliers |
| **Logistic Regression** | Classification | Helpful | ✅ | ✅ odds ratios | Linear boundary |
| **Decision Tree** | Both | ❌ | ✅ (leaf shares) | ✅✅ rules | Overfits if deep |
| **K-means** | Clustering | ✅ **Required** | – | ✅ centroids | Choose K; spherical clusters |
| **SVM (LinearSVC)** | Binary classification | ✅ | ❌ | Moderate | Linear only in Spark |
| **Naive Bayes** | Classification | ❌ (Gaussian: no; Multinomial: non-negative) | ✅ | ✅ | Fast; independence assumed |

---

### 11. The "question → algorithm" method

Underline keywords in the exam question.

| Keyword in the question | Choose |
| --- | --- |
| "predict the **price / marks / salary / amount**" | **Linear Regression** |
| "predict **pass or fail / yes or no / spam or not**" | **Logistic Regression** (or SVM / Naive Bayes / Tree) |
| "**probability** that…" | Logistic Regression / Naive Bayes |
| "**rules**", "explain the decision", "if-then" | **Decision Tree** |
| "**group / segment / cluster** customers" | **K-means** |
| "**maximum margin**", "support vectors", "hyperplane" | **SVM** |
| "**text**, emails, word counts", "Bayes' theorem" | **Naive Bayes** |
| "**multiclass**" (iris species, grades) | Logistic (multinomial), Decision Tree, Naive Bayes, OvR-SVM |
| "how good is the model", "**confusion matrix**" | Evaluate: accuracy / precision / recall / F1 |
| "**R²**, **RMSE**, **MAE**" | Regression metrics |
| "relationship between features", "**heat map**" | Correlation matrix + `sns.heatmap` |
| "choose the **best parameters**" | `ParamGridBuilder` + `CrossValidator` |
| "**overfitting**" | Compare train vs test; regularize / limit depth / more data |
| "**scale** the features" | `StandardScaler` / `MinMaxScaler` in a Pipeline |
| "**missing values**" | `Imputer` / `na.drop()` |
| "text categories to numbers" | `StringIndexer` + `OneHotEncoder` |

---

### 12. Ten exam-style scenarios

Assume `df` (200 students) and `feats` from Section 2 are available.

#### Scenario 1: Linear regression (predict a number)

**Question.** Train a linear regression on (hours → marks) = (1,40), (2,50), (3,55), (4,65), (5,75); print the coefficients and predict for 6 hours.

```python
data = VectorAssembler(inputCols=["hours"], outputCol="features").transform(
    spark.createDataFrame([(1.0, 40.0), (2.0, 50.0), (3.0, 55.0), (4.0, 65.0), (5.0, 75.0)], ["hours", "label"]))
m = LinearRegression().fit(data)
print(m.coefficients, m.intercept)
from pyspark.ml.linalg import Vectors
print(m.predict(Vectors.dense([6.0])))
```

```text
[8.5] 31.5
82.5
```

(31.5 + 8.5 × 6 = 82.5 ✔.)

#### Scenario 2: Evaluate a regression model

**Question.** Compute RMSE, MAE and R² on a test set.

→ Project A. Use `RegressionEvaluator` with `metricName` set to `rmse`, `mae`, `r2` (results 2.8413, 2.2595, 0.9291).

#### Scenario 3: Logistic regression (classification)

**Question.** Train logistic regression to predict `passed`; show accuracy and AUC.

```python
data = VectorAssembler(inputCols=feats, outputCol="features").transform(df).select("features", F.col("passed").alias("label"))
train, test = data.randomSplit([0.8, 0.2], seed=42)
p = LogisticRegression(maxIter=100).fit(train).transform(test)
print(MulticlassClassificationEvaluator(metricName="accuracy").evaluate(p),
      BinaryClassificationEvaluator(metricName="areaUnderROC").evaluate(p))
```

```text
0.9143 0.988
```

#### Scenario 4: Confusion matrix

**Question.** Show the confusion matrix and compute precision and recall.

```python
p.groupBy("label", "prediction").count().orderBy("label", "prediction").show()
```

```text
+-----+----------+-----+
|label|prediction|count|
+-----+----------+-----+
|    0|       0.0|   22|
|    0|       1.0|    3|
|    1|       1.0|   10|
+-----+----------+-----+
```

```text
 TN = 22, FP = 3, FN = 0, TP = 10
 precision = 10/13 = 0.769     recall = 10/10 = 1.0     accuracy = 32/35 = 0.9143
```

#### Scenario 5: Decision tree and its rules

**Question.** Train a depth-2 decision tree and print its rules.

```python
from pyspark.ml.classification import DecisionTreeClassifier
dtm = DecisionTreeClassifier(maxDepth=2, seed=42).fit(train)
print(dtm.toDebugString)
print(dtm.featureImportances)
```

```text
  If (feature 0 <= 4.95)
   If (feature 2 <= 78.85)
    Predict: 0.0
   Else (feature 2 > 78.85)
    Predict: 1.0
  Else (feature 0 > 4.95)
   Predict: 1.0
```

(feature 0 = hours, feature 2 = prev_marks.) Interpretation: more than ~5 study hours → pass.

#### Scenario 6: K-means

**Question.** Cluster the 7 points A to G into 2 groups; print the centres and the cost.

→ Phase 8, Section 11: centres (3.9, 5.1) and (1.25, 1.5), cost 8.525, silhouette 0.8315.

#### Scenario 7: SVM

**Question.** Train a linear SVM on the 8-point dataset; print weights and the margin.

→ Phase 9, Section 11: w = (0.62, 0.772), b = −3.1784, margin 2/‖w‖ = 2.0199.

#### Scenario 8: Naive Bayes

**Question.** Using the weather data, classify (Sunny, Cool, High, Strong).

→ Phase 10, Section 10: P(Yes) = 0.2046, P(No) = 0.7954, so predict **No**.

#### Scenario 9: Correlation heat map

**Question.** Draw the correlation heat map of the student features and label.

```python
import pandas as pd, matplotlib.pyplot as plt, seaborn as sns
from pyspark.ml.stat import Correlation

cols = ["hours", "attendance", "prev_marks", "final_marks"]
v = VectorAssembler(inputCols=cols, outputCol="v").transform(df)
corr = pd.DataFrame(Correlation.corr(v, "v").head()[0].toArray(), index=cols, columns=cols)
sns.heatmap(corr, annot=True, fmt=".2f", cmap="coolwarm", vmin=-1, vmax=1)
plt.title("Correlation heat map"); plt.show()
```

Key reading: hours ↔ final_marks = **0.83**; features are almost uncorrelated with each other.

#### Scenario 10: Tune a model with cross-validation

**Question.** Use `CrossValidator` to choose the best `regParam` for logistic regression.

```python
lr = LogisticRegression(maxIter=100)
grid = ParamGridBuilder().addGrid(lr.regParam, [0.0, 0.01, 0.1]).build()
cv = CrossValidator(estimator=lr, estimatorParamMaps=grid,
                    evaluator=BinaryClassificationEvaluator(), numFolds=3, seed=42)
cvm = cv.fit(train)
print(cvm.bestModel.getRegParam(), [round(x, 4) for x in cvm.avgMetrics])
```

(From Phase 11, with scaling in the pipeline, the best `regParam` was **0.01** with CV AUC 0.9951.)

---

### 13. Troubleshooting master table

| Symptom | Cause | Fix |
| --- | --- | --- |
| `Column features must be of type VectorUDT` | Missing assembler | `VectorAssembler` (or put it first in the Pipeline) |
| `label does not exist` | Wrong label column name | `.alias("label")` or `labelCol=` |
| `Encountered null while assembling a row` | Missing values | `Imputer` / `na.drop()` / `handleInvalid` |
| `Unseen label` | New category at prediction time | `StringIndexer(handleInvalid="keep")` |
| `Labels MUST be in [0, ...)` | Negative or text labels | `StringIndexer` to 0.0, 1.0… |
| `Vector values MUST NOT be Negative` | Negative features with Naive Bayes | MinMax scale or Gaussian NB |
| Suspiciously perfect scores | Leakage | Remove label-derived features |
| Train ≫ test accuracy | Overfitting | Lower depth, regularize, more data, CV |
| Low AUC | Passed `prediction` instead of scores | Use `rawPrediction` / `probability` |
| K-means groups look wrong | Unscaled features | Scaler inside the Pipeline |
| Different results each run | No seeds | `seed=42` everywhere |
| `Python in worker has different version` | Two Pythons | `PYSPARK_PYTHON` = `PYSPARK_DRIVER_PYTHON` |
| `ModuleNotFoundError: seaborn/matplotlib` | Not installed | `pip install seaborn matplotlib` |
| Heat map not shown | Missing `plt.show()` | Add it, or `savefig` |
| CV extremely slow | Large grid × folds | Smaller grid, `parallelism`, `TrainValidationSplit` |
| Pipeline fails on new data | Column names differ | Match the training schema |
| `toPandas()` out of memory | Collected too much | Aggregate in Spark first |

---

### 14. Master template

```python
# ---------------- SETUP ----------------
import sys; sys.path.append(r"C:\sparklabs")
from make_data import make_students
from pyspark.ml import Pipeline, PipelineModel
from pyspark.ml.feature import VectorAssembler, StandardScaler
from pyspark.ml.classification import LogisticRegression           # or any algorithm
from pyspark.ml.evaluation import BinaryClassificationEvaluator, MulticlassClassificationEvaluator
from pyspark.ml.tuning import ParamGridBuilder, CrossValidator
from pyspark.sql import functions as F

# ---------------- 1 LOAD + 2 UNDERSTAND ----------------
df = spark.createDataFrame(make_students(), ["id","hours","attendance","prev_marks","final_marks","passed"])
df.printSchema(); df.describe().show()

# ---------------- 3-4 PREPARE + SPLIT ----------------
feats = ["hours", "attendance", "prev_marks"]
data = df.select(*feats, F.col("passed").alias("label"))
train, test = data.randomSplit([0.8, 0.2], seed=42)

# ---------------- 5 PIPELINE ----------------
lr = LogisticRegression(featuresCol="scaled", maxIter=100)
pipe = Pipeline(stages=[VectorAssembler(inputCols=feats, outputCol="features"),
                        StandardScaler(inputCol="features", outputCol="scaled"), lr])

# ---------------- 6 FIT + 7 PREDICT + 8 EVALUATE ----------------
model = pipe.fit(train)
pred = model.transform(test)
print("accuracy:", MulticlassClassificationEvaluator(metricName="accuracy").evaluate(pred))
print("AUC     :", BinaryClassificationEvaluator(metricName="areaUnderROC").evaluate(pred))
pred.groupBy("label", "prediction").count().show()                 # confusion counts

# ---------------- 9 TUNE ----------------
grid = ParamGridBuilder().addGrid(lr.regParam, [0.0, 0.01, 0.1]).build()
cv = CrossValidator(estimator=pipe, estimatorParamMaps=grid,
                    evaluator=BinaryClassificationEvaluator(), numFolds=3, seed=42)
best = cv.fit(train)

# ---------------- 10 SAVE + USE ----------------
best.bestModel.write().overwrite().save("C:/sparklabs/models/passfail")
PipelineModel.load("C:/sparklabs/models/passfail").transform(test).select("prediction").show(5)
```

**Unit VI in five lines**

```text
1. ML learns a rule from data: supervised (regression, classification) or unsupervised (clustering).
2. Spark ML = DataFrames + Transformers/Estimators + Pipelines; features go in one Vector column.
3. Six algorithms: Linear/Logistic Regression, Decision Tree, K-means, SVM, Naive Bayes (each with maths).
4. Judge with the right metric on TEST data: confusion matrix/F1/AUC, RMSE/MAE/R², silhouette; tune with CV.
5. Compare to a baseline, watch for overfitting and leakage, and save the whole Pipeline.
```

**Phase 12 completed**

---

> **Next → Phase 13 (final): Unit VI Revision** (syllabus coverage map, formula and code cheat sheets for all six algorithms and the metrics, combined error table, **exactly 30 viva questions**, last-10-minute sheet). File: `13_Unit6_Final_Revision_30_Viva.md`
