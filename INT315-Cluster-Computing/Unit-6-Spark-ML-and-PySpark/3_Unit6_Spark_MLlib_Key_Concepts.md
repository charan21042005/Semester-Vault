# Unit VI — Phase 3: Spark MLlib and Key Concepts of Spark ML

> **Continuing from Phase 2.** We know the ML techniques. Now we meet the **tool**: Spark MLlib. We learn its building blocks (DataFrame, Vector, Transformer, Estimator, Pipeline, Evaluator, tuning) and the feature-preparation tools you will use in *every* algorithm. This covers the syllabus topics *Introduction to Spark MLlib* and *Key Concepts of Spark ML*.
>
> Every snippet and output was run in a real PySpark 3.5 session. This phase has no algorithm maths; it is the **grammar** of Spark ML. Master it and the six algorithm phases become easy.

---

## Phase 3 — Roadmap

| # | Topic |
| --- | --- |
| 1 | What is Spark MLlib? |
| 2 | The two APIs: `spark.ml` and `spark.mllib` |
| 3 | Why use MLlib: features and benefits |
| 4 | The package map |
| 5 | The key concepts at a glance |
| 6 | Transformer and Estimator |
| 7 | Vectors: dense and sparse |
| 8 | `VectorAssembler`: the mandatory first step |
| 9 | Standard column names and output columns |
| 10 | Feature transformers: Imputer |
| 11 | StringIndexer and OneHotEncoder |
| 12 | Scaling: StandardScaler and MinMaxScaler |
| 13 | Bucketizer |
| 14 | Pipeline |
| 15 | Saving and loading models |
| 16 | Parameters |
| 17 | Hyperparameter tuning (preview) |
| 18 | The universal ML code template |
| 19 | Common errors |
| 20 | Practical scenarios |
| 21 | Viva questions |
| 22 | Master revision |

---

### 1. What is Spark MLlib?

> **MLlib is Apache Spark's scalable machine learning library. It provides common learning algorithms, feature tools, pipelines and utilities that run in parallel on a cluster.**

```text
                    SPARK
 ┌───────────────────────────────────────────────┐
 │ Spark SQL │ Streaming │ GraphX │  MLlib  ◀ Unit VI
 ├───────────────────────────────────────────────┤
 │                 Spark Core (RDDs)             │
 └───────────────────────────────────────────────┘
```

MLlib sits on top of Spark Core, next to Spark SQL, Streaming and GraphX, so one program can read data, clean it, train a model and score a stream.

---

### 2. The two APIs: `spark.ml` and `spark.mllib`

| | **`pyspark.ml`** (use this) | `pyspark.mllib` (old) |
| --- | --- | --- |
| Data structure | **DataFrames** | RDDs |
| Status | **Primary API**, actively developed | Maintenance mode (bug fixes only) |
| Pipelines | ✅ Yes | ❌ No |
| Import | `from pyspark.ml... import ...` | `from pyspark.mllib... import ...` |

> **Rule for Unit VI:** use **`pyspark.ml`** (DataFrame-based) for everything. We touch `pyspark.mllib` only for one small helper in Phase 11 (`MulticlassMetrics`). When people say "MLlib" today they usually mean the DataFrame-based `spark.ml` API.

---

### 3. Why use MLlib: features and benefits

| Feature | Meaning |
| --- | --- |
| **Scalable** | Trains on data spread over many machines |
| **Fast, in-memory** | Iterative algorithms keep data in memory (much faster than MapReduce) |
| **Easy** | High-level APIs in Python, Scala, Java and R |
| **Unified platform** | Same DataFrames used for SQL, streaming and ML |
| **Pipelines** | Chain preparation and modelling into one reusable object |
| **Rich library** | Classification, regression, clustering, recommendation, feature tools, tuning |
| **Persistence** | Save and load models and pipelines |
| **Integrates** | Works with Kafka streams, Hadoop files, databases |

---

### 4. The package map

You will import from the same few modules again and again.

| Module | Contains | Examples |
| --- | --- | --- |
| `pyspark.ml.feature` | Feature tools | `VectorAssembler`, `StringIndexer`, `OneHotEncoder`, `StandardScaler`, `Imputer` |
| `pyspark.ml.regression` | Regression algorithms | `LinearRegression`, `DecisionTreeRegressor` |
| `pyspark.ml.classification` | Classification algorithms | `LogisticRegression`, `DecisionTreeClassifier`, `LinearSVC`, `NaiveBayes` |
| `pyspark.ml.clustering` | Clustering | `KMeans` |
| `pyspark.ml.evaluation` | Metrics | `RegressionEvaluator`, `BinaryClassificationEvaluator`, `MulticlassClassificationEvaluator`, `ClusteringEvaluator` |
| `pyspark.ml.tuning` | Tuning | `ParamGridBuilder`, `CrossValidator`, `TrainValidationSplit` |
| `pyspark.ml.stat` | Statistics | `Correlation` |
| `pyspark.ml.linalg` | Vectors | `Vectors` |
| `pyspark.ml` | Pipeline | `Pipeline`, `PipelineModel` |

---

### 5. The key concepts at a glance

```text
 DataFrame ──▶ Transformer ──▶ DataFrame        (transform: adds columns)
 DataFrame ──▶ Estimator.fit() ──▶ Model        (learns; the Model is a Transformer)
 Pipeline = chain of Transformers and Estimators
 Evaluator = computes a metric from predictions
 ParamGrid + CrossValidator = tries settings and picks the best
```

| Concept | One-line meaning |
| --- | --- |
| **DataFrame** | The dataset (rows and columns) |
| **Transformer** | Turns one DataFrame into another by adding/changing columns. Has `transform()` |
| **Estimator** | An algorithm that **learns** from a DataFrame. Has `fit()` and returns a **Model** |
| **Model** | The result of `fit()`. It is a Transformer (has `transform()`) |
| **Pipeline** | An ordered chain of stages; itself an Estimator |
| **Parameter** | A setting of a Transformer/Estimator (such as `maxIter`) |
| **Evaluator** | Measures prediction quality |

---

### 6. Transformer and Estimator

The two most important ideas in Spark ML.

#### 6.1 Transformer

A Transformer has a `transform(df)` method that **adds columns** to a DataFrame. It does **not learn** anything.

| Transformer | What it does |
| --- | --- |
| `VectorAssembler` | Combines columns into one `features` vector |
| `Bucketizer` | Puts numbers into ranges |
| `Tokenizer` | Splits text into words |
| **Any trained model** | Adds a `prediction` column |

```text
Transformer:    DataFrame ───── transform() ─────▶ DataFrame + new column(s)
```

#### 6.2 Estimator

An Estimator has a `fit(df)` method that **learns from data** and returns a **Model**.

| Estimator | Returns this Model | What was learned |
| --- | --- | --- |
| `StandardScaler` | `StandardScalerModel` | Mean and standard deviation |
| `StringIndexer` | `StringIndexerModel` | The label → index mapping |
| `LinearRegression` | `LinearRegressionModel` | Coefficients, intercept |
| `LogisticRegression` | `LogisticRegressionModel` | Coefficients, intercept |
| `DecisionTreeClassifier` | `DecisionTreeClassificationModel` | The tree |
| `KMeans` | `KMeansModel` | Cluster centres |

```text
Estimator:      DataFrame ──── fit() ────▶ Model ──── transform() ────▶ DataFrame + prediction
                (training)                  (learned)                    (any data)
```

**The pattern that never changes**

```python
model = estimator.fit(train)          # LEARN from the training data
result = model.transform(test)        # APPLY to the test data
```

> **Naming rule:** the Estimator is `LinearRegression`; the Model it produces is `LinearRegressionModel`. *Estimator = recipe, Model = the cooked dish.*

---

### 7. Vectors: dense and sparse

Spark ML algorithms read all features from **one column of type Vector**.

```python
from pyspark.ml.linalg import Vectors

d = Vectors.dense([1.0, 0.0, 3.0])                # dense: stores every value
s = Vectors.sparse(3, [0, 2], [1.0, 3.0])          # sparse: (size, positions, values)

print(d)                     # [1.0,0.0,3.0]
print(s)                     # (3,[0,2],[1.0,3.0])
print(s.toArray())           # [1. 0. 3.]
print(d.size)                # 3
print(d.dot(Vectors.dense([1, 1, 1])))   # 4.0
```

| Vector type | Stores | Best when |
| --- | --- | --- |
| **Dense** `[1.0, 0.0, 3.0]` | **All** values | Most values are non-zero |
| **Sparse** `(3,[0,2],[1.0,3.0])` | Size + positions of the **non-zero** values only | Mostly zeros (text, one-hot) |

**Reading a sparse vector:** `(3, [0, 2], [1.0, 3.0])` means "length 3; position 0 holds 1.0, position 2 holds 3.0; everything else is 0".

---

### 8. `VectorAssembler`: the mandatory first step

Our data has features in **separate columns**. Spark ML algorithms need them **combined into one vector column**. `VectorAssembler` does that.

```text
 hours  attendance  prev_marks                   features
  6.8      51.3        50.1       ──▶ VectorAssembler ──▶ [6.8, 51.3, 50.1]
```

```python
from pyspark.ml.feature import VectorAssembler

va = VectorAssembler(inputCols=["hours", "attendance", "prev_marks"], outputCol="features")
data = va.transform(df)
data.select("features", "final_marks").show(3, truncate=False)
```

```text
+----------------+-----------+
|features        |final_marks|
+----------------+-----------+
|[6.8,51.3,50.1] |63.4       |
|[7.1,94.6,39.8] |75.8       |
|[4.8,51.5,47.0] |52.8       |
+----------------+-----------+
```

| Piece | Meaning |
| --- | --- |
| `inputCols` | The feature columns to combine (numeric, boolean or vector) |
| `outputCol` | Name of the new vector column (conventionally `features`) |

> **Do not include the label in `inputCols`.** The label stays as a separate column.

**Handling missing values (`handleInvalid`)**

| Value | Behaviour when a null/NaN is found |
| --- | --- |
| `"error"` (default) | Fails: *Encountered null while assembling a row with handleInvalid = "error"* |
| `"skip"` | Drops that row |
| `"keep"` | Keeps the row and stores `NaN` in the vector |

```text
Input:   (1.0, 2.0) (2.0, NULL) (3.0, 3.0)
skip  →  rows 1 and 3 only
keep  →  [1.0,2.0]  [2.0,NaN]  [3.0,3.0]
```

Better still, **fill** missing values first (next section).

---

### 9. Standard column names and output columns

Algorithms look for specific column names by default.

| Parameter | Default name | Holds |
| --- | --- | --- |
| `featuresCol` | `features` | The input vector |
| `labelCol` | `label` | The target |
| `predictionCol` | `prediction` | The predicted value |
| `probabilityCol` | `probability` | Class probabilities (classifiers) |
| `rawPredictionCol` | `rawPrediction` | Raw scores (classifiers) |

If your columns have other names, tell the algorithm:

```python
LinearRegression(featuresCol="features", labelCol="final_marks")
```

Or rename once and rely on the defaults (we do this: `F.col("final_marks").alias("label")`).

**What `model.transform()` adds**

| Algorithm type | New columns |
| --- | --- |
| Regression | `prediction` |
| Classification | `rawPrediction`, `probability`, `prediction` |
| Clustering | `prediction` (cluster number) |

---

### 10. Feature transformers: Imputer

Real data has **missing values**. `Imputer` fills them with the column's **mean**, **median** or **mode**.

Sample data (one attendance value missing):

```python
df_demo = spark.createDataFrame(
    [(1, "CSE", 2.0, 60.0), (2, "ECE", 5.0, None), (3, "CSE", 7.0, 90.0), (4, "IT", 3.0, 70.0)],
    ["id", "dept", "hours", "attendance"])

from pyspark.ml.feature import Imputer
imp = Imputer(inputCols=["attendance"], outputCols=["attendance_imp"], strategy="mean")
df2 = imp.fit(df_demo).transform(df_demo)
df2.show()
```

```text
+---+----+-----+----------+-----------------+
| id|dept|hours|attendance|   attendance_imp|
+---+----+-----+----------+-----------------+
|  1| CSE|  2.0|      60.0|             60.0|
|  2| ECE|  5.0|      NULL|73.33333333333333|
|  3| CSE|  7.0|      90.0|             90.0|
|  4|  IT|  3.0|      70.0|             70.0|
+---+----+-----+----------+-----------------+
```

Check: mean of 60, 90, 70 = 220 / 3 = **73.33** ✔. (`Imputer` is an Estimator: it *learns* the mean during `fit`.)

---

### 11. StringIndexer and OneHotEncoder

Algorithms need **numbers**, but categories like `CSE`, `ECE`, `IT` are text. Two steps convert them.

#### 11.1 `StringIndexer`: text → index number

```python
from pyspark.ml.feature import StringIndexer
sim = StringIndexer(inputCol="dept", outputCol="dept_idx").fit(df2)
df3 = sim.transform(df2)
df3.select("dept", "dept_idx").show()
print(sim.labels)
```

```text
+----+--------+
|dept|dept_idx|
+----+--------+
| CSE|     0.0|
| ECE|     1.0|
| CSE|     0.0|
|  IT|     2.0|
+----+--------+

['CSE', 'ECE', 'IT']
```

The most **frequent** value gets index 0 (`CSE` appears twice). The indices are just *labels*, not quantities. A model must not think `IT = 2` is "twice" `ECE = 1`. For that we need one-hot encoding.

> `StringIndexer` is also used to convert a **text label** (such as `Yes`/`No`) into the numeric `label` column (0.0/1.0) for classification.

#### 11.2 `OneHotEncoder`: index → a vector of 0/1

```python
from pyspark.ml.feature import OneHotEncoder
ohe = OneHotEncoder(inputCols=["dept_idx"], outputCols=["dept_vec"]).fit(df3)
ohe.transform(df3).select("dept", "dept_idx", "dept_vec").show(truncate=False)
```

```text
+----+--------+-------------+
|dept|dept_idx|dept_vec     |
+----+--------+-------------+
|CSE |0.0     |(2,[0],[1.0])|
|ECE |1.0     |(2,[1],[1.0])|
|CSE |0.0     |(2,[0],[1.0])|
|IT  |2.0     |(2,[],[])    |
+----+--------+-------------+
```

```text
CSE → [1, 0]       ECE → [0, 1]       IT → [0, 0]   (the last category is dropped)
```

By default `dropLast=True`: 3 categories become **2** columns, because the last category is implied when all others are 0. This avoids a redundant column.

**Different `handleInvalid` for unseen categories**

A category seen at prediction time but not in training (for example a new department `MECH`) causes `Unseen label: MECH. To handle unseen labels, set Param handleInvalid to keep.` Use `StringIndexer(..., handleInvalid="keep")` to put unseen values in an extra bucket.

#### 11.3 Putting it together with `VectorAssembler`

```python
va = VectorAssembler(inputCols=["hours", "attendance_imp", "dept_vec"], outputCol="features")
df5 = va.transform(ohe.transform(df3))
df5.select("id", "features").show(truncate=False)
```

```text
+---+-------------------------------+
|id |features                       |
+---+-------------------------------+
|1  |[2.0,60.0,1.0,0.0]             |
|2  |[5.0,73.33333333333333,0.0,1.0]|
|3  |[7.0,90.0,1.0,0.0]             |
|4  |[3.0,70.0,0.0,0.0]             |
+---+-------------------------------+
```

Now every row is a numeric vector: `[hours, attendance, is_CSE, is_ECE]`.

---

### 12. Scaling: StandardScaler and MinMaxScaler

Features can have very different ranges (hours 1 to 10; attendance 50 to 100; income 20,000 to 90,000). Algorithms based on **distances or gradients** (K-means, SVM, regularized regression) are dominated by the large-valued feature. **Scaling** puts them on comparable ranges.

#### 12.1 StandardScaler (z-score)

**Formula**

```text
          x − μ
 z  =  ─────────
            σ

 μ = mean of the column       σ = (sample) standard deviation
```

After scaling, every feature has **mean 0** and **standard deviation 1**.

```python
from pyspark.ml.feature import StandardScaler
sc = StandardScaler(inputCol="features", outputCol="scaled", withMean=True, withStd=True)
scm = sc.fit(df5)
scm.transform(df5).select("scaled").show(truncate=False)
print(scm.mean, scm.std)
```

```text
+------------------------------------------------------------------+
|scaled                                                            |
+------------------------------------------------------------------+
|[-1.014722137803819,-1.0690449676496967,0.8660254037844387,-0.5]  |
|[0.338240712601273,0.0,-0.8660254037844387,1.5]                   |
|[1.2402159462046676,1.3363062095621219,0.8660254037844387,-0.5]   |
|[-0.5637345210021216,-0.2672612419124239,-0.8660254037844387,-0.5]|
+------------------------------------------------------------------+

[4.25,73.33333333333333,0.5,0.25] [2.217355782608345,12.472191289246476,0.5773502691896257,0.5]
```

**Hand check for the first value (hours = 2.0)**

```text
mean μ = (2 + 5 + 7 + 3) / 4 = 4.25
sample σ = sqrt( ((2−4.25)² + (5−4.25)² + (7−4.25)² + (3−4.25)²) / (4−1) )
         = sqrt( (5.0625 + 0.5625 + 7.5625 + 1.5625) / 3 ) = sqrt(4.9167) = 2.2174
z = (2 − 4.25) / 2.2174 = −1.0147   ✔ (matches −1.01472...)
```

| Parameter | Meaning |
| --- | --- |
| `withMean=True` | Subtract the mean (centre at 0). Default is **False** |
| `withStd=True` | Divide by the standard deviation. Default **True** |

> Note: `withMean=True` makes sparse vectors dense. For sparse data use `withMean=False`.

#### 12.2 MinMaxScaler

**Formula** (scales into [0, 1] by default)

```text
          x − min
 x' =  ─────────────
         max − min
```

```python
from pyspark.ml.feature import MinMaxScaler
mm = MinMaxScaler(inputCol="features", outputCol="mm").fit(df5)
mm.transform(df5).select("mm").show(truncate=False)
```

```text
+------------------------------------------------+
|mm                                              |
+------------------------------------------------+
|(4,[2],[1.0])                                   |
|[0.6000000000000001,0.44444444444444425,0.0,1.0]|
|[1.0,1.0,1.0,0.0]                               |
|[0.2,0.3333333333333333,0.0,0.0]                |
+------------------------------------------------+
```

Hand check, hours: min 2, max 7. Value 5 → (5 − 2) / (7 − 2) = **0.6** ✔.

**StandardScaler vs MinMaxScaler**

| | StandardScaler | MinMaxScaler |
| --- | --- | --- |
| Result | Mean 0, std 1 (values can be negative) | Within [0, 1] |
| Sensitive to outliers | Less | **More** (one huge value squashes the rest) |
| Typical use | SVM, regression, K-means | Neural networks, bounded inputs |

**The leakage rule for scalers:** `fit` the scaler **on the training data only**, then `transform` both train and test with that fitted scaler.

```python
scaler_model = StandardScaler(inputCol="features", outputCol="scaled").fit(train)   # learn from train
train_s = scaler_model.transform(train)
test_s  = scaler_model.transform(test)        # reuse the TRAIN mean and std
```

A Pipeline (Section 14) does this correctly and automatically.

---

### 13. Bucketizer

Turns a continuous number into **range buckets**.

```python
from pyspark.ml.feature import Bucketizer
b = Bucketizer(splits=[0, 4, 7, float("inf")], inputCol="hours", outputCol="hours_bucket")
b.transform(df_demo).select("hours", "hours_bucket").show()
```

```text
+-----+------------+
|hours|hours_bucket|
+-----+------------+
|  2.0|         0.0|
|  5.0|         1.0|
|  7.0|         2.0|
|  3.0|         0.0|
+-----+------------+
```

Buckets: `[0,4)` → 0, `[4,7)` → 1, `[7,∞)` → 2.

Other useful feature tools (know the names): `Tokenizer`, `HashingTF` / `CountVectorizer` / `IDF` (text), `PCA` (dimension reduction), `Normalizer`, `Binarizer`, `PolynomialExpansion`, `ChiSqSelector`.

---

### 14. Pipeline

A **Pipeline** chains several stages (Transformers and Estimators) into **one object**.

```text
Raw DataFrame ─▶ Imputer ─▶ StringIndexer ─▶ OneHotEncoder ─▶ VectorAssembler ─▶ [Algorithm]
                  └──────────── one Pipeline ───────────────────────────────────────┘
```

**Why use a Pipeline?**

| Benefit | Explanation |
| --- | --- |
| **One call** | `pipeline.fit(train)` runs every stage in order |
| **No leakage** | Statistics (means, scalers, indexers) are learned from **training data only** |
| **Reusable** | The fitted `PipelineModel` can be saved and applied to new data |
| **Tunable** | `CrossValidator` can tune the whole pipeline |

```python
from pyspark.ml import Pipeline

imp = Imputer(inputCols=["attendance"], outputCols=["attendance_imp"], strategy="mean")
si  = StringIndexer(inputCol="dept", outputCol="dept_idx")
ohe = OneHotEncoder(inputCols=["dept_idx"], outputCols=["dept_vec"])
va  = VectorAssembler(inputCols=["hours", "attendance_imp", "dept_vec"], outputCol="features")

pipeline = Pipeline(stages=[imp, si, ohe, va])
pmodel = pipeline.fit(df_demo)                 # learns the mean, the indexer mapping, ...
pmodel.transform(df_demo).select("id", "features").show(truncate=False)
print([type(s).__name__ for s in pmodel.stages])
```

```text
+---+-------------------------------+
|id |features                       |
+---+-------------------------------+
|1  |[2.0,60.0,1.0,0.0]             |
|2  |[5.0,73.33333333333333,0.0,1.0]|
|3  |[7.0,90.0,1.0,0.0]             |
|4  |[3.0,70.0,0.0,0.0]             |
+---+-------------------------------+

['ImputerModel', 'StringIndexerModel', 'OneHotEncoderModel', 'VectorAssembler']
```

**Reading the types:** `fit()` turned each **Estimator** (`Imputer`, `StringIndexer`, `OneHotEncoder`) into a **Model**; the **Transformer** `VectorAssembler` stayed as it was. The `Pipeline` (an Estimator) became a `PipelineModel` (a Transformer).

**A Pipeline can end with an algorithm**

```python
pipeline = Pipeline(stages=[assembler, scaler, LinearRegression(labelCol="label")])
```

Then `pipeline.fit(train)` does the *whole* job: assemble, scale (learned on train) and train the regression. `model.transform(test)` applies everything to test data.

---

### 15. Saving and loading models

A trained model or pipeline can be written to disk and loaded later, even in another program.

```python
# save
pmodel.write().overwrite().save("C:/sparklabs/models/prep_pipeline")

# load
from pyspark.ml import PipelineModel
loaded = PipelineModel.load("C:/sparklabs/models/prep_pipeline")
print(loaded.transform(df_demo).count())     # 4
```

For a single model, use its own class:

```python
model.write().overwrite().save("C:/sparklabs/models/lr")

from pyspark.ml.regression import LinearRegressionModel
same_model = LinearRegressionModel.load("C:/sparklabs/models/lr")
print(same_model.coefficients, same_model.intercept)
```

| Method | Purpose |
| --- | --- |
| `model.save(path)` / `model.write().overwrite().save(path)` | Save (the second replaces an existing folder) |
| `ModelClass.load(path)` | Load a model of that class |

> The saved path is a **folder** holding metadata and data files (Parquet), just like Spark's other outputs.

---

### 16. Parameters

Every Transformer/Estimator has **parameters** (settings).

**Three ways to set them**

```python
from pyspark.ml.regression import LinearRegression

lr = LinearRegression(maxIter=50, regParam=0.1)     # 1) in the constructor
lr.setElasticNetParam(0.5)                          # 2) with a setter
lr2 = lr.copy({lr.regParam: 0.01})                  # 3) copy with overrides
```

**Reading parameters**

```python
lr = LinearRegression()
print(lr.getRegParam(), lr.getMaxIter(), lr.getSolver(), lr.getFeaturesCol(), lr.getPredictionCol())
```

```text
0.0 100 auto features prediction
```

**Listing all parameters with explanations**

```python
print(lr.explainParams())      # prints every parameter, its meaning and default
```

`explainParams()` is the quickest way to discover what an algorithm accepts.

---

### 17. Hyperparameter tuning (preview)

**Hyperparameters** (like `regParam`, `maxDepth`, `k`) are chosen by us. Instead of guessing, Spark can **try many combinations** and pick the best using cross-validation (Phase 2, Section 15).

```python
from pyspark.ml.tuning import ParamGridBuilder, CrossValidator
from pyspark.ml.classification import LogisticRegression
from pyspark.ml.evaluation import BinaryClassificationEvaluator

lr = LogisticRegression(maxIter=100)

grid = (ParamGridBuilder()
        .addGrid(lr.regParam, [0.0, 0.01, 0.1])
        .addGrid(lr.elasticNetParam, [0.0, 0.5])
        .build())

cv = CrossValidator(estimator=lr,
                    estimatorParamMaps=grid,
                    evaluator=BinaryClassificationEvaluator(metricName="areaUnderROC"),
                    numFolds=3, seed=42)

cv_model = cv.fit(train)              # trains 6 combinations × 3 folds = 18 models
best = cv_model.bestModel
```

| Piece | Role |
| --- | --- |
| `ParamGridBuilder` | Lists the values to try (here 3 × 2 = **6** combinations) |
| `CrossValidator` | Tests each combination with k-fold cross-validation |
| `Evaluator` | The metric used to compare combinations |
| `bestModel` | The model refit with the winning setting |

| Related tool | Difference |
| --- | --- |
| `CrossValidator` | k folds: reliable, k times slower |
| `TrainValidationSplit` | One train/validation split: faster, less reliable |

The complete tuning workflow is in Phase 11.

---

### 18. The universal ML code template

Every algorithm phase in Unit VI follows this skeleton. Learn it once.

```python
from pyspark.ml.feature import VectorAssembler
from pyspark.ml.regression import LinearRegression           # or any algorithm
from pyspark.ml.evaluation import RegressionEvaluator
from pyspark.sql import functions as F

# 1. Load data
df = spark.createDataFrame(make_students(),
        ["id", "hours", "attendance", "prev_marks", "final_marks", "passed"])

# 2. Assemble features and name the label
va = VectorAssembler(inputCols=["hours", "attendance", "prev_marks"], outputCol="features")
data = va.transform(df).select("features", F.col("final_marks").alias("label"))

# 3. Split
train, test = data.randomSplit([0.8, 0.2], seed=42)

# 4. Train
model = LinearRegression(featuresCol="features", labelCol="label").fit(train)

# 5. Predict
pred = model.transform(test)

# 6. Evaluate
rmse = RegressionEvaluator(labelCol="label", metricName="rmse").evaluate(pred)
print("RMSE:", rmse)
```

```text
Load → Assemble → Split → Fit → Transform → Evaluate
```

> **For exams:** when the question gives a dataset and an algorithm, write these six steps in order. The algorithm is the only part that changes.

---

### 19. Common errors

| Error / symptom | Cause | Fix |
| --- | --- | --- |
| `requirement failed: Column a must be of type ... VectorUDT ... but was actually ... DoubleType` | Passed a plain numeric column as `featuresCol` | Combine columns with `VectorAssembler` |
| `label does not exist. Available: a, b, y, features` | No column named `label` | Rename the target with `.alias("label")` or set `labelCol` |
| `Encountered null while assembling a row with handleInvalid = "error"` | Missing values in a feature column | `Imputer`, `na.drop()`, or `handleInvalid="skip"/"keep"` |
| `Unseen label: c. To handle unseen labels, set Param handleInvalid to keep.` | Category not seen during `StringIndexer.fit` | `handleInvalid="keep"` |
| `Labels MUST be in [0, 2147483647), but got -1.0` | Negative class label | Labels must be 0, 1, 2, … (use `StringIndexer`) |
| `Vector values MUST NOT be Negative, NaN or Infinity` | Negative feature for Naive Bayes | Use non-negative features (scale to [0,1]) or Gaussian NB |
| Results change on every run | No seed in `randomSplit` or the algorithm | Always set `seed=42` |
| Test score unrealistically high | Leakage (label-derived feature, or scaler fitted on all data) | Remove the feature; use a Pipeline |
| Label in `inputCols` | Included the target in the features | Remove it from `inputCols` |
| `fit` on test data | Wrong data in `fit` | `fit(train)`, then `transform(test)` |
| Forgot to rename the label for the evaluator | Evaluator looks for `label` | Set `labelCol=` on the evaluator |

---

### 20. Phase 3 — practical scenarios

1. Create **dense and sparse vectors** and print `toArray()`.
2. Use **`VectorAssembler`** to build `features` from the three student columns.
3. Fill a missing value using **`Imputer`** (mean) and verify by hand.
4. **Index** a categorical column with `StringIndexer`; print `labels`.
5. **One-hot encode** the indexed column; explain why one category disappears.
6. **Standardize** features and verify one value by hand with `z = (x − μ)/σ`.
7. **Min-max scale** features and verify one value by hand.
8. Build a **Pipeline** with imputing, indexing, encoding and assembling; print the stage types.
9. **Save and reload** the fitted pipeline.
10. Print all parameters of `LogisticRegression` using **`explainParams()`**.
11. Define a **`ParamGridBuilder`** with 2 × 3 values; count the combinations.
12. Write the **six-step ML template** from memory for a regression task.

---

### 21. Phase 3 — viva questions and answers

#### Q1. What is Spark MLlib?

Spark's scalable machine learning library, providing algorithms, feature tools, pipelines and utilities that run in parallel.

#### Q2. What is the difference between `spark.ml` and `spark.mllib`?

`spark.ml` is the DataFrame-based primary API with pipelines; `spark.mllib` is the older RDD-based API in maintenance mode.

#### Q3. What are the advantages of MLlib?

Scalability, in-memory speed, easy APIs, a unified platform with SQL and streaming, pipelines, a rich algorithm library and model persistence.

#### Q4. What is a Transformer?

A component with a `transform()` method that converts one DataFrame into another by adding or changing columns, without learning (for example `VectorAssembler`, or any fitted model).

#### Q5. What is an Estimator?

An algorithm with a `fit()` method that learns from a DataFrame and returns a Model (for example `LinearRegression`, `StandardScaler`).

#### Q6. What is the relationship between an Estimator and a Model?

`fit()` on an Estimator produces a Model, and the Model is a Transformer with `transform()`.

#### Q7. What is a Pipeline?

An ordered chain of Transformers and Estimators treated as a single Estimator; fitting it produces a `PipelineModel`.

#### Q8. Why use a Pipeline?

It runs all steps in one call, learns preprocessing statistics only from training data (avoiding leakage), is reusable and can be tuned and saved.

#### Q9. Why is `VectorAssembler` required?

Spark ML algorithms read features from one Vector column, and `VectorAssembler` combines the individual feature columns into it.

#### Q10. What is the difference between dense and sparse vectors?

Dense vectors store every value; sparse vectors store only the size and the positions and values of non-zero elements.

#### Q11. What do `StringIndexer` and `OneHotEncoder` do?

`StringIndexer` maps text categories to numeric indices; `OneHotEncoder` turns those indices into 0/1 vectors so a model does not assume an order.

#### Q12. What is the formula for standardization?

`z = (x − μ) / σ`, giving mean 0 and standard deviation 1.

#### Q13. What is the formula for min-max scaling?

`x' = (x − min) / (max − min)`, scaling to [0, 1].

#### Q14. Why must a scaler be fitted on training data only?

Fitting on all data lets test-set information influence training (data leakage).

#### Q15. What does `Imputer` do?

Replaces missing values with the mean, median or mode learned from the data.

#### Q16. What are the default column names for features, label and prediction?

`features`, `label` and `prediction`.

#### Q17. What columns does a classifier add on `transform`?

`rawPrediction`, `probability` and `prediction`.

#### Q18. How can you see all the parameters of an algorithm?

Call `explainParams()` on it.

#### Q19. What are `ParamGridBuilder` and `CrossValidator`?

`ParamGridBuilder` lists hyperparameter values to try; `CrossValidator` evaluates each combination with k-fold cross-validation and selects the best model.

#### Q20. How do you save and load a model?

`model.write().overwrite().save(path)` and `ModelClass.load(path)` (or `PipelineModel.load`).

---

### 22. Phase 3 — final master recap

```text
MLlib = Spark's ML library.  USE pyspark.ml (DataFrame-based).

Transformer : transform()   adds columns          (VectorAssembler, Bucketizer, any Model)
Estimator   : fit() → Model learns from data      (LinearRegression, StandardScaler, StringIndexer)
Pipeline    : chain of stages → fit() → PipelineModel
Evaluator   : metric from predictions
ParamGrid + CrossValidator : tuning

FEATURES    Imputer → StringIndexer → OneHotEncoder → VectorAssembler → StandardScaler
VECTORS     dense [1.0,0.0,3.0]   sparse (3,[0,2],[1.0,3.0])
TEMPLATE    Load → Assemble → Split → Fit → Transform → Evaluate
```

**Seven facts to memorise**

1. Always `fit` on train, `transform` on test.
2. Features must be **one Vector column** (`VectorAssembler`).
3. Estimator → `fit()` → Model; Model is a Transformer.
4. Text categories need `StringIndexer` (+ `OneHotEncoder`).
5. Scale for K-means, SVM and regularized models; not needed for trees or Naive Bayes.
6. Fit scalers on training data only (use a Pipeline).
7. Always set `seed=42` for repeatable results.

### Phase 3 completed

---

> **Next → Phase 4: Data Understanding and the Correlation Heat Map** (descriptive statistics, mean, variance, standard deviation, covariance, Pearson correlation with its formula, Spark's `Correlation`, building the correlation heat map with seaborn, multicollinearity). File: `4_Unit6_Data_Understanding_and_Correlation_Heatmap.md`
