# Unit VI — Phase 9: Support Vector Machine (SVM)

> **Continuing from Phase 8.** Back to supervised classification. Logistic regression finds *a* line that separates two classes. The **Support Vector Machine** asks a sharper question: among all separating lines, which one is **best**? Its answer: the one with the **widest margin** between the classes. We build the geometry, derive the **margin = 2/‖w‖**, understand **hinge loss** and the **soft margin**, and then run Spark's `LinearSVC`, checking the margin and the support vectors against our own arithmetic.
>
> All numbers are real outputs. The tiny example's weights, margin and support vectors were computed by Spark and verified by hand.

---

## Phase 9 — Roadmap

| # | Topic |
| --- | --- |
| 1 | The idea: the widest road between two classes |
| 2 | The hyperplane |
| 3 | Distance from a point to the hyperplane |
| 4 | The margin and support vectors |
| 5 | The hard-margin optimization problem |
| 6 | The soft margin: slack and C |
| 7 | Hinge loss |
| 8 | Hinge loss vs log-loss vs 0-1 loss |
| 9 | Non-linear data: the kernel idea |
| 10 | Spark's `LinearSVC`: what it is and is not |
| 11 | PySpark code: the tiny example with hand verification |
| 12 | Finding the support vectors |
| 13 | PySpark code: pass/fail on our students |
| 14 | The effect of `regParam` |
| 15 | Scaling |
| 16 | Multiclass SVM with One-vs-Rest |
| 17 | SVM vs logistic regression |
| 18 | Important parameters |
| 19 | Strengths and weaknesses |
| 20 | Common errors |
| 21 | Practical scenarios |
| 22 | Viva questions |
| 23 | Master revision |

---

### 1. The idea: the widest road between two classes

Two classes can usually be separated by **many** different lines:

```text
 x₂                                  x₂
  │  ○ ○     ╱  ╱ ╱  ● ●               │  ○ ○     │          ● ●
  │ ○  ○   ╱  ╱ ╱  ●  ●               │ ○  ○ ┌───┼───┐    ●  ●
  │  ○   ╱ ╱ ╱   ●                    │  ○   │MARGIN │  ●
  └──────────────── x₁                 └──────┴───────┴──────── x₁
   Many separating lines.               The SVM line: in the middle of the WIDEST road.
   Which one is best?
```

> **SVM picks the separating line (hyperplane) that maximizes the margin: the distance to the nearest points of each class.**

**Why a wide margin?** A line that hugs one class is fragile: new points near the boundary get misclassified. A line in the middle of the widest gap is the most robust to small changes, so it **generalizes better**.

| Term | Meaning |
| --- | --- |
| **Hyperplane** | The decision boundary: a line in 2D, a plane in 3D, a flat surface in higher dimensions |
| **Margin** | The width of the empty "road" around the hyperplane |
| **Support vectors** | The points **closest** to the boundary; they alone decide where it lies |

---

### 2. The hyperplane

A hyperplane is the set of points `x` that satisfy

```text
 w · x + b = 0          (w₁x₁ + w₂x₂ + … + wₙxₙ + b = 0)
```

| Symbol | Meaning |
| --- | --- |
| `w` | **Weight vector**: perpendicular (normal) to the hyperplane, giving its direction |
| `b` | **Bias** (intercept): shifts the hyperplane |

The **decision value** of a point is `f(x) = w · x + b`.

```text
 f(x) > 0   →  one side of the plane   →  predict class +1
 f(x) < 0   →  other side              →  predict class −1
 f(x) = 0   →  exactly on the plane
```

**Prediction rule:** `class = sign(w · x + b)`. (This is the same *linear* form as logistic regression's `z`; SVM and logistic regression differ in **how they choose w and b**.)

---

### 3. Distance from a point to the hyperplane

The perpendicular distance from point `x` to the hyperplane is

```text
            | w · x + b |
 d(x)  =  ───────────────
               ‖w‖

 ‖w‖ = √(w₁² + w₂² + … + wₙ²)        (the length of w)
```

**Example.** Hyperplane `3x₁ + 4x₂ − 12 = 0`, so `w = (3, 4)`, `b = −12`, `‖w‖ = 5`. Point `(4, 3)`:

```text
 f = 3×4 + 4×3 − 12 = 12      d = |12| / 5 = 2.4
```

The point is 2.4 units from the plane, on the positive side.

---

### 4. The margin and support vectors

Choose the scale of `w` and `b` so that the **closest points** on each side satisfy exactly:

```text
 w · x + b = +1     (closest positive points)
 w · x + b = −1     (closest negative points)
```

```text
              w·x + b = +1    w·x + b = 0    w·x + b = −1
 x₂                │               │               │
  │  ● ●           │               │               │       ○ ○
  │ ●  ●  [●]──────┤               │               ├──────[○]  ○
  │         ◄──── margin = 2/‖w‖ ────►              │
  │                │               │               │
  └──────────────────────────────────────────────────────────── x₁
        [ ] = support vectors (lie exactly on the margin lines)
```

**Width of the margin.** The distance from the middle plane (`f = 0`) to either margin line (`f = ±1`) is `1/‖w‖`. So the total width is

```text
 margin  =  2 / ‖w‖
```

> **Key insight: a smaller ‖w‖ means a wider margin.** Maximizing the margin is the same as minimizing ‖w‖.

**Support vectors** are the points exactly on the margin lines. They are the **only** points that matter: delete any other point and the hyperplane does not move. That is where the name comes from.

---

### 5. The hard-margin optimization problem

If the classes are perfectly separable, SVM solves:

```text
 minimize     ½ ‖w‖²
 subject to   yᵢ (w · xᵢ + b) ≥ 1      for every training point i
```

with labels coded as `yᵢ ∈ {+1, −1}`.

| Part | Meaning |
| --- | --- |
| `½‖w‖²` | Minimizing it maximizes the margin 2/‖w‖ (the ½ is for convenience; squared makes it smooth) |
| `yᵢ(w·xᵢ + b) ≥ 1` | Every point is on the **correct side**, and **outside** the margin |

For a correctly classified point, `yᵢ f(xᵢ)` is positive: the larger it is, the more confidently the point is classified.

**The problem:** real data is rarely perfectly separable (noise, overlaps). A strict hard margin may not exist, or may be forced into a very narrow, overfit margin by a single outlier. So we relax it.

---

### 6. The soft margin: slack and C

Allow some points to **violate the margin** (even be misclassified) at a **cost**. For each point introduce a **slack variable** `ξᵢ ≥ 0`:

```text
 minimize     ½ ‖w‖²  +  C Σ ξᵢ
 subject to   yᵢ (w · xᵢ + b) ≥ 1 − ξᵢ ,    ξᵢ ≥ 0
```

| `ξᵢ` | Meaning |
| --- | --- |
| 0 | Point is outside the margin, correctly classified |
| between 0 and 1 | Inside the margin but still on the correct side |
| greater than 1 | **Misclassified** |

**The parameter C controls the trade-off**

| | Small C | Large C |
| --- | --- | --- |
| Penalty for violations | Low | High |
| Margin | **Wide** (tolerates some errors) | **Narrow** (tries to classify every point) |
| Risk | Underfitting (too relaxed) | Overfitting (chases noise/outliers) |
| Behaviour | Smoother, more general | Tighter, more complex |

> **Viva line:** *C balances a wide margin against training errors. Large C = strict, small margin, risk of overfitting. Small C = tolerant, wide margin, risk of underfitting.*

---

### 7. Hinge loss

The soft-margin problem can be written as minimizing a **loss function** called the **hinge loss**:

```text
 hinge(y, f) = max( 0 ,  1 − y · f(x) )
```

Let `m = y · f(x)` (the "margin" of that one point).

| m = y·f(x) | Meaning | Hinge loss |
| --- | --- | --- |
| **≥ 1** | Correct and outside the margin | **0** (no penalty) |
| 0.5 | Correct but inside the margin | 0.5 |
| 0 | On the boundary | 1 |
| −1 | Wrong side | 2 |
| −2 | Badly wrong | 3 |

```text
 loss
  3 │╲
  2 │ ╲
  1 │  ╲
    │   ╲
  0 │────╲_________________  ← zero once m ≥ 1
    └───────────────────────── m = y·f(x)
      -2  -1   0   1   2
```

**Spark's objective** (what `LinearSVC` minimizes):

```text
 (1/n) Σ max(0, 1 − yᵢ(w·xᵢ + b))   +   regParam × ½ ‖w‖²
```

| Spark term | Role |
| --- | --- |
| Average hinge loss | Penalizes margin violations |
| `regParam × ½‖w‖²` | L2 penalty: pushes ‖w‖ down (wider margin) |

**Relation to C:** the textbook C multiplies the violations; Spark's `regParam` multiplies the penalty on ‖w‖. They play opposite roles: **large `regParam` ≈ small C** (stronger regularization, wider margin, simpler model); **small `regParam` ≈ large C** (fit the training data tightly).

---

### 8. Hinge loss vs log-loss vs 0-1 loss

All three give a "cost" for a point with margin `m = y·f(x)`.

| m | 0-1 loss (error) | **Hinge** `max(0, 1−m)` | Log-loss `ln(1 + e^−m)` |
| --- | --- | --- | --- |
| 2 | 0 | **0** | 0.127 |
| 1 | 0 | **0** | 0.313 |
| 0 | 1 | **1** | 0.693 |
| −1 | 1 | **2** | 1.313 |

| | Hinge (SVM) | Log-loss (logistic regression) |
| --- | --- | --- |
| Confident correct points | **Zero** loss | Always small positive loss |
| Which points shape the model? | Only **support vectors** and violators | **All** points |
| Output | Decision value (a score) | A **probability** |
| Smoothness | Has a kink at m = 1 | Smooth everywhere |

---

### 9. Non-linear data: the kernel idea

Some data cannot be separated by a straight line:

```text
 x₂
  │    ● ● ● ●
  │  ●  ○ ○ ○  ●        a ring of ● around ○: no straight line separates them
  │  ●  ○ ○ ○  ●
  │    ● ● ● ●
  └────────────── x₁
```

**The kernel trick:** map the data into a **higher-dimensional space** where a straight hyperplane *does* separate it, without ever computing the new coordinates explicitly. The solution in the dual form depends only on dot products `xᵢ · xⱼ`, which a **kernel function** `K(xᵢ, xⱼ)` replaces.

| Kernel | Formula | Use |
| --- | --- | --- |
| Linear | `K = x · y` | Linearly separable data (**what Spark offers**) |
| Polynomial | `K = (x · y + c)^d` | Curved boundaries |
| **RBF (Gaussian)** | `K = exp(−γ‖x − y‖²)` | Very flexible; the default choice elsewhere |

> **Important Spark fact:** Spark MLlib's SVM, **`LinearSVC`, supports only the *linear* kernel and *binary* classification.** There is no RBF kernel in `pyspark.ml`. For non-linear problems in Spark, add non-linear features (for example `PolynomialExpansion`), or use trees / random forests / MLP. (scikit-learn's `SVC` offers kernels, but it is not distributed.)

---

### 10. Spark's `LinearSVC`: what it is and is not

| Property | `LinearSVC` |
| --- | --- |
| Problem | **Binary** classification only |
| Kernel | **Linear** only |
| Loss | **Hinge loss** with **L2** regularization |
| Optimizer | OWLQN (an L-BFGS variant) |
| Output columns | `rawPrediction` (`[−f, +f]`) and `prediction`; **no probability column** |
| Prediction rule | `prediction = 1` if `rawPrediction[1] > threshold` (default **0.0**) |
| Multiclass | Wrap with **`OneVsRest`** (Section 16) |
| Labels | 0.0 and 1.0 (Spark converts to ±1 internally) |

---

### 11. PySpark code: the tiny example with hand verification

Eight points in 2D, linearly separable:

| Class 1 (positive) | Class 0 (negative) |
| --- | --- |
| (3, 3) | (1, 1) |
| (4, 4) | (2, 1) |
| (5, 3) | (1, 2) |
| (4, 2.5) | (1.5, 1.5) |

```python
from pyspark.ml.feature import VectorAssembler
from pyspark.ml.classification import LinearSVC
import numpy as np

pts = [(3.0, 3.0, 1), (4.0, 4.0, 1), (5.0, 3.0, 1), (4.0, 2.5, 1),
       (1.0, 1.0, 0), (2.0, 1.0, 0), (1.0, 2.0, 0), (1.5, 1.5, 0)]

df = spark.createDataFrame(pts, ["x1", "x2", "label"])
fv = VectorAssembler(inputCols=["x1", "x2"], outputCol="features").transform(df)

model = LinearSVC(maxIter=200, regParam=0.01, featuresCol="features", labelCol="label").fit(fv)

w = model.coefficients.toArray()
b = model.intercept
print("w =", w.round(4), " b =", round(b, 4))
print("‖w‖ =", round(float(np.linalg.norm(w)), 4), "  margin 2/‖w‖ =", round(2 / float(np.linalg.norm(w)), 4))
```

```text
w = [0.62  0.772]   b = -3.1784
‖w‖ = 0.9901   margin 2/‖w‖ = 2.0199
```

**The learned hyperplane**

```text
 0.62·x₁ + 0.772·x₂ − 3.1784 = 0
```

**Hand check of ‖w‖ and the margin**

```text
 ‖w‖ = √(0.62² + 0.772²) = √(0.3844 + 0.5960) = √0.9804 = 0.9901
 margin = 2 / 0.9901 = 2.0199          ✔
```

**Decision values and predictions**

```python
for r in model.transform(fv).collect():
    print(r.features.toArray().tolist(), "label", r.label, "rawPrediction", r.rawPrediction.toArray().round(4), "pred", r.prediction)
```

```text
[3.0, 3.0] label 1  rawPrediction [-0.9976  0.9976]  pred 1.0
[4.0, 4.0] label 1  rawPrediction [-2.3896  2.3896]  pred 1.0
[5.0, 3.0] label 1  rawPrediction [-2.2376  2.2376]  pred 1.0
[4.0, 2.5] label 1  rawPrediction [-1.2316  1.2316]  pred 1.0
[1.0, 1.0] label 0  rawPrediction [ 1.7864 -1.7864]  pred 0.0
[2.0, 1.0] label 0  rawPrediction [ 1.1664 -1.1664]  pred 0.0
[1.0, 2.0] label 0  rawPrediction [ 1.0144 -1.0144]  pred 0.0
[1.5, 1.5] label 0  rawPrediction [ 1.0904 -1.0904]  pred 0.0
```

**Reading `rawPrediction` = `[−f(x), +f(x)]`**, where `f(x) = w·x + b` is the decision value.

**Hand check for (3, 3):**

```text
 f(3, 3) = 0.62×3 + 0.772×3 − 3.1784 = 1.86 + 2.316 − 3.1784 = 0.9976   ✔  (class +1)
```

**Hand check for (1, 1):**

```text
 f(1, 1) = 0.62 + 0.772 − 3.1784 = −1.7864   ✔  (class −1, so predicted 0)
```

Training accuracy is **100%**: all 8 points are on the correct side.

---

### 12. Finding the support vectors

Support vectors have **|f(x)| ≈ 1** (they sit on the margin lines). Compute `y·f(x)` with labels coded as ±1:

| Point | Class | f(x) | y·f(x) | On the margin? |
| --- | --- | --- | --- | --- |
| **(3, 3)** | +1 | +0.9976 | **0.9976** | ✅ support vector |
| **(1, 2)** | −1 | −1.0144 | **1.0144** | ✅ support vector |
| (1.5, 1.5) | −1 | −1.0904 | 1.0904 | near the margin |
| (2, 1) | −1 | −1.1664 | 1.1664 | near the margin |
| (4, 2.5) | +1 | +1.2316 | 1.2316 | near the margin |
| (1, 1) | −1 | −1.7864 | 1.7864 | far outside (irrelevant) |
| (5, 3) | +1 | +2.2376 | 2.2376 | far outside (irrelevant) |
| (4, 4) | +1 | +2.3896 | 2.3896 | far outside (irrelevant) |

```text
 x₂
  4 │                 ● (4,4)
    │
  3 │         [●](3,3)       ● (5,3)
    │    ╲        ╲   ● (4,2.5)       ← positive side: f ≥ +1
    │      ╲  ← the hyperplane f = 0
  2 │ [○](1,2)  ╲    
    │  ○ (1.5,1.5) ╲ ○ (2,1)           ← negative side: f ≤ −1
  1 │ ○ (1,1)
    └─────────────────────────── x₁
 [ ] = support vectors: (3,3) positive and (1,2) negative
```

**Observations**

1. **(3, 3)** and **(1, 2)** are the closest points to the boundary: the support vectors. Points (1, 1), (5, 3), (4, 4) are far away and **would not change the hyperplane if removed**.
2. The values for the support vectors are almost exactly ±1, as the theory says. Not *exactly* 1 because Spark solves the **soft-margin** problem with a small `regParam`, not the strict hard-margin one.
3. Margin width ≈ **2.02**, the width of the empty road between the two classes.

---

### 13. PySpark code: pass/fail on our students

Same task and split as Logistic Regression (Phase 6): predict `passed` from `hours`, `attendance`, `prev_marks`.

```python
import sys
sys.path.append(r"C:\sparklabs")
from make_data import make_students
from pyspark.ml.classification import LinearSVC
from pyspark.ml.evaluation import MulticlassClassificationEvaluator, BinaryClassificationEvaluator
from pyspark.sql import functions as F

df = spark.createDataFrame(make_students(),
        ["id", "hours", "attendance", "prev_marks", "final_marks", "passed"])
va = VectorAssembler(inputCols=["hours", "attendance", "prev_marks"], outputCol="features")
data = va.transform(df).select("features", F.col("passed").alias("label"))
train, test = data.randomSplit([0.8, 0.2], seed=42)

svm = LinearSVC(maxIter=100, regParam=0.01)
model = svm.fit(train)
pred = model.transform(test)

print("coefficients:", model.coefficients)
print("intercept   :", model.intercept)

acc = MulticlassClassificationEvaluator(metricName="accuracy").evaluate(pred)
auc = BinaryClassificationEvaluator(metricName="areaUnderROC").evaluate(pred)
print("accuracy:", round(acc, 4), " AUC:", round(auc, 4))
```

```text
coefficients: [0.9714093520904447,0.05799892711683513,0.07988237930801814]
intercept   : -14.563317309371035
accuracy: 0.9143  AUC: 0.984
```

**Reading it**

- The hyperplane: `0.971·hours + 0.058·attendance + 0.080·prev_marks − 14.563 = 0`.
- All three weights are positive: more hours, attendance and previous marks push towards "pass".
- **Hours** has by far the largest weight, again agreeing with the correlation heat map.
- **Accuracy 91.4%** on the test set, **AUC 0.984**.

**Comparison on the same split**

| Model | Test accuracy |
| --- | --- |
| Logistic Regression (Phase 6) | 0.9143 |
| **Linear SVM** | **0.9143** |
| Decision Tree, depth 4 (Phase 7) | 0.8857 |

For nearly-linear data, SVM and logistic regression perform alike. The difference is **how** they get there (hinge loss vs log-loss) and that SVM gives scores rather than probabilities.

---

### 14. The effect of `regParam`

`regParam` is our "inverse of C": it controls the margin/error trade-off.

```python
for rp in (0.01, 0.1, 1.0, 10.0):
    m = LinearSVC(maxIter=100, regParam=rp).fit(train)
    # train/test accuracy and ‖w‖ for each
```

```text
regParam   train acc   test acc    ‖w‖
  0.01       0.9515     0.9143    0.9764
  0.1        0.9515     0.9429    0.4844
  1.0        0.9455     0.8857    0.2077
 10.0        0.5455     0.2857    0.0228
```

**Reading the table**

| regParam | What happens |
| --- | --- |
| 0.01 | Low regularization: ‖w‖ large (0.98), narrow margin |
| **0.1** | **Sweet spot**: best test accuracy 0.9429, ‖w‖ shrinks to 0.48 (wider margin) |
| 1.0 | Over-regularized: ‖w‖ 0.21, accuracy drops |
| 10.0 | **Collapse**: ‖w‖ ≈ 0.02, the model is nearly flat and predicts **everyone as "pass"**. Test accuracy 0.2857 = 10/35 (exactly the share of real passes) and train accuracy 0.5455 = 90/165 |

This shows the **bias-variance trade-off** in a single parameter: ‖w‖ falls as regularization rises (a wider margin = a simpler model), and too much regularization causes **underfitting**. In practice choose `regParam` with **cross-validation** (Phase 11).

---

### 15. Scaling

SVM is sensitive to feature scales: the margin is a geometric (distance) concept. Spark's `standardization=True` (default) standardizes internally during optimization, and returns coefficients in the original units.

For clarity and for combining with other steps, you can also scale explicitly in a **Pipeline**:

```python
from pyspark.ml import Pipeline
from pyspark.ml.feature import StandardScaler

pipe = Pipeline(stages=[
    StandardScaler(inputCol="features", outputCol="scaled", withMean=True, withStd=True),
    LinearSVC(featuresCol="scaled", maxIter=100, regParam=0.01)
])
pm = pipe.fit(train)
print(MulticlassClassificationEvaluator(metricName="accuracy").evaluate(pm.transform(test)))
```

```text
0.9143
```

Same accuracy here, as expected, since Spark was already standardizing internally. Remember the rule: the scaler is **fitted on the training data only** (a Pipeline does this automatically).

---

### 16. Multiclass SVM with One-vs-Rest

`LinearSVC` is binary. For **K classes**, `OneVsRest` trains **K binary classifiers**, each separating *one class from all the others*, and predicts the class whose classifier gives the **highest score**.

```text
 Classifier 0: class 0 vs {1, 2}
 Classifier 1: class 1 vs {0, 2}      → predict the class with the highest decision value
 Classifier 2: class 2 vs {0, 1}
```

```python
from pyspark.ml.classification import OneVsRest

ovr = OneVsRest(classifier=LinearSVC(maxIter=100, regParam=0.01),
                labelCol="label", featuresCol="features")
ovr_model = ovr.fit(flower_df)          # flower_df = the 15-row Iris sample (Phase 6)

pred = ovr_model.transform(flower_df)
print("accuracy:", MulticlassClassificationEvaluator(metricName="accuracy").evaluate(pred))
print("models  :", len(ovr_model.models))
print([int(r[0]) for r in pred.select("prediction").collect()])
```

```text
accuracy: 1.0
models  : 3
[0, 0, 0, 0, 0, 1, 1, 1, 1, 1, 2, 2, 2, 2, 2]
```

Three binary SVMs were trained (one per species) and together classify all 15 flowers correctly. (Training accuracy on 15 well-separated rows; do not read it as general performance.)

---

### 17. SVM vs logistic regression

| | **Linear SVM** | **Logistic Regression** |
| --- | --- | --- |
| Goal | Maximize the **margin** | Maximize the **likelihood** |
| Loss | Hinge | Log-loss |
| Output | Decision score; **no probabilities** | **Probabilities** |
| Points that matter | Support vectors | All points |
| Boundary | Linear (kernels in other libraries) | Linear |
| Interpretation | Weights, margin | Odds ratios |
| Regularization knob | `regParam` (≈ 1/C) | `regParam`, `elasticNetParam` |
| Multiclass in Spark | `OneVsRest` | Native softmax |
| When to prefer | Clear margin, high dimensions, outlier-light data | You need probabilities / interpretability |

---

### 18. Important parameters

| Parameter | Default | Meaning |
| --- | --- | --- |
| `maxIter` | 100 | Maximum iterations |
| `regParam` | 0.0 | L2 regularization (higher = wider margin, simpler) |
| `tol` | 1e-6 | Convergence tolerance |
| `fitIntercept` | True | Learn the bias b |
| `standardization` | True | Standardize features internally |
| `threshold` | 0.0 | Cut-off applied to the raw score |
| `weightCol` | none | Row weights (helps with class imbalance) |
| `aggregationDepth` | 2 | Tree-aggregation depth (for large clusters) |
| `featuresCol`, `labelCol`, `predictionCol`, `rawPredictionCol` | standard | Column names |

---

### 19. Strengths and weaknesses

| Strengths | Weaknesses |
| --- | --- |
| Maximum-margin idea gives **good generalization** | **Binary only** in Spark (needs `OneVsRest`) |
| Effective in **high-dimensional** spaces | **Linear** kernel only in Spark |
| Only support vectors matter (robust to far-away points) | **No probabilities** |
| Regularization is built in | Sensitive to **feature scaling** |
| Works well with clear margin | Hard to interpret vs trees; hyperparameter tuning needed |
| | Slower to tune on very noisy overlapping data |

---

### 20. Common errors

| Symptom | Cause | Fix |
| --- | --- | --- |
| Error with a 3-class label | `LinearSVC` is binary | Use `OneVsRest(classifier=LinearSVC(...))` |
| No `probability` column | SVM gives scores | Use `rawPrediction`, or Logistic Regression |
| Accuracy near chance with huge `regParam` | Over-regularization | Lower `regParam`; tune by CV |
| Wanting an RBF kernel | Not available in Spark MLlib | Add polynomial features, use trees/MLP, or sklearn on small data |
| Labels −1/+1 | Spark expects 0/1 | Convert labels to 0.0 and 1.0 |
| Poor results with raw features | Different scales | Keep `standardization=True` or scale in a Pipeline |
| Misreading `rawPrediction` | `[−f, +f]` | Index 1 is the score of class 1 |
| Perfect accuracy with `final_marks` as a feature | Leakage | Remove label-derived features |
| Imbalanced classes ignored | Majority class dominates | `weightCol`, or evaluate with F1/AUC |
| Different results each run | No seed in the split | `randomSplit(..., seed=42)` |

---

### 21. Phase 9 — practical scenarios

1. Compute the **distance from (4, 3)** to the hyperplane `3x₁ + 4x₂ − 12 = 0`.
2. For `w = (0.62, 0.772)`, compute **‖w‖ and the margin 2/‖w‖**.
3. Compute the **decision value f(x)** of (3, 3) and (1, 1) by hand.
4. Fit `LinearSVC` on the 8-point data; compare your numbers with `model.coefficients` and `intercept`.
5. List the **support vectors** by finding the points with |f(x)| closest to 1.
6. Compute **hinge loss** for margins m = 2, 0.5, 0 and −1.
7. Train an SVM on the **200 students**; print accuracy and AUC.
8. Sweep **`regParam` = 0.01, 0.1, 1, 10**; explain the pattern using ‖w‖.
9. Build a **Pipeline** with `StandardScaler` and `LinearSVC`.
10. Use **`OneVsRest`** to classify the 3-species flower sample.
11. Compare **SVM and logistic regression** accuracy on the same split.
12. Explain why Spark's SVM **cannot** separate a ring-shaped class pattern, and name two alternatives.

---

### 22. Phase 9 — viva questions and answers

#### Q1. What is a Support Vector Machine?

A supervised classifier that finds the hyperplane separating classes with the maximum margin.

#### Q2. What is a hyperplane?

The decision boundary `w · x + b = 0`: a line in 2D, a plane in 3D, a flat surface in higher dimensions.

#### Q3. What is the margin?

The width of the empty region between the classes around the hyperplane; for support vectors on `f = ±1` it equals `2/‖w‖`.

#### Q4. What are support vectors?

The training points closest to the hyperplane, lying on the margin lines; they alone determine the hyperplane.

#### Q5. Why do we want a large margin?

A wider margin is more robust to small changes and noise, so the model generalizes better.

#### Q6. Write the hard-margin optimization problem.

Minimize `½‖w‖²` subject to `yᵢ(w·xᵢ + b) ≥ 1` for all training points.

#### Q7. What is the soft margin and why is it needed?

It allows some points to violate the margin via slack variables `ξᵢ`, with objective `½‖w‖² + CΣξᵢ`, because real data is rarely perfectly separable.

#### Q8. What does the parameter C control?

The trade-off between margin width and training errors: large C = narrow margin and strict fit (overfitting risk); small C = wide margin and tolerance (underfitting risk).

#### Q9. Define hinge loss.

`max(0, 1 − y·f(x))`: zero for points correctly classified outside the margin, growing linearly for violations.

#### Q10. How does Spark's `regParam` relate to C?

They act inversely: a large `regParam` behaves like a small C (stronger regularization, wider margin).

#### Q11. What is the kernel trick?

Using a kernel function to compute dot products in a higher-dimensional space implicitly, so non-linear boundaries can be learned with a linear method.

#### Q12. Which kernels does Spark's SVM support?

Only the linear kernel (`LinearSVC`).

#### Q13. Can `LinearSVC` handle more than two classes?

Not directly; use `OneVsRest`.

#### Q14. What is the prediction rule of a linear SVM?

Predict class +1 if `w·x + b > 0` (threshold 0), otherwise class −1.

#### Q15. Does `LinearSVC` output probabilities?

No, only `rawPrediction` scores and the predicted class.

#### Q16. What is the difference between hinge loss and log-loss?

Hinge loss is zero for confident correct points and only support vectors/violators affect the model; log-loss is always positive and all points contribute, giving probabilities.

#### Q17. How does SVM differ from logistic regression?

SVM maximizes the margin using hinge loss and outputs scores; logistic regression maximizes likelihood using log-loss and outputs probabilities.

#### Q18. Why should features be scaled for SVM?

The margin is a distance, so features with larger ranges would dominate; scaling puts them on equal footing.

#### Q19. What happened when we set `regParam=10` in our experiment?

‖w‖ collapsed to about 0.02 and the model nearly stopped discriminating, giving chance-level accuracy (underfitting).

#### Q20. In the tiny example, what were the support vectors and the margin?

Support vectors (3, 3) and (1, 2) (|f| ≈ 1), and the margin 2/‖w‖ = 2.02 with w = (0.62, 0.772).

---

### 23. Phase 9 — final master recap

```text
SVM                  maximum-margin classifier
HYPERPLANE           w·x + b = 0 ;  predict sign(w·x + b)
DISTANCE             d = |w·x + b| / ‖w‖
MARGIN               support vectors on w·x+b = ±1 ;  margin = 2/‖w‖   (small ‖w‖ = wide margin)
HARD MARGIN          min ½‖w‖²  s.t.  y(w·x+b) ≥ 1
SOFT MARGIN          min ½‖w‖² + C Σξ         (C large = strict/overfit ; C small = tolerant/underfit)
HINGE LOSS           max(0, 1 − y·f(x))       (Spark: mean hinge + regParam·½‖w‖²)
KERNELS              linear / polynomial / RBF   → Spark LinearSVC = LINEAR + BINARY only
TINY EXAMPLE         w=(0.62,0.772), b=−3.1784, ‖w‖=0.9901, margin 2.0199, support vectors (3,3),(1,2)
STUDENTS             accuracy 0.9143, AUC 0.984 ; regParam 0.1 best (0.9429) ; 10.0 collapses
MULTICLASS           OneVsRest(classifier=LinearSVC(...))
PYSPARK              LinearSVC(maxIter, regParam).fit(train) → rawPrediction = [−f, +f]
```

**Seven facts to memorise**

1. SVM chooses the hyperplane with the **maximum margin**.
2. Margin = **2/‖w‖**; support vectors lie on `f = ±1`.
3. Only **support vectors** determine the boundary.
4. **Hinge loss** = `max(0, 1 − y·f)`; zero beyond the margin.
5. **C large** = narrow margin, overfit risk; **regParam large** (Spark) = wide margin, underfit risk.
6. Spark's `LinearSVC` is **linear and binary** (use `OneVsRest` for more classes), with **no probabilities**.
7. Scale your features.

### Phase 9 completed

---

> **Next → Phase 10: Naive Bayes** (Bayes' theorem, the "naive" independence assumption, a complete hand calculation on the weather data reproduced by Spark, Laplace smoothing, Multinomial/Bernoulli/Gaussian variants, text classification with CountVectorizer, PySpark code). File: `10_Unit6_Naive_Bayes.md`
