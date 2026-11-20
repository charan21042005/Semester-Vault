# Unit VI — Phase 7: Decision Tree

> **Continuing from Phase 6.** Linear and logistic regression draw straight lines. A **decision tree** asks a series of simple questions and follows the answers, like a flowchart. It handles numbers and categories, captures non-linear patterns, and is the most **explainable** model in this unit. We learn the intuition, **entropy, Gini impurity and information gain** (worked by hand on the classic weather dataset), how Spark builds trees, regression trees, overfitting control and feature importance.
>
> All outputs are real. The hand-computed gains (0.2467 for Outlook and so on) match our own Python calculation, and Spark's trees confirm the same root split.

---

## Phase 7 — Roadmap

| # | Topic |
| --- | --- |
| 1 | The idea: a flowchart that learns itself |
| 2 | Anatomy of a tree |
| 3 | Measuring purity: entropy |
| 4 | Gini impurity |
| 5 | Information gain |
| 6 | A complete worked example: the weather dataset |
| 7 | The tree-building algorithm |
| 8 | Continuous features: threshold splits |
| 9 | Regression trees |
| 10 | Overfitting and how to control it |
| 11 | Feature importance |
| 12 | PySpark code: weather data (categorical) |
| 13 | Reading `toDebugString` |
| 14 | PySpark code: students (numeric) |
| 15 | Gini vs entropy in Spark |
| 16 | PySpark code: decision tree regressor |
| 17 | A peek at Random Forest |
| 18 | Important parameters |
| 19 | Strengths and weaknesses |
| 20 | Common errors |
| 21 | Practical scenarios |
| 22 | Viva questions |
| 23 | Master revision |

---

### 1. The idea: a flowchart that learns itself

Imagine deciding whether to play tennis. You ask questions:

```text
                     Outlook?
              ┌─────────┼──────────┐
            Sunny    Overcast     Rain
              │          │          │
          Humidity?     YES       Wind?
          ┌───┴───┐              ┌──┴──┐
        High   Normal          Weak  Strong
         NO      YES            YES    NO
```

> **A decision tree is a model that repeatedly splits the data using the most informative question about a feature, until the groups at the bottom are (nearly) pure, then predicts from those groups.**

The algorithm's job is to **choose which question to ask at each step**. The answer comes from a measure of **purity**.

---

### 2. Anatomy of a tree

```text
                 [Root node]               ← the first question (all data)
                /           \
        [Internal node]   [Leaf]           ← leaf = final answer (prediction)
          /        \
      [Leaf]      [Leaf]
```

| Term | Meaning |
| --- | --- |
| **Root node** | The top node; contains all training data |
| **Internal (decision) node** | A test on one feature (for example *humidity = High?*) |
| **Branch** | One outcome of a test |
| **Leaf (terminal) node** | A final prediction |
| **Depth** | Number of levels from root to the deepest leaf |
| **Split** | Dividing a node's data by a feature test |

**How a tree predicts:** start at the root, answer each question for the new record, follow the branch, and stop at a leaf. The leaf's majority class (classification) or average value (regression) is the prediction.

---

### 3. Measuring purity: entropy

A node is **pure** if all its records belong to one class. We want splits that make the child nodes **purer** than the parent. **Entropy** measures **impurity (disorder)**.

```text
 Entropy(S) = − Σ pᵢ · log₂(pᵢ)        (sum over the classes, pᵢ = proportion of class i)
```

**Special cases**

| Node content | Entropy | Meaning |
| --- | --- | --- |
| All one class (pure) | **0** | No uncertainty |
| 50% / 50% | **1** | Maximum uncertainty (two classes) |
| 9 Yes / 5 No | **0.940** | Fairly mixed |

**Worked calculation for 9 Yes and 5 No (total 14):**

```text
 p(Yes) = 9/14 = 0.6429          p(No) = 5/14 = 0.3571

 Entropy = −[ 0.6429 × log₂(0.6429) + 0.3571 × log₂(0.3571) ]
         = −[ 0.6429 × (−0.6374)   + 0.3571 × (−1.4854)    ]
         = −[ −0.4098 − 0.5305 ] = 0.9403
```

```text
 Entropy
   1 │          ●            ← maximum at p = 0.5
     │        ╱   ╲
     │      ╱       ╲
   0 │──●──╱─────────╲──●──
        p=0   0.5    p=1
```

---

### 4. Gini impurity

Another impurity measure (the default in Spark). It is faster to compute (no logarithm).

```text
 Gini(S) = 1 − Σ pᵢ²
```

| Node content | Gini | Meaning |
| --- | --- | --- |
| Pure | **0** | |
| 50% / 50% (2 classes) | **0.5** | Maximum |
| 9 Yes / 5 No | **0.4592** | Mixed |

**Calculation for 9 Yes / 5 No:**

```text
 Gini = 1 − [ (9/14)² + (5/14)² ] = 1 − [ 0.4133 + 0.1276 ] = 0.4592
```

| | Entropy | Gini |
| --- | --- | --- |
| Range (2 classes) | 0 to 1 | 0 to 0.5 |
| Formula | `−Σ p log₂ p` | `1 − Σ p²` |
| Speed | Slightly slower (log) | Faster |
| Result | Almost always very similar trees | |
| Spark option | `impurity="entropy"` | `impurity="gini"` (default) |

---

### 5. Information gain

> **Information gain = impurity of the parent − weighted average impurity of the children.**

It tells us **how much a split reduces uncertainty**. We choose the feature with the **highest** information gain.

```text
 Gain(S, A) = Entropy(S) − Σ ( |Sᵥ| / |S| ) · Entropy(Sᵥ)
                           v ∈ values of A
```

| Symbol | Meaning |
| --- | --- |
| `S` | The data at the current node |
| `A` | A candidate feature for the split |
| `Sᵥ` | The subset of `S` where `A` has value `v` |
| `|Sᵥ|/|S|` | The weight (share of records) of that subset |

The same formula works with **Gini** in place of entropy (called **Gini gain**).

---

### 6. A complete worked example: the weather dataset

The classic 14-day "play tennis" data: predict **Play** (Yes/No).

| Day | Outlook | Temp | Humidity | Wind | Play |
| --- | --- | --- | --- | --- | --- |
| 1 | Sunny | Hot | High | Weak | No |
| 2 | Sunny | Hot | High | Strong | No |
| 3 | Overcast | Hot | High | Weak | Yes |
| 4 | Rain | Mild | High | Weak | Yes |
| 5 | Rain | Cool | Normal | Weak | Yes |
| 6 | Rain | Cool | Normal | Strong | No |
| 7 | Overcast | Cool | Normal | Strong | Yes |
| 8 | Sunny | Mild | High | Weak | No |
| 9 | Sunny | Cool | Normal | Weak | Yes |
| 10 | Rain | Mild | Normal | Weak | Yes |
| 11 | Sunny | Mild | Normal | Strong | Yes |
| 12 | Overcast | Mild | High | Strong | Yes |
| 13 | Overcast | Hot | Normal | Weak | Yes |
| 14 | Rain | Mild | High | Strong | No |

**Step 1: entropy of the whole dataset** (9 Yes, 5 No): **Entropy(S) = 0.9403**.

**Step 2: information gain for each feature**

#### Feature: Outlook

| Value | Days | Yes | No | Entropy |
| --- | --- | --- | --- | --- |
| Sunny | 5 | 2 | 3 | 0.9710 |
| Overcast | 4 | 4 | 0 | **0** (pure!) |
| Rain | 5 | 3 | 2 | 0.9710 |

```text
 Weighted entropy = (5/14)(0.9710) + (4/14)(0) + (5/14)(0.9710) = 0.6936
 Gain(Outlook) = 0.9403 − 0.6936 = 0.2467
```

#### Feature: Humidity

| Value | Days | Yes | No | Entropy |
| --- | --- | --- | --- | --- |
| High | 7 | 3 | 4 | 0.9852 |
| Normal | 7 | 6 | 1 | 0.5917 |

```text
 Weighted = (7/14)(0.9852) + (7/14)(0.5917) = 0.7885
 Gain(Humidity) = 0.9403 − 0.7885 = 0.1518
```

#### Feature: Wind

| Value | Days | Yes | No | Entropy |
| --- | --- | --- | --- | --- |
| Weak | 8 | 6 | 2 | 0.8113 |
| Strong | 6 | 3 | 3 | 1.0000 |

```text
 Weighted = (8/14)(0.8113) + (6/14)(1.0) = 0.8922
 Gain(Wind) = 0.9403 − 0.8922 = 0.0481
```

#### Feature: Temperature

| Value | Days | Yes | No | Entropy |
| --- | --- | --- | --- | --- |
| Hot | 4 | 2 | 2 | 1.0000 |
| Mild | 6 | 4 | 2 | 0.9183 |
| Cool | 4 | 3 | 1 | 0.8113 |

```text
 Weighted = (4/14)(1.0) + (6/14)(0.9183) + (4/14)(0.8113) = 0.9111
 Gain(Temp) = 0.9403 − 0.9111 = 0.0292
```

**Summary of gains** (verified by code)

| Feature | Information gain | Gini gain |
| --- | --- | --- |
| **Outlook** | **0.2467** | **0.1163** |
| Humidity | 0.1518 | 0.0918 |
| Wind | 0.0481 | 0.0306 |
| Temp | 0.0292 | 0.0187 |

**Decision: Outlook is the root**, because it has the highest gain under both measures. Notice that **Overcast is already pure** (4 Yes, 0 No), so that branch becomes a **leaf: YES**.

**Step 3: recurse on the impure branches**

*Sunny branch* (5 days: 2 Yes, 3 No). Test **Humidity**:

```text
 Sunny + High   : days 1, 2, 8   → No, No, No     (pure: NO)
 Sunny + Normal : days 9, 11     → Yes, Yes       (pure: YES)
 Gain = 0.9710 (the whole entropy is removed) → split on Humidity
```

*Rain branch* (5 days: 3 Yes, 2 No). Test **Wind**:

```text
 Rain + Weak   : days 4, 5, 10   → Yes, Yes, Yes  (pure: YES)
 Rain + Strong : days 6, 14      → No, No         (pure: NO)
 Gain = 0.9710 → split on Wind
```

**The finished tree** (14/14 training records classified correctly):

```text
                     Outlook?
              ┌─────────┼──────────┐
            Sunny    Overcast     Rain
              │          │          │
          Humidity?     YES       Wind?
          ┌───┴───┐              ┌──┴──┐
        High   Normal          Weak  Strong
         NO      YES            YES    NO
```

**Rules read from the tree** (each root-to-leaf path is an if-then rule):

```text
 IF Outlook = Overcast                          THEN Play = Yes
 IF Outlook = Sunny AND Humidity = High         THEN Play = No
 IF Outlook = Sunny AND Humidity = Normal       THEN Play = Yes
 IF Outlook = Rain  AND Wind = Weak             THEN Play = Yes
 IF Outlook = Rain  AND Wind = Strong           THEN Play = No
```

> Decision trees give **human-readable rules**, which is why they are popular when you must *explain* a decision.

---

### 7. The tree-building algorithm

```text
 BUILD(node data):
   1. If the node is pure, or a stopping rule is met → make a LEAF (majority class / mean)
   2. For every candidate split (feature, value), compute the impurity reduction (gain)
   3. Choose the split with the HIGHEST gain
   4. Divide the data into child nodes
   5. Repeat BUILD on each child
```

**Properties**

| Property | Explanation |
| --- | --- |
| **Greedy** | Picks the best split *now*, without looking ahead |
| **Recursive partitioning** | Splits, then splits the parts again |
| **Top-down** | Starts at the root |
| Variants | ID3 (entropy, multiway), C4.5, **CART** (Gini/variance, **binary** splits) |

**Spark uses CART-style binary splits.** Even for a categorical feature with three values it splits into two *groups* of values (for example `{Overcast}` vs `{Rain, Sunny}`). That is why Spark's tree on the weather data looks slightly different from our multiway hand tree (Section 13).

**Stopping rules** (to avoid growing forever): `maxDepth` reached, a node has too few records (`minInstancesPerNode`), or the gain is too small (`minInfoGain`).

---

### 8. Continuous features: threshold splits

For a **numeric** feature (hours = 1.0 … 10.0) the question is a **threshold**: *"hours ≤ 4.95?"*

```text
 To find the best threshold:
   1. Sort the feature values.
   2. Consider a threshold between each pair of neighbouring values (the midpoint).
   3. Compute the gain of each candidate; keep the best.
```

For huge data, Spark does not test every value. It groups each feature into at most **`maxBins`** (default 32) bins and tests only the bin boundaries. This makes training fast on big data, at a small cost in precision.

You will see thresholds such as `feature 0 <= 4.95` in the printed trees (Section 14).

---

### 9. Regression trees

A tree can also predict a **number**. The recipe changes in two places:

| | Classification tree | **Regression tree** |
| --- | --- | --- |
| Impurity | Entropy or Gini | **Variance** (or SSE/MSE) |
| Leaf prediction | Majority class | **Mean** of the labels in the leaf |
| Split goal | Maximize impurity reduction | Maximize **variance (SSE) reduction** |

**Worked example.** Our five students again: hours `1..5`, marks `40, 50, 55, 65, 75`. Total SSE around the mean (57) is **730**.

Try every threshold:

| Split | Left group (mean) → SSE | Right group (mean) → SSE | Total SSE |
| --- | --- | --- | --- |
| hours ≤ 1.5 | {40} (40) → 0 | {50,55,65,75} (61.25) → 368.75 | 368.75 |
| hours ≤ 2.5 | {40,50} (45) → 50 | {55,65,75} (65) → 200 | 250.0 |
| **hours ≤ 3.5** | {40,50,55} (**48.33**) → 116.67 | {65,75} (**70**) → 50 | **166.67** ← best |
| hours ≤ 4.5 | {40,50,55,65} (52.5) → 325 | {75} (75) → 0 | 325.0 |

The best split is **hours ≤ 3.5**: SSE drops from 730 to 166.67.

**Spark agrees** (✔ verified):

```python
from pyspark.ml.regression import DecisionTreeRegressor

m = DecisionTreeRegressor(maxDepth=1).fit(tiny)      # tiny = (hours, marks) data
print(m.toDebugString)
```

```text
  If (feature 0 <= 3.5)
   Predict: 48.333333333333336
  Else (feature 0 > 3.5)
   Predict: 70.0
```

Exactly our hand result: threshold **3.5**, leaf means **48.33** and **70.0**.

**Deeper trees:**

| maxDepth | Predictions for hours 1..5 | RMSE |
| --- | --- | --- |
| 1 | 48.33, 48.33, 48.33, 70, 70 | 5.7735 |
| 2 | 40, 52.5, 52.5, 65, 75 | 1.5811 |
| 3 | 40, 50, 55, 65, 75 | **0.0** |

At depth 3 the tree has **memorised** every point: RMSE 0 (overfitting). Also note that a regression tree makes **step-shaped (piecewise constant)** predictions, not a smooth line, so it cannot extrapolate beyond the training range.

---

### 10. Overfitting and how to control it

A tree allowed to grow without limits will keep splitting until every leaf is pure, **memorising noise**. We saw this in Phase 2:

```text
maxDepth    train   test
   1        0.855   0.743   underfit
   3        0.933   0.886   good
   5        0.988   0.886
   8        1.000   0.829   overfit (perfect training, worse test)
```

**Controls (hyperparameters)**

| Parameter | Effect |
| --- | --- |
| `maxDepth` | **Most important.** Lower = simpler tree |
| `minInstancesPerNode` | Each child needs at least this many records |
| `minInfoGain` | Ignore splits that gain less than this |
| `maxBins` | Resolution of numeric splits |
| **Pruning** (concept) | Grow big, then cut branches that do not help on validation data |
| **Ensembles** | Random Forest / GBT average many trees |

Choose `maxDepth` using **cross-validation** (Phase 11).

---

### 11. Feature importance

A tree can tell **which features mattered most**. Each time a feature is used to split, it reduces impurity. Spark adds up the reductions for each feature (weighted by how many records pass through the node) and normalizes them to **sum to 1**.

```python
print(model.featureImportances)
```

A vector like `[0.64, 0.22, 0.14]` means the first feature accounts for 64% of the total impurity reduction.

> Feature importance from a tree is a **quick guide**, not a causal proof. It can favour features with many distinct values.

---

### 12. PySpark code: weather data (categorical)

Categorical text features must be indexed first (Phase 3). We build one **Pipeline**.

```python
from pyspark.ml.feature import VectorAssembler, StringIndexer
from pyspark.ml import Pipeline
from pyspark.ml.classification import DecisionTreeClassifier
from pyspark.ml.evaluation import MulticlassClassificationEvaluator

W = [("Sunny","Hot","High","Weak","No"), ("Sunny","Hot","High","Strong","No"),
     ("Overcast","Hot","High","Weak","Yes"), ("Rain","Mild","High","Weak","Yes"),
     ("Rain","Cool","Normal","Weak","Yes"), ("Rain","Cool","Normal","Strong","No"),
     ("Overcast","Cool","Normal","Strong","Yes"), ("Sunny","Mild","High","Weak","No"),
     ("Sunny","Cool","Normal","Weak","Yes"), ("Rain","Mild","Normal","Weak","Yes"),
     ("Sunny","Mild","Normal","Strong","Yes"), ("Overcast","Mild","High","Strong","Yes"),
     ("Overcast","Hot","Normal","Weak","Yes"), ("Rain","Mild","High","Strong","No")]

wdf = spark.createDataFrame(W, ["outlook", "temp", "humidity", "wind", "play"])

cols = ["outlook", "temp", "humidity", "wind"]
indexers = [StringIndexer(inputCol=c, outputCol=c + "_i") for c in cols]
label_idx = StringIndexer(inputCol="play", outputCol="label")
assembler = VectorAssembler(inputCols=[c + "_i" for c in cols], outputCol="features")

dt = DecisionTreeClassifier(featuresCol="features", labelCol="label",
                            impurity="entropy", maxDepth=5, minInstancesPerNode=1)

model = Pipeline(stages=indexers + [label_idx, assembler, dt]).fit(wdf)
out = model.transform(wdf)

acc = MulticlassClassificationEvaluator(labelCol="label", metricName="accuracy").evaluate(out)
print("accuracy:", acc, " nodes:", model.stages[-1].numNodes)
```

```text
accuracy: 1.0   nodes: 13
```

**What the indexers decided** (needed to read the tree):

```text
outlook  : Rain=0, Sunny=1, Overcast=2
temp     : Mild=0, Cool=1, Hot=2
humidity : High=0, Normal=1
wind     : Weak=0, Strong=1
play     : Yes=0.0, No=1.0
```

(`StringIndexer` orders by frequency, so the most common value gets the smallest index.)

**Accuracy by depth** (training data):

| maxDepth | Accuracy | Nodes |
| --- | --- | --- |
| 2 | 0.857 | 5 |
| 3 | 0.857 | 7 |
| 5 | **1.000** | 13 |

---

### 13. Reading `toDebugString`

```python
print(model.stages[-1].toDebugString)
```

The tree for `maxDepth=5`:

```text
  If (feature 0 in {2.0})
   Predict: 0.0
  Else (feature 0 not in {2.0})
   If (feature 2 in {1.0})
    If (feature 3 in {0.0})
     Predict: 0.0
    Else (feature 3 not in {0.0})
     If (feature 0 in {1.0})
      Predict: 0.0
     Else (feature 0 not in {1.0})
      Predict: 1.0
   Else (feature 2 not in {1.0})
    If (feature 0 in {0.0})
     If (feature 3 in {0.0})
      Predict: 0.0
     Else (feature 3 not in {0.0})
      Predict: 1.0
    Else (feature 0 not in {0.0})
     Predict: 1.0
```

**Translation using the index mapping** (feature 0 = outlook, 2 = humidity, 3 = wind; prediction 0.0 = Yes, 1.0 = No):

| Tree line | Meaning |
| --- | --- |
| `If (feature 0 in {2.0})  Predict 0.0` | **Outlook = Overcast → Yes** ✔ (our pure leaf) |
| `Else … If (feature 2 in {1.0})` | Not Overcast, and **Humidity = Normal** |
| `… If (feature 3 in {0.0})  Predict 0.0` | Normal humidity and **Wind = Weak → Yes** |
| `… Else … If (feature 0 in {1.0})  Predict 0.0` | Normal humidity, **Strong** wind, **Sunny → Yes** (day 11) |
| `… Else  Predict 1.0` | Normal humidity, Strong wind, **Rain → No** (day 6) |
| `Else (feature 2 not in {1.0})` | Not Overcast, **Humidity = High** |
| `If (feature 0 in {0.0})` (Rain) → Weak → Yes, Strong → No | High humidity and **Rain**: depends on wind (days 4 and 14) |
| `Else Predict 1.0` | High humidity and **Sunny → No** (days 1, 2, 8) |

The root is **Outlook** with **Overcast isolated as a pure Yes**, exactly as our hand calculation found. The remaining structure differs from our hand tree only because Spark makes **binary** splits (several of them) rather than a three-way split.

**How to read `If (feature k in {…})` vs `<=`**

| Form | Used for |
| --- | --- |
| `feature k in {…}` | **Categorical** feature: value belongs to a set |
| `feature k <= value` | **Continuous** feature: threshold |

**Shallow tree (`maxDepth=3`)** has 7 nodes, **accuracy 0.857**: it cannot split every impure branch, so it makes 2 mistakes (for example day 6, Rain with Normal humidity and Strong wind). Feature importances of that shallow tree were `Outlook 0.632` and `Humidity 0.368`, so with only a few splits Wind and Temp are not used.

---

### 14. PySpark code: students (numeric)

Classify `passed` from `hours`, `attendance`, `prev_marks` (label from Phase 2).

```python
import sys
sys.path.append(r"C:\sparklabs")
from make_data import make_students
from pyspark.ml.feature import VectorAssembler
from pyspark.ml.classification import DecisionTreeClassifier
from pyspark.ml.evaluation import MulticlassClassificationEvaluator
from pyspark.sql import functions as F

df = spark.createDataFrame(make_students(),
        ["id", "hours", "attendance", "prev_marks", "final_marks", "passed"])
va = VectorAssembler(inputCols=["hours", "attendance", "prev_marks"], outputCol="features")
data = va.transform(df).select("features", F.col("passed").alias("label"))
train, test = data.randomSplit([0.8, 0.2], seed=42)

dt = DecisionTreeClassifier(featuresCol="features", labelCol="label", maxDepth=2, seed=42)
model = dt.fit(train)
print(model.toDebugString)
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

**Reading it** (feature 0 = hours, 2 = prev_marks):

```text
                hours ≤ 4.95 ?
               ┌──── yes ────┐──── no ────┐
        prev_marks ≤ 78.85 ?               PASS (1)
          ┌─ yes ─┐─ no ─┐
        FAIL (0)    PASS (1)
```

| Rule | Plain English |
| --- | --- |
| hours > 4.95 | Studying more than about **5 hours** → predicted **pass** |
| hours ≤ 4.95 and previous marks > 78.85 | Studies little but had strong previous marks → still **pass** |
| hours ≤ 4.95 and previous marks ≤ 78.85 | → **fail** |

This tree matches intuition (and the correlation heat map: hours is the dominant feature).

**A deeper tree and its importances**

```python
model = DecisionTreeClassifier(featuresCol="features", labelCol="label",
                               impurity="entropy", maxDepth=4, seed=42).fit(train)
pred = model.transform(test)
print("accuracy:", MulticlassClassificationEvaluator(labelCol="label", metricName="accuracy").evaluate(pred))
print("depth:", model.depth, "nodes:", model.numNodes)
print("importances:", model.featureImportances)
```

```text
accuracy: 0.8857
depth: 4  nodes: 15
importances: (3,[0,1,2],[0.6492733539306115,0.14121784382603797,0.2095088022433506])
```

| Feature | Importance |
| --- | --- |
| hours | **0.649** |
| prev_marks | 0.210 |
| attendance | 0.141 |

Importances sum to 1, and `hours` dominates, consistent with Phase 4's correlations.

**Comparison with Logistic Regression** (Phase 6) on the same split: tree accuracy 0.886 vs logistic 0.914. For data with a genuinely **linear** structure, logistic regression is hard to beat; trees shine when the structure is non-linear or categorical.

---

### 15. Gini vs entropy in Spark

```python
for impur in ("gini", "entropy"):
    m = DecisionTreeClassifier(featuresCol="features", labelCol="label",
                               impurity=impur, maxDepth=4, seed=42).fit(train)
    acc = MulticlassClassificationEvaluator(labelCol="label", metricName="accuracy").evaluate(m.transform(test))
    print(impur, "depth", m.depth, "nodes", m.numNodes, "accuracy", round(acc, 4))
```

```text
gini    depth 4 nodes 23 accuracy 0.8857
entropy depth 4 nodes 15 accuracy 0.8857
```

Both reach the **same test accuracy**; they simply grew slightly different trees (23 vs 15 nodes). In practice the choice rarely matters.

---

### 16. PySpark code: decision tree regressor

Predict `final_marks` (a number) instead of `passed`:

```python
from pyspark.ml.regression import DecisionTreeRegressor
from pyspark.ml.evaluation import RegressionEvaluator

rdata = va.transform(df).select("features", F.col("final_marks").alias("label"))
rtrain, rtest = rdata.randomSplit([0.8, 0.2], seed=42)

rm = DecisionTreeRegressor(featuresCol="features", labelCol="label", maxDepth=5, seed=42).fit(rtrain)
rp = rm.transform(rtest)

print("RMSE:", round(RegressionEvaluator(labelCol="label", metricName="rmse").evaluate(rp), 4))
print("R2  :", round(RegressionEvaluator(labelCol="label", metricName="r2").evaluate(rp), 4))
print(rm.featureImportances)
```

```text
RMSE: 6.298
R2  : 0.6518
(3,[0,1,2],[0.7572676773412393,0.0812086214720587,0.161523701186702])
```

**Compare with Linear Regression** (Phase 5) on the same split:

| Model | RMSE | R² |
| --- | --- | --- |
| **Linear Regression** | **2.84** | **0.929** |
| Decision Tree Regressor | 6.30 | 0.652 |

**Why is the tree worse here?** Our data was *generated* from a straight-line formula. A linear model matches it perfectly, while a tree must approximate a line with **steps**, needing many splits and lots of data. **Always match the model to the structure of the data**, and always compare against a simple baseline.

---

### 17. A peek at Random Forest

A single tree is unstable: a small change in data can change the tree a lot (high variance). A **random forest** trains **many trees** on random subsets of rows and features and **averages their votes**.

```text
   data ──▶ Tree 1 ─┐
        ──▶ Tree 2 ─┼──▶ majority vote / average ──▶ prediction
        ──▶ Tree 3 ─┘
```

```python
from pyspark.ml.classification import RandomForestClassifier

rf = RandomForestClassifier(featuresCol="features", labelCol="label", numTrees=20, maxDepth=4, seed=42).fit(train)
acc = MulticlassClassificationEvaluator(labelCol="label", metricName="accuracy").evaluate(rf.transform(test))
print(round(acc, 4), rf.featureImportances)
```

```text
0.8  (3,[0,1,2],[0.6571977083579126,0.16317107216611818,0.17963121947596913])
```

Here the forest scores 0.80 on a tiny 35-row test set, no better than the single tree. **That is normal**: with so few test rows, one or two students change the score by several percent, and forests pay off mainly on larger, noisier data. The feature importances again rank `hours` first.

| Single tree | Random forest |
| --- | --- |
| Easy to read | Harder to interpret |
| High variance (overfits) | Lower variance |
| Fast | Slower (many trees) |

---

### 18. Important parameters

| Parameter | Default | Meaning |
| --- | --- | --- |
| `maxDepth` | 5 | Maximum depth of the tree |
| `maxBins` | 32 | Number of bins for continuous features (must be ≥ number of categories of a categorical feature) |
| `minInstancesPerNode` | 1 | Minimum records in each child |
| `minInfoGain` | 0.0 | Minimum gain to accept a split |
| `impurity` | `gini` | `gini` or `entropy` (classification); `variance` (regression) |
| `seed` | random | For reproducibility |
| `featuresCol`, `labelCol`, `predictionCol` | standard names | |
| `probabilityCol`, `rawPredictionCol` | | Class probabilities from leaf proportions |

**Model attributes:** `depth`, `numNodes`, `numFeatures`, `numClasses`, `featureImportances`, `toDebugString`.

---

### 19. Strengths and weaknesses

| Strengths | Weaknesses |
| --- | --- |
| **Easy to explain**: readable rules | Prone to **overfitting** if deep |
| Handles numeric **and** categorical features | **Unstable**: small data changes alter the tree |
| **No scaling** needed | Piecewise-constant: poor with smooth linear trends |
| Captures **non-linear** patterns and interactions | Cannot extrapolate beyond training range (regression) |
| Gives feature importance | Greedy splitting may miss the globally best tree |
| Robust to outliers in features | Biased towards features with many values |

---

### 20. Common errors

| Symptom | Cause | Fix |
| --- | --- | --- |
| Train accuracy 100%, test much lower | Overfitting (deep tree) | Lower `maxDepth`, raise `minInstancesPerNode` |
| `maxBins ... must be at least as large as the number of values in each categorical feature` | A categorical feature has more categories than `maxBins` | Increase `maxBins` |
| Reading a categorical split wrongly | Mapping of indexes unknown | Print `StringIndexerModel.labels` |
| Text categories not handled | No `StringIndexer` | Index them first |
| Using `final_marks` to predict `passed` | Leakage | Remove it |
| Trees much worse than linear regression | Data is genuinely linear | Compare with a linear baseline |
| Different trees each run | No `seed` | Set `seed=42` |
| `Predict: 0.0` misread | The number is the **encoded label** | Map back with the label indexer's `labels` |
| Expecting smooth predictions | Trees give steps | Use linear/regularized models for smooth trends |
| Interpreting importance as causation | Importance is predictive, not causal | Be careful in conclusions |

---

### 21. Phase 7 — practical scenarios

1. Compute **entropy** for a node with 6 Yes and 2 No.
2. Compute **Gini** for the same node.
3. Compute the **information gain of Humidity** by hand and confirm 0.1518.
4. Decide the **root** of the weather tree and justify it with gains.
5. Draw the **complete tree** and write its **if-then rules**.
6. Build the weather tree in **Spark** with a Pipeline; print accuracy and `toDebugString`.
7. Translate a line of the debug string using the **indexer label mapping**.
8. Fit a **regression tree by hand** on hours `1..5` and marks `40,50,55,65,75` (best split 3.5).
9. Train a tree on **students** with `maxDepth` 1, 3 and 8; plot/print train vs test accuracy.
10. Print **feature importances** and rank the features.
11. Compare **gini vs entropy**.
12. Compare a **decision tree regressor** against **linear regression**, and explain the winner.

---

### 22. Phase 7 — viva questions and answers

#### Q1. What is a decision tree?

A model that recursively splits data using feature tests into a tree of decisions, with predictions at the leaves.

#### Q2. Define root node, internal node, leaf and depth.

The root is the first split containing all data; internal nodes are tests; leaves give predictions; depth is the number of levels from root to the deepest leaf.

#### Q3. How does a decision tree choose a split?

It picks the feature and threshold that give the largest reduction in impurity (highest information gain).

#### Q4. What is entropy? Give the formula and range.

A measure of impurity: `−Σ p log₂ p`; 0 for a pure node and 1 for a 50/50 two-class node.

#### Q5. What is Gini impurity?

`1 − Σ p²`; 0 for a pure node and 0.5 for a 50/50 two-class node.

#### Q6. What is information gain?

Parent impurity minus the weighted average impurity of the children.

#### Q7. In the weather dataset, which feature is the root and why?

Outlook, because its information gain (0.2467) is the highest; Overcast also gives a pure Yes branch.

#### Q8. What is the entropy of 9 Yes and 5 No?

About 0.940.

#### Q9. How are splits made for continuous features?

By testing thresholds between neighbouring sorted values (Spark uses at most `maxBins` candidate thresholds).

#### Q10. What kind of splits does Spark make?

Binary splits (CART style), also for categorical features, where it divides values into two groups.

#### Q11. How does a regression tree differ from a classification tree?

It uses variance (SSE) reduction as the split criterion and predicts the mean of the training labels in each leaf.

#### Q12. Why do decision trees overfit?

Unlimited growth lets them create leaves for individual records, memorising noise.

#### Q13. How do you control overfitting in a tree?

Limit `maxDepth`, raise `minInstancesPerNode` or `minInfoGain`, prune, or use ensembles.

#### Q14. What is `maxBins`?

The maximum number of bins used to discretize continuous features (and a limit on categorical arity), trading precision for speed.

#### Q15. What are feature importances?

Normalised totals of impurity reduction contributed by each feature across the tree; they sum to 1.

#### Q16. Do decision trees need feature scaling?

No, because splits depend on ordering and thresholds, not on distances.

#### Q17. What are the main strengths of decision trees?

Interpretability, handling mixed feature types, no scaling, capturing non-linearity.

#### Q18. What are the main weaknesses?

Overfitting, instability, step-wise predictions and no extrapolation.

#### Q19. What is a random forest?

An ensemble of many trees trained on random subsets of data and features, combining their votes or averages to reduce variance.

#### Q20. In our tiny regression-tree example, what split did the tree choose and what were the leaf values?

`hours ≤ 3.5`, with leaf predictions 48.33 (left) and 70.0 (right).

---

### 23. Phase 7 — final master recap

```text
DECISION TREE          root → internal nodes (tests) → leaves (predictions)
IMPURITY               Entropy = −Σ p log₂ p   (0..1)        Gini = 1 − Σ p²   (0..0.5)
GAIN                   Gain = Impurity(parent) − Σ (|child|/|parent|)·Impurity(child)
WEATHER EXAMPLE        H(S)=0.9403 ; Gain: Outlook 0.2467 > Humidity 0.1518 > Wind 0.0481 > Temp 0.0292
                       root = Outlook ; Overcast = pure YES ; Sunny→Humidity ; Rain→Wind
REGRESSION TREE        split by SSE reduction ; leaf = mean ; hours≤3.5 → 48.33 | 70.0
CONTROL OVERFITTING    maxDepth · minInstancesPerNode · minInfoGain · pruning · Random Forest
STUDENTS               depth-2 rule: hours ≤ 4.95 ? (then prev_marks ≤ 78.85) ; importance hours 0.65
PYSPARK                DecisionTreeClassifier(maxDepth, impurity, seed).fit(train) ; .toDebugString ; .featureImportances
```

**Seven facts to memorise**

1. Choose the split with the **highest information gain**.
2. Entropy 0 = pure; entropy 1 = maximum mix (two classes).
3. Gain = parent impurity − weighted children impurity.
4. Spark trees use **binary** splits; Gini is the default impurity.
5. Regression tree: variance reduction; leaf = **mean**.
6. Deep trees overfit; control with `maxDepth`.
7. Trees need **no scaling** and give readable rules.

### Phase 7 completed

---

> **Next → Phase 8: K-means Clustering** (unsupervised learning, the algorithm and its objective WCSS, a full hand iteration on 7 points, k-means++ initialization, choosing K with the elbow method and silhouette, scaling, PySpark code and cluster interpretation). File: `8_Unit6_KMeans_Clustering.md`
