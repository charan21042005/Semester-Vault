# Unit VI — Phase 5: Linear Regression

> **Continuing from Phase 4.** We start the algorithms. **Linear regression** is the foundation of machine learning: it predicts a **number** by fitting a straight line (or a flat surface). We build it from scratch: the idea, the loss function, the least-squares mathematics (derived and worked by hand), the normal equation, **gradient descent coded from scratch**, regularization, and finally the PySpark code with a full reading of the output.
>
> Every number here was computed for real. The hand calculations are checked against both numpy and Spark, and they agree.

---

## Phase 5 — Roadmap

| # | Topic |
| --- | --- |
| 1 | The idea of linear regression |
| 2 | The model: simple and multiple |
| 3 | Residuals and the loss function |
| 4 | Least squares for simple regression (derivation) |
| 5 | A complete worked example by hand |
| 6 | The normal equation (matrix form) |
| 7 | Gradient descent: the iterative way |
| 8 | Gradient descent from scratch in Python |
| 9 | How Spark solves it |
| 10 | PySpark code: the tiny example |
| 11 | PySpark code: multiple regression on our students |
| 12 | Reading the model summary |
| 13 | Interpreting coefficients |
| 14 | Evaluating: RMSE, MAE, R² |
| 15 | Assumptions of linear regression |
| 16 | Regularization: Ridge, Lasso, Elastic Net |
| 17 | Important parameters |
| 18 | Strengths and weaknesses |
| 19 | Common errors |
| 20 | Practical scenarios |
| 21 | Viva questions |
| 22 | Master revision |

---

### 1. The idea of linear regression

> **Linear regression finds the straight line (or hyperplane) that best fits the data, and uses it to predict a continuous number.**

**Intuition.** Look at study hours and marks:

```text
 marks
  75 │                          ●
  65 │                   ●       /
  55 │            ●          /
  50 │      ●           /
  40 │ ●          /          ← the best-fit line passes as close as possible to all points
     └──────────────────────── hours
       1   2   3   4   5
```

Once we have the line, we can predict marks for any number of hours, even one we never saw (for example 3.5 hours).

**Where it is used:** predicting prices, sales, temperature, salaries, delivery times, energy use. Any "how much?" question with roughly straight-line behaviour.

---

### 2. The model: simple and multiple

**Simple linear regression** (one feature)

```text
 ŷ = β₀ + β₁ x
```

**Multiple linear regression** (several features)

```text
 ŷ = β₀ + β₁x₁ + β₂x₂ + … + βₖxₖ
```

| Symbol | Name | Meaning |
| --- | --- | --- |
| ŷ ("y-hat") | Prediction | The model's estimate of the label |
| y | Actual value | The true label |
| x₁ … xₖ | Features | Inputs |
| **β₀** | **Intercept** (bias) | The prediction when all features are 0 |
| **β₁ … βₖ** | **Coefficients** (weights) | The change in ŷ for a **one-unit increase** in that feature |

**Matrix form** (compact, used by Spark): put a column of 1s in `X` for the intercept.

```text
 ŷ = X β        X is n×(k+1),   β is (k+1)×1
```

The **parameters** the algorithm *learns* are β₀ … βₖ.

---

### 3. Residuals and the loss function

**Residual** = actual − predicted. It is the error for one point.

```text
 eᵢ = yᵢ − ŷᵢ
```

```text
 marks
  75 │                    ● actual
     │                    │  ← residual (vertical gap)
  74 │                    ◦ predicted (on the line)
```

We want the line with the **smallest total error**. But residuals can be positive and negative and would cancel. So we **square** them.

**Loss functions**

| Name | Formula | Notes |
| --- | --- | --- |
| **SSE** (sum of squared errors) | `Σ (yᵢ − ŷᵢ)²` | The quantity least squares minimizes |
| **MSE** (mean squared error) | `SSE / n` | Average squared error |
| **RMSE** | `√MSE` | In the **same units** as y |
| **MAE** | `Σ |yᵢ − ŷᵢ| / n` | Average absolute error |

**Why square the errors?**

1. Positive and negative errors cannot cancel.
2. Big mistakes are punished **more** (error 10 costs 100, not 10).
3. The squared loss is smooth, so calculus gives a clean solution.

> **The least-squares principle:** choose `β₀, β₁, …` to **minimize SSE**. That is *what "training" means* for linear regression.

---

### 4. Least squares for simple regression (derivation)

For `ŷ = β₀ + β₁x`, the loss is

```text
 L(β₀, β₁) = Σ (yᵢ − β₀ − β₁xᵢ)²
```

At the minimum, both slopes of `L` are zero. Take partial derivatives:

```text
 ∂L/∂β₀ = −2 Σ (yᵢ − β₀ − β₁xᵢ)        = 0      …(1)
 ∂L/∂β₁ = −2 Σ xᵢ (yᵢ − β₀ − β₁xᵢ)     = 0      …(2)
```

From (1): `Σyᵢ = nβ₀ + β₁Σxᵢ`, so dividing by n:

```text
 ȳ = β₀ + β₁ x̄      ⟹      β₀ = ȳ − β₁ x̄          …(3)
```

(**The best-fit line always passes through the point (x̄, ȳ).**)

Substituting (3) into (2) and simplifying gives the slope:

```text
        Σ (xᵢ − x̄)(yᵢ − ȳ)         Sxy        cov(X, Y)
 β₁  = ───────────────────── =  ─────── =  ───────────
           Σ (xᵢ − x̄)²              Sxx        var(X)
```

**The two formulas to remember**

```text
 β₁ = Sxy / Sxx            β₀ = ȳ − β₁ · x̄

 Sxy = Σ (xᵢ − x̄)(yᵢ − ȳ)       Sxx = Σ (xᵢ − x̄)²
```

Note the link with Phase 4: the slope is the **covariance divided by the variance of x** (and equals `r · σᵧ/σₓ`).

---

### 5. A complete worked example by hand

Study hours (x) and exam marks (y), the same five students as in Phase 4:

| i | x | y | x − x̄ | y − ȳ | (x−x̄)(y−ȳ) | (x−x̄)² |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | 1 | 40 | −2 | −17 | 34 | 4 |
| 2 | 2 | 50 | −1 | −7 | 7 | 1 |
| 3 | 3 | 55 | 0 | −2 | 0 | 0 |
| 4 | 4 | 65 | 1 | 8 | 8 | 1 |
| 5 | 5 | 75 | 2 | 18 | 36 | 4 |
| | x̄ = 3 | ȳ = 57 | | | **Sxy = 85** | **Sxx = 10** |

**Step 1: slope and intercept**

```text
 β₁ = Sxy / Sxx = 85 / 10 = 8.5
 β₀ = ȳ − β₁x̄  = 57 − 8.5 × 3 = 57 − 25.5 = 31.5

 Model:  marks = 31.5 + 8.5 × hours
```

**Step 2: predictions and residuals**

| x | y (actual) | ŷ = 31.5 + 8.5x | Residual y − ŷ | Residual² |
| --- | --- | --- | --- | --- |
| 1 | 40 | 40.0 | 0.0 | 0.00 |
| 2 | 50 | 48.5 | 1.5 | 2.25 |
| 3 | 55 | 57.0 | −2.0 | 4.00 |
| 4 | 65 | 65.5 | −0.5 | 0.25 |
| 5 | 75 | 74.0 | 1.0 | 1.00 |
| | | | | **SSE = 7.5** |

**Step 3: error metrics**

```text
 MSE  = SSE / n = 7.5 / 5 = 1.5
 RMSE = √1.5            = 1.2247
 MAE  = (0 + 1.5 + 2 + 0.5 + 1) / 5 = 5 / 5 = 1.0
```

**Step 4: R² (how much variation the line explains)**

```text
 SST = Σ(y − ȳ)² = 289 + 49 + 4 + 64 + 324 = 730      (total variation)
 SSE = 7.5                                              (unexplained)
 R²  = 1 − SSE/SST = 1 − 7.5/730 = 0.98973
```

**Step 5: a prediction.** For 3.5 hours: `31.5 + 8.5 × 3.5 = 61.25` marks.

**Interpretation:** each extra study hour is associated with **8.5 more marks**; a student who studies 0 hours is predicted to score 31.5 (the intercept, an extrapolation beyond our data). The line explains **98.97%** of the variation in marks.

---

### 6. The normal equation (matrix form)

For many features we solve all the coefficients **at once**. Setting the gradient of SSE to zero gives the **normal equation**:

```text
 β = (XᵀX)⁻¹ Xᵀ y
```

**Check with our example.** Put a column of 1s in front of x:

```text
     ⎡1 1⎤          ⎡40⎤
     ⎢1 2⎥          ⎢50⎥
 X = ⎢1 3⎥ ,    y = ⎢55⎥
     ⎢1 4⎥          ⎢65⎥
     ⎣1 5⎦          ⎣75⎦

 XᵀX = ⎡  5  15⎤       Xᵀy = ⎡285⎤
       ⎣ 15  55⎦              ⎣940⎦
```

```text
 det(XᵀX) = 5×55 − 15×15 = 275 − 225 = 50

 (XᵀX)⁻¹ = (1/50) ⎡ 55  −15⎤
                  ⎣−15    5⎦

 β = (1/50) ⎡ 55×285 − 15×940⎤ = (1/50) ⎡15675 − 14100⎤ = (1/50) ⎡1575⎤ = ⎡31.5⎤
            ⎣−15×285 +  5×940⎦          ⎣−4275 +  4700⎦          ⎣ 425⎦   ⎣ 8.5⎦
```

**β₀ = 31.5 and β₁ = 8.5**, exactly the same as the scalar formulas. 

| Normal equation | Gradient descent |
| --- | --- |
| Exact answer in one step | Approaches the answer step by step |
| Needs `(XᵀX)⁻¹`: expensive with many features | Scales to huge feature counts |
| No learning rate | Needs a learning rate and iterations |

---

### 7. Gradient descent: the iterative way

When the normal equation is too expensive, we find the minimum by **walking downhill** on the loss surface.

```text
 Loss
  │ ╲
  │  ╲           start here
  │   ╲_  ←step
  │     ╲__←step
  │        ╲____ ← minimum (best β)
  └───────────────── β
```

**The idea:** at the current β, compute the **gradient** (the slope of the loss). Move β a small step **in the opposite direction** of the gradient. Repeat until it stops improving.

**Loss:** `MSE = (1/n) Σ (yᵢ − (β₀ + β₁xᵢ))²`

**Gradients (partial derivatives)**

```text
 ∂MSE/∂β₁ = (2/n) Σ (ŷᵢ − yᵢ) xᵢ
 ∂MSE/∂β₀ = (2/n) Σ (ŷᵢ − yᵢ)
```

**Update rule** (repeat many times)

```text
 β₁ ← β₁ − η · ∂MSE/∂β₁
 β₀ ← β₀ − η · ∂MSE/∂β₀
```

`η` (eta) is the **learning rate**: the step size.

| Learning rate | Effect |
| --- | --- |
| Too small | Very slow progress |
| Just right | Smooth descent to the minimum |
| Too large | Overshoots; may bounce around or **diverge** |

**One step by hand.** Start with `β₁ = 0, β₀ = 0` (so every ŷ = 0) and `η = 0.02`:

```text
 ∂/∂β₁ = (2/5) Σ (0 − yᵢ)xᵢ = (2/5)(−(40×1 + 50×2 + 55×3 + 65×4 + 75×5))
       = (2/5)(−(40 + 100 + 165 + 260 + 375)) = (2/5)(−940) = −376
 ∂/∂β₀ = (2/5) Σ (0 − yᵢ)  = (2/5)(−285) = −114

 β₁ ← 0 − 0.02 × (−376) = 7.52
 β₀ ← 0 − 0.02 × (−114) = 2.28
```

After the first step the line is already `ŷ = 2.28 + 7.52x`, much closer than `ŷ = 0`.

---

### 8. Gradient descent from scratch in Python

Let's code exactly what we did by hand. This is plain Python with numpy, no Spark.

```python
import numpy as np

x = np.array([1, 2, 3, 4, 5], dtype=float)
y = np.array([40, 50, 55, 65, 75], dtype=float)

m = c = 0.0           # m = slope (β₁), c = intercept (β₀)
lr = 0.02             # learning rate η

for it in range(1, 20001):
    yhat = m * x + c
    dm = (2 / len(x)) * np.sum((yhat - y) * x)      # gradient wrt slope
    dc = (2 / len(x)) * np.sum(yhat - y)            # gradient wrt intercept
    m -= lr * dm
    c -= lr * dc
    if it in (1, 2, 3, 10, 100, 1000, 20000):
        mse = np.mean((m * x + c - y) ** 2)
        print("iteration", it, "slope", round(m, 4), "intercept", round(c, 4), "MSE", round(mse, 4))
```

**Output**

```text
iteration 1     slope 7.52     intercept 2.28     MSE 1037.6864
iteration 2     slope 11.4576  intercept 3.5664   MSE 382.3089
iteration 3     slope 13.5083  intercept 4.3288   MSE 199.1986
iteration 10    slope 15.4795  intercept 6.1995   MSE 117.9541
iteration 100   slope 12.3042  intercept 17.7657  MSE 35.8342
iteration 1000  slope 8.5085   intercept 31.4694  MSE 1.5002
iteration 20000 slope 8.5      intercept 31.5     MSE 1.5
```

**Reading the story**

- Start: slope 0, MSE = 3395 (predicting 0 for everyone).
- Iteration 1 reproduces our hand step: **slope 7.52, intercept 2.28** ✔.
- The slope first **overshoots** (15.5 at iteration 10) because the intercept lags, then settles back.
- After 20,000 iterations: **slope 8.5, intercept 31.5, MSE 1.5**. It has reached exactly the least-squares answer from Sections 5 and 6. ✔

**Three ways, one answer:** scalar formulas, normal equation and gradient descent all give `ŷ = 31.5 + 8.5x`.

---

### 9. How Spark solves it

Spark's `LinearRegression` has a `solver` parameter.

| Solver | Method | Used when |
| --- | --- | --- |
| `"normal"` | Normal equation (weighted least squares) | Few features (up to 4096) and **no L1 regularization** |
| `"l-bfgs"` | Iterative quasi-Newton optimizer (a smarter gradient descent) | Many features, or L1/elastic net |
| `"auto"` (default) | Picks `normal` when possible, otherwise `l-bfgs` | Always a safe choice |

You will see this in the output: for the tiny example, `totalIterations` is **0**, meaning Spark solved it in one step with the normal equation.

---

### 10. PySpark code: the tiny example

```python
from pyspark.ml.feature import VectorAssembler
from pyspark.ml.regression import LinearRegression

df = spark.createDataFrame([(1, 40), (2, 50), (3, 55), (4, 65), (5, 75)], ["hours", "marks"])

va = VectorAssembler(inputCols=["hours"], outputCol="features")
data = va.transform(df).select("features", "marks")
data.show(truncate=False)
```

```text
+--------+-----+
|features|marks|
+--------+-----+
|[1.0]   |40   |
|[2.0]   |50   |
|[3.0]   |55   |
|[4.0]   |65   |
|[5.0]   |75   |
+--------+-----+
```

```python
lr = LinearRegression(featuresCol="features", labelCol="marks")
model = lr.fit(data)

print("coefficients:", model.coefficients)
print("intercept   :", model.intercept)
```

```text
coefficients: [8.500000000000005]
intercept   : 31.499999999999993
```

✔ **Spark found 8.5 and 31.5**, the same as our hand calculation (differences of 1e-15 are floating-point rounding).

```python
s = model.summary
print("RMSE:", s.rootMeanSquaredError)
print("R2  :", s.r2)
print("MAE :", s.meanAbsoluteError)
print("MSE :", s.meanSquaredError)
print("iterations:", s.totalIterations)
```

```text
RMSE: 1.2247448713915878
R2  : 0.9897260273972603
MAE : 0.9999999999999986
MSE : 1.4999999999999971
iterations: 0
```

✔ **All match the hand values:** RMSE 1.2247, R² 0.98973, MAE 1.0, MSE 1.5.

```python
model.transform(data).show()
```

```text
+--------+-----+-----------------+
|features|marks|       prediction|
+--------+-----+-----------------+
|   [1.0]|   40|             40.0|
|   [2.0]|   50|             48.5|
|   [3.0]|   55|57.00000000000001|
|   [4.0]|   65|65.50000000000001|
|   [5.0]|   75|74.00000000000003|
+--------+-----+-----------------+
```

Predictions are exactly the ŷ column from our hand table (40, 48.5, 57, 65.5, 74).

---

### 11. PySpark code: multiple regression on our students

Now the real thing: predict `final_marks` from **three features**, with a proper train/test split.

**Step 1: load and prepare** (the template from Phase 3)

```python
import sys
sys.path.append(r"C:\sparklabs")
from make_data import make_students
from pyspark.ml.feature import VectorAssembler
from pyspark.ml.regression import LinearRegression
from pyspark.ml.evaluation import RegressionEvaluator
from pyspark.sql import functions as F

df = spark.createDataFrame(make_students(),
        ["id", "hours", "attendance", "prev_marks", "final_marks", "passed"])

va = VectorAssembler(inputCols=["hours", "attendance", "prev_marks"], outputCol="features")
data = va.transform(df).select("features", F.col("final_marks").alias("label"))

train, test = data.randomSplit([0.8, 0.2], seed=42)
print(train.count(), test.count())
```

```text
165 35
```

**Step 2: train**

```python
lr = LinearRegression(featuresCol="features", labelCol="label")
model = lr.fit(train)

print("coefficients:", model.coefficients)
print("intercept   :", model.intercept)
```

```text
coefficients: [4.0612,0.2479,0.34]       (rounded to 4 decimals)
intercept   : 5.867
```

**Compare with the hidden truth we used to create the data:**

| Parameter | True value | Learned by Spark |
| --- | --- | --- |
| hours | 4 | **4.0612** |
| attendance | 0.25 | **0.2479** |
| prev_marks | 0.35 | **0.3400** |
| intercept | 5 | **5.867** |

The algorithm recovered the rule from noisy data. 🎉 (Small differences come from the random noise and the limited sample.)

**Step 3: predict on the test set**

```python
pred = model.transform(test)
pred.select("features", "label", "prediction").show(5, truncate=False)
```

```text
+---------------+-----+------------------+
|features       |label|prediction        |
+---------------+-----+------------------+
|[1.5,79.2,62.7]|53.8 |52.912683341693636|
|[1.6,88.0,77.1]|62.7 |60.39683743505971 |
|[1.8,52.4,41.0]|35.9 |40.10854581750547 |
|[2.8,82.5,65.0]|59.6 |59.79239269800211 |
|[3.0,99.9,63.0]|62.2 |64.23802321727736 |
+---------------+-----+------------------+
```

**Check one prediction by hand** (first row, hours 1.5, attendance 79.2, previous 62.7):

```text
 ŷ = 5.867 + 4.0612×1.5 + 0.2479×79.2 + 0.34×62.7
   = 5.867 + 6.092 + 19.634 + 21.318
   = 52.91            ✔ (actual 53.8)
```

**Step 4: evaluate on the test set**

```python
for metric in ("rmse", "mse", "mae", "r2"):
    value = RegressionEvaluator(labelCol="label", predictionCol="prediction", metricName=metric).evaluate(pred)
    print(metric, round(value, 4))
```

```text
rmse 2.8413
mse 8.073
mae 2.2595
r2 0.9291
```

**Meaning:** on unseen students, predictions are off by about **2.84 marks** (RMSE) and **2.26 marks** on average (MAE), and the model explains **92.9%** of the variation in marks.

**Step 5: predict for a brand-new student**

```python
from pyspark.ml.linalg import Vectors
print(model.predict(Vectors.dense([6.0, 80.0, 60.0])))
```

```text
70.46829670260948
```

A student studying 6 hours, with 80% attendance and 60 previous marks is predicted to score **about 70.5**.

---

### 12. Reading the model summary

`model.summary` (computed on the **training** data) gives rich statistics.

```python
s = model.summary
print("train RMSE:", round(s.rootMeanSquaredError, 4))
print("train R2  :", round(s.r2, 4))
print("train MAE :", round(s.meanAbsoluteError, 4))
print("adj R2    :", round(s.r2adj, 4))
print("std errors:", [round(v, 4) for v in s.coefficientStandardErrors])
print("t values  :", [round(v, 3) for v in s.tValues])
print("p values  :", [round(v, 4) for v in s.pValues])
```

```text
train RMSE: 3.1186
train R2  : 0.9442
train MAE : 2.5593
adj R2    : 0.9432
std errors: [0.0925, 0.0164, 0.0163, 1.6589]
t values  : [43.906, 15.097, 20.87, 3.537]
p values  : [0.0, 0.0, 0.0, 0.0005]
```

**The order is [hours, attendance, prev_marks, intercept]** (the intercept is last).

| Statistic | Meaning | Our values |
| --- | --- | --- |
| **Standard error** | Uncertainty of each coefficient | hours ±0.09 |
| **t value** | `coefficient / standard error`; larger |t| = more reliable | hours 43.9 |
| **p value** | Probability of seeing such a coefficient if the true effect were 0. **Small (< 0.05) = significant** | all ≈ 0 |
| **R²** | Share of variation explained | 0.944 (train) |
| **Adjusted R²** | R² penalised for adding features | 0.943 |

**Reading:** all three features and the intercept are **statistically significant** (p < 0.05). Test R² (0.929) is only slightly lower than train R² (0.944), so there is **no overfitting**.

**Residuals**

```python
s.residuals.show(3)
```

```text
+-------------------+
|          residuals|
+-------------------+
|-1.0672444456746888|
|-0.5802366557783998|
| -2.891658216144755|
+-------------------+
```

A residual is `label − prediction`. Residuals should look like random noise around 0 (Section 15).

| `summary` attribute | Meaning |
| --- | --- |
| `r2`, `r2adj` | R², adjusted R² |
| `rootMeanSquaredError`, `meanSquaredError`, `meanAbsoluteError` | Error metrics on training data |
| `coefficientStandardErrors`, `tValues`, `pValues` | Significance tests (only for the `normal` solver without regularization) |
| `residuals` | DataFrame of residuals |
| `predictions` | DataFrame with predictions |
| `totalIterations`, `objectiveHistory` | Optimizer progress |
| `numInstances`, `degreesOfFreedom` | Sample counts |

---

### 13. Interpreting coefficients

Interpretation rule:

> **A coefficient is the expected change in the label for a one-unit increase in that feature, with all other features held constant.**

| Coefficient | Value | Meaning |
| --- | --- | --- |
| hours | 4.0612 | Each extra daily study hour adds about **4.06 marks** |
| attendance | 0.2479 | Each extra 1% attendance adds about **0.25 marks** (so +10% ≈ +2.5 marks) |
| prev_marks | 0.3400 | Each extra previous mark adds about **0.34 marks** |
| intercept | 5.867 | Baseline when all features are 0 (a pure extrapolation) |

**Caution about comparing coefficient sizes:** hours (4.06) looks "more important" than attendance (0.25), but the features have different units and ranges. To compare importance fairly, **standardize** the features first, then compare coefficients.

---

### 14. Evaluating: RMSE, MAE, R²

Full treatment is in Phase 11. The essentials:

| Metric | Formula | Meaning | Good value |
| --- | --- | --- | --- |
| **MSE** | `Σ(y − ŷ)² / n` | Average squared error | Lower |
| **RMSE** | `√MSE` | Typical error in label units; punishes big errors | Lower |
| **MAE** | `Σ|y − ŷ| / n` | Average absolute error; robust to outliers | Lower |
| **R²** | `1 − SSE/SST` | Fraction of variance explained (1 = perfect, 0 = no better than the mean) | Closer to 1 |
| **Adjusted R²** | `1 − (1−R²)(n−1)/(n−k−1)` | R² corrected for the number of features k | Closer to 1 |

**Relationship:** RMSE ≥ MAE always. If RMSE is much larger than MAE, a few big errors exist.

**Our test results:** RMSE 2.84 ≥ MAE 2.26 ✔, R² = 0.929.

---

### 15. Assumptions of linear regression

The model works best when these hold.

| # | Assumption | Meaning | How to check |
| --- | --- | --- | --- |
| 1 | **Linearity** | The relationship is a straight line | Scatter plot; residuals show no curve |
| 2 | **Independence** | Rows do not influence each other | Think about how data was collected |
| 3 | **Homoscedasticity** | Residuals have **constant variance** | Residual-vs-prediction plot shows an even band, not a funnel |
| 4 | **Normal residuals** | Errors are roughly bell-shaped | Histogram of residuals |
| 5 | **No multicollinearity** | Features not strongly correlated with each other | Correlation heat map (Phase 4) |

```text
 GOOD residual plot            BAD (funnel = heteroscedasticity)     BAD (curve = non-linear)
  e │ · ·  ·  · ·               e │         ·  ·                      e │ ·            · ·
  0 │·──·──·──·──·── ŷ          0 │ ·  ·──·──·──·── ŷ               0 │──·──·        ·── ŷ
    │ ·  · ·   ·  ·               │ ·                                   │       · ·· 
```

If assumptions fail: transform the label (log), add polynomial features, remove correlated features, or use a non-linear model (Decision Tree).

---

### 16. Regularization: Ridge, Lasso, Elastic Net

**The problem:** with many features or correlated features, ordinary least squares can produce **huge, unstable coefficients** that overfit (Phase 2).

**The fix:** add a **penalty** on coefficient size to the loss.

| Method | Loss minimized | Penalty |
| --- | --- | --- |
| **OLS** (plain) | `SSE` | none |
| **Ridge** (L2) | `SSE + λ Σ βⱼ²` | Squares of coefficients |
| **Lasso** (L1) | `SSE + λ Σ |βⱼ|` | Absolute values |
| **Elastic Net** | `SSE + λ [ α Σ|βⱼ| + (1−α) Σ βⱼ² ]` | A mix |

**What each does**

| | Ridge (L2) | Lasso (L1) |
| --- | --- | --- |
| Effect on coefficients | **Shrinks** them towards 0 | Shrinks and can set some **exactly to 0** |
| Feature selection | No | **Yes** (drops useless features) |
| Handles correlated features | Well | Picks one of the group |

**The strength λ ("lambda")** controls the trade-off: λ = 0 means no penalty (OLS); a big λ shrinks the coefficients strongly (risking underfitting).

**Spark parameters**

| Spark parameter | Maths symbol | Meaning |
| --- | --- | --- |
| `regParam` | λ | Strength of the penalty |
| `elasticNetParam` | α | `0` = pure **Ridge (L2)**; `1` = pure **Lasso (L1)**; between = Elastic Net |

**Code and verified results**

```python
for name, reg, en in (("ridge", 0.1, 0.0), ("lasso", 0.1, 1.0), ("elastic", 0.1, 0.5)):
    m = LinearRegression(featuresCol="features", labelCol="label",
                         regParam=reg, elasticNetParam=en).fit(train)
    print(name, m.coefficients, round(m.intercept, 4))
```

```text
ridge   [4.0314,0.2463,0.3377]  6.3022
lasso   [4.026,0.2417,0.3338]   6.9256
elastic [4.0287,0.244,0.3357]   6.6127
```

**Compared with plain OLS** `[4.0612, 0.2479, 0.34]`, all regularized models shrink each coefficient slightly.

**A stronger penalty** (`regParam=2.0`, Lasso):

```text
lasso (λ = 2.0)  [3.3582, 0.1243, 0.2152]
```

The coefficients shrink a lot: a penalty that is too strong underfits. The value of λ is a **hyperparameter** tuned by cross-validation (Phase 11).

> Spark standardizes features internally (`standardization=True`, default) before penalizing, so the penalty treats features fairly regardless of their units. Reported coefficients are converted back to the original scale.

---

### 17. Important parameters

| Parameter | Default | Meaning |
| --- | --- | --- |
| `featuresCol` | `features` | Input vector column |
| `labelCol` | `label` | Target column |
| `predictionCol` | `prediction` | Output column |
| `maxIter` | 100 | Max iterations (iterative solvers) |
| `regParam` | 0.0 | λ, regularization strength |
| `elasticNetParam` | 0.0 | α, 0 = Ridge, 1 = Lasso |
| `fitIntercept` | True | Learn β₀ |
| `standardization` | True | Standardize features before fitting |
| `solver` | `auto` | `normal`, `l-bfgs` or `auto` |
| `tol` | 1e-6 | Convergence tolerance |
| `weightCol` | none | Per-row weights |

---

### 18. Strengths and weaknesses

| Strengths | Weaknesses |
| --- | --- |
| Simple, fast, scales to huge data | Assumes a **linear** relationship |
| **Interpretable** coefficients | Sensitive to **outliers** (squared loss) |
| Gives significance tests (p-values) | Affected by **multicollinearity** |
| A solid **baseline** for any regression task | Can underfit complex patterns |
| Regularization controls overfitting | Predicts values outside the realistic range (e.g. marks above 100) |

---

### 19. Common errors

| Symptom | Cause | Fix |
| --- | --- | --- |
| `Column features must be of type VectorUDT` | No `VectorAssembler` | Assemble features first |
| `label does not exist` | Target not named `label` | `.alias("label")` or `labelCol="final_marks"` |
| Very low R² on test, high on train | Overfitting | Regularize, fewer features, more data |
| R² suspiciously ≈ 1.0 | **Leakage** (label-derived feature) | Remove the leaking feature |
| Huge unstable coefficients | Multicollinearity | Drop/combine features, Ridge |
| `pValues` not available | Regularization or `l-bfgs` solver used | Statistics exist only for plain `normal` solver |
| Negative R² on test | Model worse than predicting the mean | Re-examine features and data |
| Mixing the coefficient order | Intercept is separate (`model.intercept`) | Remember `coefficients` follow `inputCols` order |
| Different results each run | No seed in `randomSplit` | `seed=42` |
| Predicting with a plain list | `predict` needs a Vector | `Vectors.dense([...])` |

---

### 20. Phase 5 — practical scenarios

1. By hand, fit `marks = β₀ + β₁·hours` for the 5-point dataset; confirm **31.5 and 8.5**.
2. Compute **SSE, MSE, RMSE, MAE and R²** by hand for that fit.
3. Solve the same problem with the **normal equation** (2×2 matrices).
4. Code **gradient descent** and observe the first step (7.52, 2.28).
5. Try learning rates **0.001, 0.02, 0.1**; explain what happens to convergence.
6. Fit the tiny example in **Spark**; compare coefficients with your hand result.
7. Fit **multiple regression** on 200 students; compare coefficients with the true (4, 0.25, 0.35, 5).
8. Print **test RMSE, MAE, R²** with `RegressionEvaluator`.
9. **Predict** for a new student (6 h, 80%, 60 marks).
10. Print the **summary**; identify significant features by p-value.
11. Fit **Ridge, Lasso and Elastic Net**; compare the coefficients.
12. Draw a **residual plot** (`toPandas()` on a sample) and judge the assumptions.

---

### 21. Phase 5 — viva questions and answers

#### Q1. What is linear regression?

A supervised algorithm that predicts a continuous value by fitting a straight line (or hyperplane) to the data.

#### Q2. Write the equation of simple and multiple linear regression.

Simple: `ŷ = β₀ + β₁x`. Multiple: `ŷ = β₀ + β₁x₁ + … + βₖxₖ`.

#### Q3. What are the intercept and coefficients?

The intercept β₀ is the prediction when all features are zero; each coefficient βⱼ is the change in prediction for a one-unit increase in feature j, other features fixed.

#### Q4. What is a residual?

The difference between the actual value and the predicted value, `y − ŷ`.

#### Q5. What is the least-squares principle?

Choose the coefficients that minimize the sum of squared residuals (SSE).

#### Q6. Why are errors squared?

So positive and negative errors do not cancel, larger errors are penalised more, and the loss is smooth and differentiable.

#### Q7. Give the formulas for slope and intercept in simple regression.

`β₁ = Σ(x−x̄)(y−ȳ) / Σ(x−x̄)²` and `β₀ = ȳ − β₁x̄`.

#### Q8. What is the normal equation?

`β = (XᵀX)⁻¹Xᵀy`, which gives the least-squares coefficients directly.

#### Q9. What is gradient descent?

An iterative optimization method that repeatedly moves the parameters opposite to the gradient of the loss by a step set by the learning rate.

#### Q10. What happens if the learning rate is too large or too small?

Too large: it overshoots and may diverge; too small: convergence is very slow.

#### Q11. Which solvers does Spark's LinearRegression use?

The normal equation (`normal`) for few features without L1, and L-BFGS (`l-bfgs`) otherwise; `auto` chooses.

#### Q12. What do RMSE, MAE and R² measure?

RMSE: typical error in label units (penalises big errors); MAE: average absolute error; R²: the fraction of variance explained.

#### Q13. What does R² = 0.93 mean?

The model explains 93% of the variance of the label.

#### Q14. What is adjusted R²?

R² adjusted for the number of features, so adding useless features does not inflate it.

#### Q15. State the assumptions of linear regression.

Linearity, independence, homoscedasticity, normally distributed residuals and no multicollinearity.

#### Q16. What is regularization and why is it used?

Adding a penalty on coefficient size to the loss to reduce overfitting and stabilise coefficients.

#### Q17. Difference between Ridge and Lasso?

Ridge (L2) shrinks coefficients; Lasso (L1) can shrink some exactly to zero, performing feature selection.

#### Q18. What do `regParam` and `elasticNetParam` mean in Spark?

`regParam` is the penalty strength λ; `elasticNetParam` is the L1/L2 mix: 0 = Ridge, 1 = Lasso.

#### Q19. What is multicollinearity and its effect on regression?

Strong correlation among features, which makes coefficients unstable and hard to interpret.

#### Q20. How do you interpret a p-value of 0.0005 for a coefficient?

The feature's effect is statistically significant; the chance of seeing such a coefficient if its true effect were zero is very small.

#### Q21. Why can linear regression predict marks above 100?

It extrapolates a straight line without bounds; the model does not know the limit of the label.

#### Q22. Why must we `fit` on training data and evaluate on test data?

To measure real performance on unseen data and detect overfitting.

---

### 22. Phase 5 — final master recap

```text
LINEAR REGRESSION          ŷ = β₀ + β₁x₁ + … + βₖxₖ          (predicts a NUMBER)
LOSS                       SSE = Σ(y − ŷ)²   (MSE = SSE/n, RMSE = √MSE)
SIMPLE SOLUTION            β₁ = Sxy/Sxx        β₀ = ȳ − β₁x̄
MATRIX SOLUTION            β = (XᵀX)⁻¹ Xᵀ y
GRADIENT DESCENT           β ← β − η · ∂Loss/∂β          (η = learning rate)
HAND EXAMPLE               hours→marks: 31.5 + 8.5·x ; SSE 7.5 ; RMSE 1.2247 ; MAE 1.0 ; R² 0.9897
STUDENTS RESULT            coef [4.0612, 0.2479, 0.34], intercept 5.867 ; test RMSE 2.84 ; R² 0.929
REGULARIZATION             Ridge L2 (shrink) · Lasso L1 (zero out) · regParam=λ · elasticNetParam=α
PYSPARK                    LinearRegression(featuresCol, labelCol).fit(train) → .transform(test)
```

**Seven facts to memorise**

1. Linear regression predicts a **number**.
2. Training = minimize **SSE** (sum of squared residuals).
3. `β₁ = Sxy/Sxx`, `β₀ = ȳ − β₁x̄`; the line passes through (x̄, ȳ).
4. Normal equation `β = (XᵀX)⁻¹Xᵀy` and gradient descent reach the same answer.
5. RMSE and MAE are in label units; **R² is a fraction** of explained variance.
6. `elasticNetParam`: **0 = Ridge, 1 = Lasso**.
7. Check test-vs-train R² to catch overfitting.

### Phase 5 completed

---

> **Next → Phase 6: Logistic Regression** (why linear regression fails for classes, the sigmoid, odds and log-odds, log-loss derived, gradient descent, decision threshold, multiclass softmax, regularization, PySpark code with probabilities, and a hand-worked tiny example). File: `6_Unit6_Logistic_Regression.md`
