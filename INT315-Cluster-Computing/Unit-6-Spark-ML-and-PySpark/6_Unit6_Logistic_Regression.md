# Unit VI — Phase 6: Logistic Regression

> **Continuing from Phase 5.** Linear regression predicts a number. **Logistic regression predicts a category** (pass/fail, spam/not spam), by predicting a **probability**. We build it from the ground up: why a straight line fails for classes, the **sigmoid**, **odds and log-odds**, the **log-loss** derived from likelihood, gradient descent **coded from scratch**, the decision threshold, multiclass **softmax**, and the PySpark code.
>
> All numbers here were computed for real: the hand-worked tiny example, our from-scratch gradient descent and Spark's result all agree.

---

## Phase 6 — Roadmap

| # | Topic |
| --- | --- |
| 1 | The problem: why not linear regression? |
| 2 | The sigmoid function |
| 3 | The logistic regression model |
| 4 | Odds and log-odds |
| 5 | The decision boundary |
| 6 | The loss function: log-loss |
| 7 | The gradient |
| 8 | A complete tiny example |
| 9 | Gradient descent from scratch |
| 10 | PySpark code: the tiny example |
| 11 | PySpark code: pass/fail on our students |
| 12 | Probabilities, `rawPrediction` and the manual sigmoid check |
| 13 | Interpreting coefficients as odds ratios |
| 14 | The decision threshold |
| 15 | Model summary |
| 16 | Regularization |
| 17 | Multiclass: softmax |
| 18 | Important parameters |
| 19 | Strengths and weaknesses |
| 20 | Common errors |
| 21 | Practical scenarios |
| 22 | Viva questions |
| 23 | Master revision |

---

### 1. The problem: why not linear regression?

We want to predict `passed` (0 or 1) from `hours`. What if we use linear regression anyway?

```text
 passed
  1 │            ● ● ●   ●  ●
    │        ●        /
 0.5│               /       ← a straight line keeps going
    │         /   ●
  0 │ ● ● ● /                
 -0.4│   /                    ← predicts -0.4 ??
    └──────────────────────── hours
```

| Problem | Why it hurts |
| --- | --- |
| Predictions can be **below 0 or above 1** | A "probability" of −0.4 or 1.7 is meaningless |
| The relationship is an **S-shape**, not a line | Chance of passing rises slowly, then fast, then flattens |
| Outliers drag the line | A few extreme students distort every prediction |

**What we need:** a function that takes any number (−∞ to +∞) and squeezes it into the range **0 to 1**, so it can be read as a **probability**. That function is the **sigmoid**.

---

### 2. The sigmoid function

```text
              1
 σ(z)  =  ─────────
           1 + e⁻ᶻ
```

```text
 σ(z)
  1 │                  ──────────────  → approaches 1
    │              ╱
0.5 │            ╱             ← σ(0) = 0.5
    │         ╱
  0 │ ───────                  → approaches 0
    └────────────────────────── z
       -4   -2   0   2   4
```

**Values (computed)**

| z | σ(z) | Meaning |
| --- | --- | --- |
| −4 | 0.018 | Almost certainly class 0 |
| −2 | 0.119 | Probably 0 |
| −1 | 0.269 | |
| **0** | **0.500** | **Undecided** |
| 1 | 0.731 | |
| 2 | 0.881 | Probably 1 |
| 4 | 0.982 | Almost certainly 1 |

**Properties**

1. Output always strictly between **0 and 1**.
2. `σ(0) = 0.5`.
3. Symmetric: `σ(−z) = 1 − σ(z)`.
4. Smooth S-shape; steepest at z = 0.
5. Derivative: `σ′(z) = σ(z)(1 − σ(z))` (used to derive the gradient).

---

### 3. The logistic regression model

**Step 1:** compute a linear score exactly as in linear regression:

```text
 z = β₀ + β₁x₁ + β₂x₂ + … + βₖxₖ
```

**Step 2:** pass it through the sigmoid to get the probability of class 1:

```text
 p = P(y = 1 | x) = σ(z) = 1 / (1 + e^−(β₀ + β₁x₁ + … + βₖxₖ))
```

**Step 3:** convert the probability into a class using a **threshold** (default 0.5):

```text
 predict 1  if  p ≥ 0.5        (equivalently  z ≥ 0)
 predict 0  otherwise
```

```text
 features ──▶ linear score z ──▶ sigmoid ──▶ probability p ──▶ threshold ──▶ class
   x₁,x₂        β·x + β₀         σ(z)       0.87              ≥ 0.5         1 (pass)
```

> **Name trap:** it is called *regression* because it regresses (fits) a linear function inside the sigmoid, but the task it solves is **classification**.

---

### 4. Odds and log-odds

This explains **where the sigmoid comes from** and how to interpret coefficients.

**Odds** = probability of success divided by probability of failure:

```text
              p
 odds  =  ───────
            1 − p
```

| p | odds | Meaning |
| --- | --- | --- |
| 0.5 | 1 | Even chance |
| 0.8 | 4 | 4 to 1 in favour |
| 0.2 | 0.25 | 1 to 4 against |

**Log-odds (logit)** = natural log of the odds:

```text
 logit(p) = ln( p / (1 − p) )
```

**The key identity.** Logistic regression assumes the **log-odds are a linear function of the features**:

```text
 ln( p / (1 − p) ) = β₀ + β₁x₁ + … + βₖxₖ  = z
```

Solve this for p and you get the sigmoid: `p = 1 / (1 + e^−z)`. So the sigmoid is simply the **inverse of the logit**.

```text
 probability p ──logit──▶ log-odds z (any real number) ──sigmoid──▶ probability p
```

---

### 5. The decision boundary

The model switches from class 0 to class 1 where **p = 0.5**, i.e. **z = 0**.

```text
 β₀ + β₁x₁ + β₂x₂ = 0      ← the decision boundary
```

- With **one feature**: a single cut-off value `x* = −β₀ / β₁`.
- With **two features**: a straight **line** in the feature plane.
- With more: a flat hyperplane.

> **Logistic regression has a *linear* decision boundary.** It cannot separate classes that need a curved boundary (a decision tree or a kernel method could).

---

### 6. The loss function: log-loss

**Why not use squared error (MSE)?** With a sigmoid inside, squared error becomes a bumpy, non-convex function that gradient descent can get stuck on. We use a loss derived from **probability** instead.

**Likelihood.** For one example with true label `y` and predicted probability `p`:

```text
 P(y | x) = pʸ (1 − p)¹⁻ʸ        (if y=1 → p ; if y=0 → 1−p)
```

For all n independent examples, the **likelihood** is the product:

```text
 L(β) = Π pᵢʸⁱ (1 − pᵢ)¹⁻ʸⁱ
```

**Maximum likelihood** means: choose β that makes the observed labels as probable as possible. Products are awkward, so take the logarithm (turning products into sums), and flip the sign to get something to *minimize*:

```text
 Loss  =  − (1/n) Σ [ yᵢ ln(pᵢ) + (1 − yᵢ) ln(1 − pᵢ) ]
```

This is called **log-loss** or **binary cross-entropy**.

**How it behaves** (true label y = 1; loss = −ln p)

| Predicted p (for the true class) | Loss | Meaning |
| --- | --- | --- |
| 0.99 | 0.0101 | Confident and right → tiny loss |
| 0.90 | 0.1054 | |
| 0.50 | 0.6931 | Pure guess = ln 2 |
| 0.10 | 2.3026 | Wrong and fairly confident → big loss |
| 0.01 | 4.6052 | Confidently **wrong** → huge loss |

> The loss **punishes confident mistakes severely**, which pushes the model to produce well-calibrated probabilities.

**Convex:** log-loss with a sigmoid is a convex function of β, so gradient descent reaches the single global minimum.

---

### 7. The gradient

Using `σ′(z) = σ(z)(1 − σ(z))` and the chain rule, the derivative of the log-loss simplifies beautifully:

```text
 ∂Loss/∂βⱼ = (1/n) Σ (pᵢ − yᵢ) · xᵢⱼ
 ∂Loss/∂β₀ = (1/n) Σ (pᵢ − yᵢ)
```

**Compare with linear regression's gradient** `(2/n) Σ (ŷᵢ − yᵢ) xᵢⱼ` (Phase 5): the same "prediction minus truth, times feature" structure. Only the prediction changed (from `β·x` to `σ(β·x)`).

**Update rule**

```text
 βⱼ ← βⱼ − η · (1/n) Σ (pᵢ − yᵢ) xᵢⱼ
```

---

### 8. A complete tiny example

Eight students: study hours and pass (1) or fail (0).

| h | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| passed | 0 | 0 | 0 | 1 | 0 | 1 | 1 | 1 |

(Student with 5 hours failed and student with 4 hours passed. The classes **overlap**, as in real life.)

**Starting point: β₁ = 0, β₀ = 0.** Then z = 0 for everyone, so every `p = σ(0) = 0.5`.

```text
 Log-loss at the start = −ln(0.5) = 0.6931       (a pure guess)
```

**Gradient at the start**

```text
 ∂/∂β₁ = (1/8) Σ (0.5 − yᵢ)·xᵢ
       = (1/8) [ 0.5×(1+2+…+8) − (4 + 6 + 7 + 8) ]
       = (1/8) [ 0.5×36 − 25 ] = (1/8)(18 − 25) = −0.875

 ∂/∂β₀ = (1/8) Σ (0.5 − yᵢ) = (1/8)(4 − 4) = 0
```

With learning rate η = 0.1: `β₁ ← 0 − 0.1×(−0.875) = 0.0875`, `β₀ ← 0`.

The negative gradient means "increase β₁": more hours should raise the pass probability. ✔

**After many steps, the model converges to** (found by Spark and by our from-scratch code, next sections):

```text
 β₁ = 1.2823      β₀ = −5.7703
 z  = 1.2823·h − 5.7703
```

**Decision boundary:** `z = 0 ⟹ h* = 5.7703 / 1.2823 = 4.5` hours.

**Probabilities from the model**

```text
 h = 2 :  z = 1.2823×2 − 5.7703 = −3.2057  →  p = 1/(1+e^3.2057) = 0.039   → FAIL
 h = 4.5: z = 0                             →  p = 0.5                      → on the boundary
 h = 6 :  z = 1.2823×6 − 5.7703 =  1.9234  →  p = 1/(1+e^−1.9234) = 0.8725  → PASS
```

**Odds ratio:** `e^β₁ = e^1.2823 = 3.605`. Each extra study hour **multiplies the odds of passing by about 3.6**.

---

### 9. Gradient descent from scratch

```python
import numpy as np

x = np.array([1, 2, 3, 4, 5, 6, 7, 8], dtype=float)
y = np.array([0, 0, 0, 1, 0, 1, 1, 1], dtype=float)

def sigmoid(z):
    return 1 / (1 + np.exp(-z))

def logloss(w, b):
    p = sigmoid(w * x + b)
    return -np.mean(y * np.log(p) + (1 - y) * np.log(1 - p))

w = b = 0.0
lr = 0.1

for it in range(1, 100001):
    p = sigmoid(w * x + b)
    gw = np.mean((p - y) * x)         # gradient wrt weight
    gb = np.mean(p - y)               # gradient wrt bias
    w -= lr * gw
    b -= lr * gb
    if it in (1, 2, 3, 10, 100, 1000, 10000, 100000):
        print("iteration", it, "w", round(w, 4), "b", round(b, 4), "log-loss", round(logloss(w, b), 4))
```

**Output**

```text
iteration 1       w 0.0875  b 0.0      log-loss 0.6407
iteration 2       w 0.1207  b -0.0096  log-loss 0.6316
iteration 3       w 0.1359  b -0.0224  log-loss 0.6281
iteration 10      w 0.1651  b -0.1242  log-loss 0.6119
iteration 100     w 0.3675  b -1.196   log-loss 0.4782
iteration 1000    w 0.9971  b -4.3722  log-loss 0.322
iteration 10000   w 1.282   b -5.7689  log-loss 0.3129
iteration 100000  w 1.2823  b -5.7703  log-loss 0.3129
```

**Reading it**

- Iteration 1 gives **w = 0.0875** ✔ exactly our hand step.
- The log-loss falls steadily from **0.6931** to **0.3129**.
- It converges to **w = 1.2823, b = −5.7703**, the same numbers Spark finds.

---

### 10. PySpark code: the tiny example

```python
from pyspark.ml.feature import VectorAssembler
from pyspark.ml.classification import LogisticRegression

h = [(1.0, 0), (2.0, 0), (3.0, 0), (4.0, 1), (5.0, 0), (6.0, 1), (7.0, 1), (8.0, 1)]
df = spark.createDataFrame(h, ["h", "label"])
df = VectorAssembler(inputCols=["h"], outputCol="features").transform(df)

model = LogisticRegression(maxIter=200, regParam=0.0).fit(df)
print(model.coefficients, model.intercept)
```

```text
[1.2823] -5.7703
```

✔ **Spark: w = 1.2823, b = −5.7703**, identical to the hand/scratch result.

```python
model.transform(df).select("h", "label", "probability", "prediction").show(truncate=False)
```

```text
+---+-----+-----------------------------------------+----------+
|h  |label|probability                              |prediction|
+---+-----+-----------------------------------------+----------+
|1.0|0    |[0.9888821743530577,0.011117825646942303]|0.0       |
|2.0|0    |[0.9610494311179487,0.038950568882051284]|0.0       |
|3.0|0    |[0.8725214897245442,0.12747851027545576] |0.0       |
|4.0|1    |[0.6550125810107279,0.3449874189892721]  |0.0       |
|5.0|0    |[0.34498741898924645,0.6550125810107535] |1.0       |
|6.0|1    |[0.12747851027544302,0.872521489724557]  |1.0       |
|7.0|1    |[0.03895056888204695,0.961049431117953]  |1.0       |
|8.0|1    |[0.011117825646941023,0.988882174353059] |1.0       |
+---+-----+-----------------------------------------+----------+
```

**Reading it**

- `probability` is a **two-element vector**: `[P(class 0), P(class 1)]`, summing to 1.
- Row `h = 6`: `P(1) = 0.8725` ✔ matches our hand value 0.8725.
- Two rows are wrong, exactly the overlap students: `h = 4` (actual 1, predicted 0, since P(1) = 0.345) and `h = 5` (actual 0, predicted 1, since P(1) = 0.655).
- Training accuracy is therefore **6/8 = 0.75**, and the AUC is 0.9375.

---

### 11. PySpark code: pass/fail on our students

**Step 1: load, assemble, split**

```python
import sys
sys.path.append(r"C:\sparklabs")
from make_data import make_students
from pyspark.ml.feature import VectorAssembler
from pyspark.ml.classification import LogisticRegression
from pyspark.ml.evaluation import BinaryClassificationEvaluator, MulticlassClassificationEvaluator
from pyspark.sql import functions as F

df = spark.createDataFrame(make_students(),
        ["id", "hours", "attendance", "prev_marks", "final_marks", "passed"])

va = VectorAssembler(inputCols=["hours", "attendance", "prev_marks"], outputCol="features")
data = va.transform(df).select("features", F.col("passed").alias("label"))      # NOT final_marks (leakage!)

train, test = data.randomSplit([0.8, 0.2], seed=42)
```

> **Leakage reminder:** the label is `passed`. We must **not** put `final_marks` in the features, because `passed` is computed from it.

**Step 2: train**

```python
lr = LogisticRegression(featuresCol="features", labelCol="label", maxIter=100)
model = lr.fit(train)

print("coefficients:", model.coefficients)
print("intercept   :", model.intercept)
print("numClasses  :", model.numClasses, " threshold:", model.getThreshold())
```

```text
coefficients: [4.172291336144106,0.23594656267082154,0.41963822857155086]
intercept   : -65.82448682397785
numClasses  : 2  threshold: 0.5
```

**Step 3: predict**

```python
pred = model.transform(test)
pred.select("features", "label", "rawPrediction", "probability", "prediction").show(4, truncate=False)
```

```text
+---------------+-----+----------------------------------------+------------------------------------------+----------+
|features       |label|rawPrediction                           |probability                               |prediction|
+---------------+-----+----------------------------------------+------------------------------------------+----------+
|[1.5,79.2,62.7]|0    |[14.567765124796388,-14.567765124796388]|[0.9999995286974518,4.713025482105948E-7] |0.0       |
|[1.6,88.0,77.1]|0    |[6.0314157482484205,-6.0314157482484205]|[0.99760366539902,0.002396334600979988]   |0.0       |
|[1.8,52.4,41.0]|0    |[28.745595163533828,-28.745595163533828]|[0.999999999999672,3.2795988147427124E-13]|0.0       |
|[2.8,82.5,65.0]|0    |[7.39999480528077,-7.39999480528077]    |[0.9993891174691493,6.108825308507315E-4] |0.0       |
+---------------+-----+----------------------------------------+------------------------------------------+----------+
```

**Step 4: evaluate**

```python
auc = BinaryClassificationEvaluator(labelCol="label", metricName="areaUnderROC").evaluate(pred)
print("AUC:", auc)
for m in ("accuracy", "f1", "weightedPrecision", "weightedRecall"):
    print(m, round(MulticlassClassificationEvaluator(labelCol="label", metricName=m).evaluate(pred), 4))
```

```text
AUC: 0.988
accuracy 0.9143
f1 0.9171
weightedPrecision 0.9341
weightedRecall 0.9143
```

**Step 5: the confusion table (counts of each actual/predicted pair)**

```python
pred.groupBy("label", "prediction").count().orderBy("label", "prediction").show()
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

**Reading it:** 35 test students: 25 actually failed (22 correctly predicted, 3 wrongly predicted to pass) and 10 actually passed (all 10 correct). Accuracy = (22 + 10) / 35 = **91.4%** ✔.

---

### 12. Probabilities, `rawPrediction` and the manual sigmoid check

Three output columns appear:

| Column | Content | Meaning |
| --- | --- | --- |
| `rawPrediction` | `[−z, +z]` | The linear score `z` (log-odds), once negative and once positive |
| `probability` | `[P(0), P(1)]` | `σ(z)` applied; sums to 1 |
| `prediction` | 0.0 or 1.0 | Class chosen by the threshold |

**Verify by hand with the first test row** `[1.5, 79.2, 62.7]`:

```python
import math
row = test.first()
z = float(model.intercept) + sum(float(c) * float(x) for c, x in zip(model.coefficients, row.features))
print("z =", z, " sigmoid =", 1 / (1 + math.exp(-z)))
print("model:", model.predictProbability(row.features))
```

```text
z = -14.56776512479638   sigmoid = 4.7130254828719454e-07
model: [0.9999995286974518,4.713025482105948e-07]
```

Calculation:

```text
 z = −65.8245 + 4.1723×1.5 + 0.2359×79.2 + 0.4196×62.7 = −14.568
 P(pass) = 1/(1 + e^14.568) = 0.00000047   ≈ 0        → predict FAIL (0) ✔
```

---

### 13. Interpreting coefficients as odds ratios

Each coefficient is the change in the **log-odds** for a one-unit increase in that feature. Exponentiating gives the **odds ratio**:

```text
 odds ratio = e^β
```

```python
import math
print([round(math.exp(float(c)), 3) for c in model.coefficients])
```

```text
[64.864, 1.266, 1.521]
```

| Feature | β | Odds ratio e^β | Meaning |
| --- | --- | --- | --- |
| hours | 4.1723 | **64.9** | Each extra study hour multiplies the odds of passing by ~65 |
| attendance | 0.2359 | 1.27 | Each +1% attendance multiplies the odds by 1.27 |
| prev_marks | 0.4196 | 1.52 | Each +1 previous mark multiplies the odds by 1.52 |

**Cautions**

1. The huge odds ratio for hours appears because our classes are **nearly separable** by `final_marks` (which depends on hours). In real, messier data, odds ratios are smaller.
2. A positive β raises the probability of class 1; a negative β lowers it.
3. Compare effects fairly only after **standardizing** the features.

---

### 14. The decision threshold

The default rule is *predict 1 if p ≥ 0.5*. But we can **move the threshold** to trade off the two kinds of mistake.

```python
LogisticRegression(maxIter=100, threshold=0.7)
```

On the earlier split, raising the threshold to 0.7 changed test accuracy from 0.9143 to **0.9429** (fewer false passes).

**Threshold sweep** (a 70:30 split with `seed=7`, 133 train / 67 test rows; positive class = pass):

```python
tr2, te2 = data.randomSplit([0.7, 0.3], seed=7)
for t in (0.3, 0.5, 0.7, 0.9):
    p = LogisticRegression(maxIter=100, threshold=t).fit(tr2).transform(te2)
    # precision and recall for class 1 computed from the counts
```

```text
threshold   precision(pass)   recall(pass)
   0.3          0.912             0.969
   0.5          0.939             0.969
   0.7          0.967             0.906
   0.9          0.963             0.812
```

| Move the threshold… | Effect |
| --- | --- |
| **Lower** (0.3) | More students predicted "pass": **higher recall**, lower precision |
| **Higher** (0.9) | Fewer predicted "pass": **higher precision**, lower recall |

**Which to choose?** It depends on the cost of mistakes:

- Cancer screening: a missed case is terrible → **low threshold** (high recall).
- Spam filter: wrongly deleting a real email is worse → **high threshold** (high precision).

> Precision and recall are explained fully in Phase 11.

---

### 15. Model summary

```python
s = model.summary
print("train accuracy:", round(s.accuracy, 4))
print("train AUC     :", round(s.areaUnderROC, 4))
print("iterations    :", s.totalIterations)
print("precision/label:", [round(x, 3) for x in s.precisionByLabel])
print("recall/label   :", [round(x, 3) for x in s.recallByLabel])
print("F1/label       :", [round(x, 3) for x in s.fMeasureByLabel()])
```

```text
train accuracy: 0.9636
train AUC     : 0.9954
iterations    : 19
precision/label: [0.948, 0.977]
recall/label   : [0.973, 0.956]
F1/label       : [0.961, 0.966]
```

(Lists are ordered `[class 0, class 1]`.) Training accuracy 0.964 vs test 0.914: a small gap, so no serious overfitting.

| `summary` attribute | Meaning |
| --- | --- |
| `accuracy`, `areaUnderROC` | Overall metrics on training data |
| `precisionByLabel`, `recallByLabel`, `fMeasureByLabel()` | Per-class metrics |
| `roc` | DataFrame of ROC curve points (FPR, TPR) |
| `pr` | Precision-recall curve |
| `objectiveHistory` | Loss per iteration |

---

### 16. Regularization

Like linear regression, logistic regression can add a penalty to the log-loss to stop large coefficients:

```text
 Loss_reg = LogLoss + λ [ α Σ|βⱼ| + (1 − α) Σ βⱼ² / 2 ]
```

Spark uses the same parameters: `regParam` = λ, `elasticNetParam` = α (0 = L2/Ridge, 1 = L1/Lasso).

```python
m = LogisticRegression(maxIter=100, regParam=0.1, elasticNetParam=1.0).fit(train)   # Lasso
print(m.coefficients, m.intercept)
```

```text
[0.509302554176907,0.009005977506358662,0.019867827254488036]  -4.559012071132378
```

The coefficients shrink dramatically (hours 4.17 → 0.51) because the penalty is strong. Regularization is especially valuable when classes are **perfectly separable** (without it, coefficients grow without limit).

---

### 17. Multiclass: softmax

For **more than two classes**, Spark uses **multinomial logistic regression** (`family="multinomial"`). Instead of one score, there is **one score per class**, and the **softmax** function converts the scores into probabilities that sum to 1:

```text
                     e^(zₖ)
 P(class k)  =  ───────────────          zₖ = β₀ₖ + β₁ₖx₁ + … (separate weights per class)
                  Σⱼ e^(zⱼ)
```

**Worked softmax example:** scores for three classes `z = [2.0, 1.0, 0.1]`:

```text
 e^2.0 = 7.389    e^1.0 = 2.718    e^0.1 = 1.105      sum = 11.213

 P = [ 7.389/11.213 , 2.718/11.213 , 1.105/11.213 ] = [0.659, 0.242, 0.099]
```

Class 0 wins with probability 0.659. (For two classes, softmax reduces to the sigmoid.)

**Spark code on the classic Iris data** (a 15-row sample: 5 flowers per species; features: sepal length, sepal width, petal length, petal width)

```python
fl = [(5.1,3.5,1.4,0.2,0),(4.9,3.0,1.4,0.2,0),(4.7,3.2,1.3,0.2,0),(4.6,3.1,1.5,0.2,0),(5.0,3.6,1.4,0.2,0),
      (7.0,3.2,4.7,1.4,1),(6.4,3.2,4.5,1.5,1),(6.9,3.1,4.9,1.5,1),(5.5,2.3,4.0,1.3,1),(6.5,2.8,4.6,1.5,1),
      (6.3,3.3,6.0,2.5,2),(5.8,2.7,5.1,1.9,2),(7.1,3.0,5.9,2.1,2),(6.3,2.9,5.6,1.8,2),(6.5,3.0,5.8,2.2,2)]

fdf = spark.createDataFrame(fl, ["sl", "sw", "pl", "pw", "label"])
fv = VectorAssembler(inputCols=["sl", "sw", "pl", "pw"], outputCol="features").transform(fdf)

mm = LogisticRegression(featuresCol="features", labelCol="label", maxIter=200, family="multinomial").fit(fv)

print("classes:", mm.numClasses)
print(mm.coefficientMatrix)
print(mm.interceptVector)
```

```text
classes: 3
DenseMatrix([[-12.29898463,  15.50908608,  -6.49046221, -13.92407803],
             [ 17.50373974, -11.61503615,  -5.73721191, -14.79571955],
             [ -5.2047551 ,  -3.89404993,  12.22767412,  28.71979758]])
[70.67382385346687,-14.51381805119006,-56.160005802276814]
```

**Reading the matrix:** one **row per class**, one **column per feature**. For class 2 (virginica), petal length has weight +12.2 and petal width +28.7: larger petals push the score towards virginica. For class 0 (setosa) the petal weights are negative: large petals argue *against* setosa.

```python
mm.transform(fv).select("label", "prediction", "probability").show(3, truncate=False)
```

```text
+-----+----------+------------------------------------------------------------------+
|label|prediction|probability                                                       |
+-----+----------+------------------------------------------------------------------+
|0    |0.0       |[0.9999999999985332,1.4667774722172753E-12,1.6645018847280626E-54]|
|0    |0.0       |[0.999999997064644,2.935356053519136E-9,6.583014209786902E-51]    |
|0    |0.0       |[0.9999999999999691,3.0927475682438534E-14,5.0579500298272984E-54]|
+-----+----------+------------------------------------------------------------------+
```

The probability vector has **three entries** (one per class). The training accuracy is **1.0** (the classes are well separated; we are not claiming this on unseen data).

| Approach | Description |
| --- | --- |
| **Multinomial (softmax)** | One model with a score per class; probabilities sum to 1 |
| **One-vs-Rest** | Train one binary classifier per class (`OneVsRest`, Phase 9) |

---

### 18. Important parameters

| Parameter | Default | Meaning |
| --- | --- | --- |
| `maxIter` | 100 | Maximum iterations |
| `regParam` | 0.0 | Regularization strength λ |
| `elasticNetParam` | 0.0 | 0 = L2, 1 = L1 |
| `family` | `auto` | `binomial`, `multinomial` (auto picks by number of classes) |
| `threshold` | 0.5 | Probability cut-off (binary) |
| `thresholds` | none | Per-class thresholds (multiclass) |
| `standardization` | True | Standardize features internally |
| `fitIntercept` | True | Learn β₀ |
| `tol` | 1e-6 | Convergence tolerance |
| `weightCol` | none | Row weights (useful for class imbalance) |

---

### 19. Strengths and weaknesses

| Strengths | Weaknesses |
| --- | --- |
| Outputs **probabilities**, not just classes | **Linear** decision boundary only |
| Fast, scalable, simple baseline | Needs feature **scaling** for best regularization behaviour |
| **Interpretable** (odds ratios) | Sensitive to multicollinearity |
| Regularization built in | Struggles with complex non-linear patterns |
| Handles multiclass via softmax | Can fail to converge with perfect separation (without regularization) |

---

### 20. Common errors

| Symptom | Cause | Fix |
| --- | --- | --- |
| `Labels MUST be in [0, ...), but got -1.0` | Negative labels | Use 0/1 (or `StringIndexer`) |
| Accuracy ≈ 100% for no reason | **Leakage** (feature derived from label) | Remove such features |
| Confusing `probability` index | `[P(0), P(1)]` | `P(pass)` is the **second** element |
| Treating coefficients as probabilities | They are log-odds contributions | Use `e^β` (odds ratios) or sigmoid |
| Extremely large coefficients | Perfect separation / multicollinearity | Add `regParam`, remove features |
| Using accuracy on imbalanced data | Misleading | Use AUC, precision, recall, F1 |
| Wrong class gets predicted often | Threshold unsuitable | Tune `threshold` |
| Linear regression used for classes | Wrong tool | Use `LogisticRegression` |
| `family="binomial"` with 3 labels | Mismatch | Use `multinomial` (or `auto`) |
| Unscaled features, slow convergence | Different ranges | `StandardScaler` / Pipeline |

---

### 21. Phase 6 — practical scenarios

1. Compute **σ(z)** for z = −2, 0, 2 by hand.
2. Convert **p = 0.8 to odds and log-odds**.
3. For `z = 1.2823·h − 5.7703`, find the **decision boundary** in hours.
4. Compute the **starting log-loss** (p = 0.5) and the **first gradient** for the 8-student example by hand.
5. Run the **from-scratch gradient descent** and confirm w ≈ 1.2823.
6. Fit the **tiny example in Spark**; read the `probability` column.
7. Train on the **200 students** (label `passed`); print AUC and accuracy.
8. Print the **confusion counts** with `groupBy("label","prediction")`.
9. **Verify one probability** by hand with the sigmoid.
10. Compute **odds ratios** for each coefficient and interpret hours.
11. Try thresholds **0.3, 0.5, 0.7** and describe the precision/recall effect.
12. Train **multinomial logistic regression** on the 15-flower sample; interpret the coefficient matrix.

---

### 22. Phase 6 — viva questions and answers

#### Q1. What is logistic regression?

A supervised classification algorithm that models the probability of a class using the sigmoid of a linear combination of features.

#### Q2. Why not use linear regression for classification?

It can output values outside [0, 1], does not model probabilities, and is distorted by outliers.

#### Q3. Write the sigmoid function and its range.

`σ(z) = 1 / (1 + e^−z)`, with output between 0 and 1.

#### Q4. What is the logistic regression equation?

`p = 1 / (1 + e^−(β₀ + β₁x₁ + … + βₖxₖ))`.

#### Q5. What are odds and log-odds?

Odds = p/(1−p); log-odds (logit) = ln(p/(1−p)), which logistic regression models as a linear function of the features.

#### Q6. What is the decision boundary?

The set of points where p = 0.5, i.e. where the linear score z = 0; it is linear in the features.

#### Q7. What loss function does logistic regression use?

Log-loss (binary cross-entropy): `−(1/n) Σ [y ln p + (1−y) ln(1−p)]`.

#### Q8. Why not mean squared error?

With a sigmoid it becomes non-convex; log-loss comes from maximum likelihood and is convex.

#### Q9. What is the gradient of log-loss?

`(1/n) Σ (pᵢ − yᵢ) xᵢⱼ`.

#### Q10. What is maximum likelihood estimation here?

Choosing coefficients that make the observed labels as probable as possible.

#### Q11. How is a coefficient interpreted?

It is the change in log-odds for a one-unit increase in the feature; `e^β` is the odds ratio.

#### Q12. What is the default threshold and what happens if you raise it?

0.5; raising it makes the model predict the positive class less often, increasing precision and lowering recall.

#### Q13. What do `rawPrediction`, `probability` and `prediction` contain?

The linear scores, the class probabilities `[P(0), P(1)]`, and the chosen class.

#### Q14. What is softmax and when is it used?

A function converting class scores into probabilities `e^zₖ / Σ e^zⱼ`; used in multinomial logistic regression for more than two classes.

#### Q15. Which parameters control regularization in Spark?

`regParam` (strength) and `elasticNetParam` (0 = L2, 1 = L1).

#### Q16. Which algorithm does Spark use to train it?

L-BFGS (and OWL-QN for L1 penalties), iterative quasi-Newton optimizers.

#### Q17. Is logistic regression a regression or classification algorithm?

A classification algorithm, despite its name.

#### Q18. When does logistic regression struggle?

When classes need a non-linear boundary, or when features are highly correlated or perfectly separate the classes without regularization.

#### Q19. Which metrics suit logistic regression on imbalanced data?

AUC, precision, recall and F1 rather than accuracy.

#### Q20. In our tiny example, what did the model say about 6 study hours?

z = 1.9234, probability of passing 0.8725, so it predicts pass.

---

### 23. Phase 6 — final master recap

```text
LOGISTIC REGRESSION                predicts a CLASS via a PROBABILITY
  z = β₀ + β₁x₁ + … + βₖxₖ         p = σ(z) = 1/(1+e^−z)           predict 1 if p ≥ threshold (0.5)
  logit: ln(p/(1−p)) = z            odds ratio = e^β
  boundary: z = 0                   (linear)
LOSS      log-loss = −(1/n) Σ [y ln p + (1−y) ln(1−p)]
GRADIENT  (1/n) Σ (p − y)·x         update: β ← β − η·gradient
TINY EX   w = 1.2823 , b = −5.7703 ; boundary 4.5 h ; P(6h) = 0.8725 ; odds ratio 3.605
STUDENTS  test accuracy 0.9143 ; AUC 0.988 ; hours odds ratio 64.9
MULTICLASS softmax: P(k) = e^zk / Σ e^zj      (family="multinomial")
PYSPARK   LogisticRegression(maxIter, regParam, elasticNetParam, threshold).fit(train)
```

**Seven facts to memorise**

1. It predicts **probabilities**, then classes by a threshold.
2. Sigmoid squeezes any number into (0, 1); `σ(0) = 0.5`.
3. Log-odds are **linear** in the features.
4. `e^β` is the **odds ratio**.
5. Log-loss punishes **confident wrong** predictions heavily.
6. Gradient = `(1/n) Σ (p − y)x` (same shape as linear regression).
7. `probability[1]` is the chance of the positive class; moving the threshold trades precision for recall.

### Phase 6 completed

---

> **Next → Phase 7: Decision Tree** (the intuition, entropy and Gini impurity with formulas, information gain worked by hand on the classic weather dataset, how Spark builds trees, `maxDepth` and overfitting, feature importance, reading `toDebugString`, regression trees, random forest preview). File: `7_Unit6_Decision_Tree.md`
