# Unit VI — Phase 4: Data Understanding and the Correlation Heat Map

> **Continuing from Phase 3.** Before training any model we must **understand the data**. This phase teaches the statistics behind that (mean, variance, covariance, correlation) with the mathematics, a hand-worked example, Spark code, and the **correlation heat map**, one of the items in your syllabus's evaluation list.
>
> All code and outputs were run in a real PySpark 3.5 session with pandas, matplotlib and seaborn. The heat map code was executed and the image files were generated successfully.

---

## Phase 4 — Roadmap

| # | Topic |
| --- | --- |
| 1 | Why understand the data first |
| 2 | Descriptive statistics and their formulas |
| 3 | Checking missing values and summaries in Spark |
| 4 | Comparing groups |
| 5 | Outliers: the IQR rule |
| 6 | Variance and covariance |
| 7 | Pearson correlation: formula and hand example |
| 8 | Interpreting a correlation value |
| 9 | Correlation in Spark |
| 10 | Spearman rank correlation |
| 11 | Correlation is not causation |
| 12 | The correlation heat map |
| 13 | Reading our heat map |
| 14 | Multicollinearity |
| 15 | Using correlation for feature selection |
| 16 | EDA checklist |
| 17 | Common errors |
| 18 | Practical scenarios |
| 19 | Viva questions |
| 20 | Master revision |

---

### 1. Why understand the data first

> **Garbage in, garbage out.** A powerful algorithm cannot rescue bad or misunderstood data.

Exploratory data analysis (EDA) answers four questions before modelling:

| Question | Tool |
| --- | --- |
| What does each column look like? (range, average, spread) | `describe`, `summary` |
| Is anything **missing** or **strange**? | null counts, outlier rules |
| Which features **relate** to the label? | Correlation |
| Are features **duplicating** each other? | Correlation heat map |

This phase uses the 200-student dataset from Phase 2 (`make_students()`).

```python
import sys
sys.path.append(r"C:\sparklabs")
from make_data import make_students
from pyspark.sql import functions as F

df = spark.createDataFrame(make_students(),
         ["id", "hours", "attendance", "prev_marks", "final_marks", "passed"])
```

---

### 2. Descriptive statistics and their formulas

For values `x₁, x₂, …, xₙ`:

| Statistic | Formula | Meaning |
| --- | --- | --- |
| **Mean** (μ) | `μ = (x₁ + x₂ + … + xₙ) / n` | The average |
| **Median** | The middle value when sorted | Typical value; robust to outliers |
| **Mode** | The most frequent value | Most common category |
| **Variance** (σ²) | `σ² = Σ(xᵢ − μ)² / (n − 1)` | Average squared distance from the mean (spread) |
| **Standard deviation** (σ) | `σ = √σ²` | Spread in the **same units** as the data |
| **Quartiles** Q1, Q2, Q3 | 25%, 50%, 75% points | Where the data splits into quarters |
| **IQR** | `Q3 − Q1` | Spread of the middle 50% |

> Spark (like most tools) uses the **sample** variance with the divisor `n − 1`.

**Mean vs median.** If one student scored 1,000,000, the mean explodes but the median barely moves. When data has outliers, the median describes the "typical" value better.

---

### 3. Checking missing values and summaries in Spark

**Count missing values in every column**

```python
df.select([F.count(F.when(F.col(c).isNull(), c)).alias(c) for c in df.columns]).show()
```

```text
+---+-----+----------+----------+-----------+------+
| id|hours|attendance|prev_marks|final_marks|passed|
+---+-----+----------+----------+-----------+------+
|  0|    0|         0|         0|          0|     0|
+---+-----+----------+----------+-----------+------+
```

No missing values. (If a column showed a number > 0 we would use `Imputer` or drop those rows; Phase 3.)

**The `summary()` table with quartiles**

```python
df.select("hours", "attendance", "prev_marks", "final_marks") \
  .summary("count", "mean", "min", "25%", "50%", "75%", "max").show()
```

```text
+-------+-----------------+----------+-----------------+-----------+
|summary|            hours|attendance|       prev_marks|final_marks|
+-------+-----------------+----------+-----------------+-----------+
|  count|              200|       200|              200|        200|
|   mean|5.473000000000002|    75.732|62.41849999999999|    67.9515|
|    min|              1.0|      50.3|             35.0|       35.9|
|    25%|              3.2|      62.4|             48.8|       58.6|
|    50%|              5.3|      76.1|             63.0|       67.9|
|    75%|              7.5|      89.4|             73.9|       76.6|
|    max|             10.0|     100.0|             89.8|       98.2|
+-------+-----------------+----------+-----------------+-----------+
```

**Quartiles directly**

```python
print(df.approxQuantile("final_marks", [0.25, 0.5, 0.75], 0.0))
```

```text
[58.6, 67.9, 76.6]
```

(The last argument `0.0` is the allowed error; 0.0 means exact. Larger values are faster on huge data.)

**How to read the table:** the mean marks (67.95) is close to the median (67.9), so the distribution is roughly symmetric, with no strong outliers.

---

### 4. Comparing groups

A very useful first look: **how do passing and failing students differ?**

```python
df.groupBy("passed").agg(
    F.count("*").alias("n"),
    F.round(F.avg("hours"), 2).alias("avg_hours"),
    F.round(F.avg("attendance"), 1).alias("avg_att"),
    F.round(F.avg("prev_marks"), 1).alias("avg_prev")
).orderBy("passed").show()
```

```text
+------+---+---------+-------+--------+
|passed|  n|avg_hours|avg_att|avg_prev|
+------+---+---------+-------+--------+
|     0|100|     3.55|   72.2|    57.6|
|     1|100|     7.39|   79.3|    67.3|
+------+---+---------+-------+--------+
```

**What it tells us**

| Feature | Failed | Passed | Gap |
| --- | --- | --- | --- |
| Study hours | 3.55 | 7.39 | **Very large** |
| Attendance | 72.2 | 79.3 | Moderate |
| Previous marks | 57.6 | 67.3 | Moderate |

Study **hours** separates the two groups most strongly. We will see the same story in the correlation numbers.

---

### 5. Outliers: the IQR rule

An **outlier** is a value far from the rest. It can distort means, correlations and scale-based algorithms.

**The IQR (Tukey) rule**

```text
IQR         = Q3 − Q1
Lower fence = Q1 − 1.5 × IQR
Upper fence = Q3 + 1.5 × IQR
A value outside the fences is a suspected outlier.
```

**For `final_marks`** (Q1 = 58.6, Q3 = 76.6):

```text
IQR = 76.6 − 58.6 = 18.0
Lower fence = 58.6 − 1.5 × 18.0 = 31.6
Upper fence = 76.6 + 1.5 × 18.0 = 103.6
```

The minimum is 35.9 and the maximum is 98.2, both **inside** the fences, so there are no outliers.

**What to do with an outlier:** check whether it is an error (fix it), a rare but real value (keep it, perhaps use a robust method), or irrelevant (remove it). Never delete values just because they are inconvenient.

---

### 6. Variance and covariance

**Variance** measures how one variable spreads. **Covariance** measures how two variables **move together**.

**Formula (sample covariance)**

```text
                Σ (xᵢ − x̄)(yᵢ − ȳ)
 cov(X, Y)  =  ─────────────────────
                      n − 1
```

**How to read the sign**

| cov(X, Y) | Meaning |
| --- | --- |
| **Positive** | When X is above its mean, Y tends to be above its mean too (move together) |
| **Negative** | When X is above its mean, Y tends to be below its mean (move oppositely) |
| **About 0** | No linear relationship |

**Worked example.** Study hours (X) and marks (Y) for five students:

| i | X (hours) | Y (marks) | X − X̄ | Y − Ȳ | (X − X̄)(Y − Ȳ) |
| --- | --- | --- | --- | --- | --- |
| 1 | 1 | 40 | −2 | −17 | 34 |
| 2 | 2 | 50 | −1 | −7 | 7 |
| 3 | 3 | 55 | 0 | −2 | 0 |
| 4 | 4 | 65 | 1 | 8 | 8 |
| 5 | 5 | 75 | 2 | 18 | 36 |
| | **X̄ = 3** | **Ȳ = 57** | | | **Sum = 85** |

```text
cov(X, Y) = 85 / (5 − 1) = 21.25
```

**In Spark**

```python
t = spark.createDataFrame([(1.0, 40.0), (2.0, 50.0), (3.0, 55.0), (4.0, 65.0), (5.0, 75.0)], ["x", "y"])
print(t.stat.cov("x", "y"))
```

```text
21.25
```

✔ It matches the hand calculation.

**The limitation of covariance:** its size depends on the **units**. Hours × marks gives 21.25, but measuring hours in minutes would multiply it by 60. We cannot tell if 21.25 is "strong" or "weak". **Correlation fixes this.**

---

### 7. Pearson correlation: formula and hand example

> **Pearson correlation coefficient (r) is the covariance divided by the product of the two standard deviations. It measures the strength and direction of the *linear* relationship between two variables, always between −1 and +1.**

```text
                cov(X, Y)              Σ (xᵢ − x̄)(yᵢ − ȳ)
     r   =  ───────────────  =  ─────────────────────────────────
                σₓ × σᵧ          √[ Σ(xᵢ − x̄)² ] × √[ Σ(yᵢ − ȳ)² ]
```

Dividing by the standard deviations removes the units, so r is a **pure number** that can be compared across any pair of variables.

**Continue the worked example**

```text
Σ(x − x̄)(y − ȳ) = 85
Σ(x − x̄)²       = 4 + 1 + 0 + 1 + 4         = 10
Σ(y − ȳ)²       = 289 + 49 + 4 + 64 + 324   = 730

            85              85
r  =  ─────────────── = ──────── = 0.99485
        √(10 × 730)      85.44
```

**In Spark**

```python
print(t.stat.corr("x", "y"))
```

```text
0.9948497511671098
```

✔ Matches: **r = 0.9948**. Hours and marks move together almost perfectly in a straight line.

---

### 8. Interpreting a correlation value

```text
   -1 ────────────── 0 ────────────── +1
 perfect            none            perfect
 negative                           positive
```

| |r| | Strength |
| --- | --- |
| 0.0 to 0.2 | Very weak / none |
| 0.2 to 0.4 | Weak |
| 0.4 to 0.6 | Moderate |
| 0.6 to 0.8 | Strong |
| 0.8 to 1.0 | Very strong |

| Value | Meaning | Example |
| --- | --- | --- |
| **r = +1** | Perfect positive line | Temperature in °C vs °F |
| **r > 0** | Both rise together | Hours studied vs marks |
| **r ≈ 0** | No linear relationship | Shoe size vs marks |
| **r < 0** | One rises, the other falls | Absences vs marks |
| **r = −1** | Perfect negative line | Fuel left vs distance driven |

**Properties**

1. `r` is **symmetric**: `corr(X, Y) = corr(Y, X)`.
2. `r` is **unit-free**: changing hours to minutes does not change it.
3. `corr(X, X) = 1` (that is why heat map diagonals are all 1).
4. `r` only detects **straight-line** relationships.

**The straight-line limitation.** For `x = −2, −1, 0, 1, 2` and `y = x²` (a perfect U-curve), Pearson `r = 0`, even though `y` is completely determined by `x`. Always look at a **scatter plot** too.

---

### 9. Correlation in Spark

#### 9.1 Between two columns

```python
print(df.stat.corr("hours", "final_marks"))
```

```text
0.8335422081149119
```

#### 9.2 A whole correlation matrix: `Correlation.corr`

First assemble the columns into a vector, then use `pyspark.ml.stat.Correlation`.

```python
from pyspark.ml.feature import VectorAssembler
from pyspark.ml.stat import Correlation
import numpy as np
np.set_printoptions(precision=3, suppress=True)

cols = ["hours", "attendance", "prev_marks", "final_marks"]
va = VectorAssembler(inputCols=cols, outputCol="v").transform(df)

matrix = Correlation.corr(va, "v", "pearson").head()[0]     # a Spark Matrix
print(matrix.toArray())
```

```text
[[1.    0.015 0.007 0.834]
 [0.015 1.    0.059 0.318]
 [0.007 0.059 1.    0.419]
 [0.834 0.318 0.419 1.   ]]
```

**Reading it** (rows and columns follow `cols`): row 0 is `hours`, row 3 is `final_marks`. So the entry in row 0, column 3 (0.834) is `corr(hours, final_marks)`.

#### 9.3 With pandas (for small data)

```python
pdf = df.select("hours", "attendance", "prev_marks", "final_marks", "passed").toPandas()
print(pdf.corr().round(3))
```

```text
             hours  attendance  prev_marks  final_marks  passed
hours        1.000       0.015       0.007        0.834   0.732
attendance   0.015       1.000       0.059        0.318   0.239
prev_marks   0.007       0.059       1.000        0.419   0.323
final_marks  0.834       0.318       0.419        1.000   0.827
passed       0.732       0.239       0.323        0.827   1.000
```

| Method | Use when |
| --- | --- |
| `df.stat.corr(a, b)` | Just one pair |
| `Correlation.corr` | A matrix on a **big** Spark DataFrame (computed in parallel) |
| `pdf.corr()` | The data is small enough for pandas |

---

### 10. Spearman rank correlation

**Pearson** detects **linear** relationships. **Spearman** works on the **ranks** of the values, so it detects any **monotonic** relationship (always rising or always falling, even if curved) and is less sensitive to outliers.

```text
Spearman = Pearson correlation computed on the RANKS of the data
```

```python
Correlation.corr(va, "v", "spearman").head()[0].toArray()
```

```text
[[1.    0.02  0.008 0.841]
 [0.02  1.    0.063 0.317]
 [0.008 0.063 1.    0.395]
 [0.841 0.317 0.395 1.   ]]
```

The values are close to Pearson's because our relationship is almost linear.

| Pearson | Spearman |
| --- | --- |
| Linear relationship | Monotonic relationship |
| Uses actual values | Uses ranks |
| Sensitive to outliers | Robust |
| Default (`"pearson"`) | `"spearman"` |

---

### 11. Correlation is not causation

> **Two variables moving together does not mean one causes the other.**

| Pattern | Explanation |
| --- | --- |
| Ice-cream sales and drowning deaths correlate | Hidden cause: hot weather |
| Hours studied and marks correlate | Plausibly causal, but other factors also matter |
| Number of firemen and fire damage correlate | Bigger fires bring more firemen |

Correlation tells us what is **useful for prediction**, not what **causes** an outcome.

---

### 12. The correlation heat map

A **heat map** shows a matrix of numbers as **colours**. For a correlation matrix, it lets you spot strong relationships at a glance.

```text
              hours  attendance  prev_marks  final_marks
 hours        ████     ·           ·           ████████
 attendance    ·      ████         ·           ███
 prev_marks    ·       ·          ████         ████
 final_marks  ████████ ███         ████        ████
 (dark = strong, light = weak)
```

**Libraries:** `matplotlib` (drawing) and `seaborn` (statistical charts, with a ready-made `heatmap`).

**Complete code (Spark → pandas → heat map)**

```python
import pandas as pd
import matplotlib.pyplot as plt
import seaborn as sns
from pyspark.ml.feature import VectorAssembler
from pyspark.ml.stat import Correlation

cols = ["hours", "attendance", "prev_marks", "final_marks"]

# 1. Spark computes the correlation matrix (parallel, scalable)
va = VectorAssembler(inputCols=cols, outputCol="v").transform(df)
matrix = Correlation.corr(va, "v", "pearson").head()[0].toArray()

# 2. Wrap the small matrix in a pandas DataFrame (adds row/column labels)
corr = pd.DataFrame(matrix, index=cols, columns=cols)
print(corr.round(3))

# 3. Draw the heat map
plt.figure(figsize=(6, 5))
sns.heatmap(corr, annot=True, fmt=".2f", cmap="coolwarm", vmin=-1, vmax=1, square=True)
plt.title("Correlation heat map")
plt.tight_layout()
plt.savefig("C:/sparklabs/corr_heatmap.png")      # save the picture
plt.show()                                         # display it
```

**Printed matrix**

```text
             hours  attendance  prev_marks  final_marks
hours        1.000       0.015       0.007        0.834
attendance   0.015       1.000       0.059        0.318
prev_marks   0.007       0.059       1.000        0.419
final_marks  0.834       0.318       0.419        1.000
```

**What each argument does**

| Argument | Meaning |
| --- | --- |
| `corr` | The matrix to draw |
| `annot=True` | Print the number in each cell |
| `fmt=".2f"` | Two decimal places |
| `cmap="coolwarm"` | Blue = negative, red = positive, white = zero |
| `vmin=-1, vmax=1` | Fix the colour scale to the full correlation range |
| `square=True` | Square cells |
| `plt.savefig(...)` | Save as a PNG file |

**Hide the duplicate upper triangle (cleaner)**

```python
import numpy as np
mask = np.triu(np.ones_like(corr, dtype=bool))      # True for the upper triangle
sns.heatmap(corr, mask=mask, annot=True, fmt=".2f", cmap="coolwarm", vmin=-1, vmax=1)
```

A correlation matrix is **symmetric**, so the upper triangle only repeats the lower one.

> **Install note:** `pip install matplotlib seaborn` (Phase 1). If running on a server without a display, add `import matplotlib; matplotlib.use("Agg")` before `pyplot` and rely on `savefig`.

---

### 13. Reading our heat map

Using the matrix above:

| Pair | r | Interpretation |
| --- | --- | --- |
| hours ↔ final_marks | **0.834** | **Very strong** positive: the best predictor |
| prev_marks ↔ final_marks | 0.419 | Moderate positive |
| attendance ↔ final_marks | 0.318 | Weak-to-moderate positive |
| hours ↔ attendance | 0.015 | None (independent features ✔) |
| hours ↔ prev_marks | 0.007 | None ✔ |
| attendance ↔ prev_marks | 0.059 | None ✔ |

**Conclusions**

1. All three features are **positively** related to `final_marks`, and `hours` is by far the strongest.
2. The **features are almost uncorrelated with one another**, which is good for linear models (no multicollinearity).
3. This matches how we **built** the data: the formula gave `hours` the largest effect (4 × hours), with weaker effects for attendance (0.25) and previous marks (0.35).

> **Remember the scale trap:** `hours` has the highest correlation not only because its coefficient is largest but also because its *values* vary more relative to the rest. Correlation is about **linear association**, not raw coefficient size.

**Correlation with the class label (`passed`)** from the pandas table: `hours` 0.732, `prev_marks` 0.323, `attendance` 0.239. Again hours leads.

---

### 14. Multicollinearity

> **Multicollinearity means two or more features are strongly correlated with each other, so they carry the same information.**

**Example:** adding a column `hours_min = hours × 60` (the same quantity in minutes):

```python
df2 = df.withColumn("hours_min", F.col("hours") * 60)
print(df2.stat.corr("hours", "hours_min"))
```

```text
1.0000000000000004
```

The correlation is **1.0** (perfect duplicate).

**Why it is a problem**

| Effect | Explanation |
| --- | --- |
| **Unstable coefficients** (linear/logistic regression) | The model cannot decide how to split the effect between the two copies, so coefficients swing wildly with small data changes |
| Harder interpretation | You cannot say which feature matters |
| No accuracy gain | The duplicate adds nothing but complexity |

**Detecting it**

| Method | Rule of thumb |
| --- | --- |
| Heat map | Off-diagonal values with **|r| > 0.8** between features |
| Variance Inflation Factor | `VIF = 1 / (1 − R²ⱼ)`, where `R²ⱼ` comes from regressing feature j on the others. VIF > 5 to 10 signals trouble |

**Cures:** drop one of the correlated features, combine them (for example an average), use regularization (Phase 5), or apply PCA. Tree-based models and Naive Bayes are less affected.

---

### 15. Using correlation for feature selection

```text
STEP 1  Correlate every feature with the LABEL     → keep features with useful |r|
STEP 2  Correlate features with EACH OTHER         → if two features have |r| > 0.8, keep one
```

**Step 1 on our data** (correlation with `final_marks`):

| Feature | r with final_marks | Decision |
| --- | --- | --- |
| hours | 0.834 | Keep (strongest) |
| prev_marks | 0.419 | Keep |
| attendance | 0.318 | Keep |

**Step 2:** the largest feature-to-feature correlation is 0.059 (attendance vs prev_marks), so none are redundant. **Keep all three.**

> **Caution:** a feature with low linear correlation can still be useful in a **non-linear** way (the U-curve example). Correlation-based selection is a quick first filter, not the final word.

---

### 16. EDA checklist

Run this before every project.

| ✔ | Check | Spark tool |
| --- | --- | --- |
| ☐ | Row and column count | `df.count()`, `len(df.columns)` |
| ☐ | Data types are right | `df.printSchema()` |
| ☐ | Missing values | null-count query |
| ☐ | Range, mean, spread | `describe`, `summary` |
| ☐ | Outliers | IQR fences |
| ☐ | Class balance (classification) | `groupBy("label").count()` |
| ☐ | Group differences | `groupBy(label).agg(avg...)` |
| ☐ | Relationship with label | correlation with label |
| ☐ | Feature-to-feature redundancy | correlation heat map |
| ☐ | Any **label-derived** feature (leakage) | Think carefully! |

---

### 17. Common errors

| Symptom | Cause | Fix |
| --- | --- | --- |
| `Correlation.corr` complains about the column | Column is not a Vector | Use `VectorAssembler` first |
| `NaN` in the correlation matrix | Constant column (zero variance) or NaNs | Remove constant columns; fill NaNs |
| Heat map shows blank | Forgot `plt.show()` / `savefig` | Add `plt.show()` |
| `ModuleNotFoundError: seaborn` | Not installed | `pip install seaborn matplotlib` |
| Reading the matrix wrongly | Mixed up row/column order | Label the matrix with `index=cols, columns=cols` |
| High correlation taken as causation | Misinterpretation | Remember: correlation ≠ causation |
| `toPandas()` on a huge DataFrame | Memory blow-up | Compute the matrix in Spark; convert only the small matrix |
| Correlation of a categorical column | Text cannot be correlated | Index/encode first, or use group comparisons |
| Strong U-shape missed | Pearson only sees straight lines | Plot the data; try Spearman or non-linear models |

---

### 18. Phase 4 — practical scenarios

1. **Count the null values** in each column of the student DataFrame.
2. Print the **`summary()`** with quartiles and comment on mean vs median.
3. Compute the **IQR fences** for `prev_marks` using `approxQuantile` and look for outliers.
4. **Compare** averages of features for `passed = 0` and `passed = 1`.
5. Compute **covariance** of `hours` and `final_marks` with `df.stat.cov`.
6. By hand, compute the **correlation of x = [1,2,3,4,5] and y = [40,50,55,65,75]**, and confirm with Spark (0.9948).
7. Build the **Pearson correlation matrix** with `Correlation.corr`.
8. Build the **Spearman** matrix and compare it with Pearson.
9. Draw the **correlation heat map** with seaborn and save it as a PNG.
10. Draw the **lower-triangle** heat map using a mask.
11. Add a **duplicate feature** and show that the correlation is 1.0; explain the harm.
12. **Select features**: list which features you would keep and why.

---

### 19. Phase 4 — viva questions and answers

#### Q1. Why do we explore data before building a model?

To find missing values, outliers and relationships, and to avoid garbage-in garbage-out.

#### Q2. What is the difference between mean and median?

The mean is the arithmetic average and is sensitive to outliers; the median is the middle value and is robust.

#### Q3. What is variance and standard deviation?

Variance is the average squared distance from the mean (spread); standard deviation is its square root, in the original units.

#### Q4. What is the IQR rule for outliers?

Values below `Q1 − 1.5·IQR` or above `Q3 + 1.5·IQR`, where `IQR = Q3 − Q1`, are suspected outliers.

#### Q5. What is covariance?

A measure of how two variables move together; positive when they rise together, negative when one rises as the other falls.

#### Q6. Why do we prefer correlation over covariance?

Covariance depends on units and has no fixed scale; correlation divides by the standard deviations to give a unit-free value between −1 and +1.

#### Q7. State the formula for Pearson correlation.

`r = Σ(xᵢ − x̄)(yᵢ − ȳ) / √[Σ(xᵢ − x̄)² · Σ(yᵢ − ȳ)²]`, equal to `cov(X, Y) / (σₓσᵧ)`.

#### Q8. What does a correlation of −0.9 mean?

A very strong negative linear relationship: as one variable rises the other falls almost proportionally.

#### Q9. What does a correlation near 0 mean?

No linear relationship (there may still be a non-linear one).

#### Q10. What is the range of the correlation coefficient?

−1 to +1.

#### Q11. What is the difference between Pearson and Spearman correlation?

Pearson measures linear association on the actual values; Spearman applies Pearson to ranks and measures monotonic association, being robust to outliers.

#### Q12. Does correlation imply causation?

No. Variables can move together because of a hidden common cause or coincidence.

#### Q13. What is a correlation heat map?

A colour-coded matrix of pairwise correlations that shows strong and weak relationships at a glance.

#### Q14. Which Spark class computes a correlation matrix?

`pyspark.ml.stat.Correlation` with `Correlation.corr(df, "vectorCol", "pearson")`.

#### Q15. Why are the diagonal cells of a correlation heat map all 1?

Every variable is perfectly correlated with itself.

#### Q16. What is multicollinearity and why is it harmful?

Strong correlation among features; it makes regression coefficients unstable and hard to interpret without improving accuracy.

#### Q17. How can you detect and fix multicollinearity?

Detect with a heat map (|r| > 0.8) or VIF; fix by dropping or combining features, regularization or PCA.

#### Q18. Which feature was the best predictor of final marks in our data and what was r?

`hours`, with r = 0.834.

#### Q19. Which Python libraries draw the heat map?

`matplotlib` and `seaborn` (`sns.heatmap`).

#### Q20. Why convert only the correlation matrix, not the whole data, to pandas?

The matrix is small, but `toPandas()` on the whole big dataset would exhaust the driver's memory.

---

### 20. Phase 4 — final master recap

```text
STATISTICS   mean · median · variance σ² = Σ(x−μ)²/(n−1) · std σ · quartiles · IQR
OUTLIERS     fences: Q1 − 1.5·IQR , Q3 + 1.5·IQR
COVARIANCE   cov = Σ(x−x̄)(y−ȳ)/(n−1)         (unit-dependent)
CORRELATION  r = cov /(σx·σy)  ∈ [−1, +1]      (unit-free, linear only)
SPARK        df.stat.corr(a,b)   Correlation.corr(vec,"pearson"/"spearman")
HEAT MAP     matrix → pandas → sns.heatmap(annot=True, cmap="coolwarm", vmin=-1, vmax=1)
INSIGHT      hours→marks r=0.834 ; features mutually ~0 (no multicollinearity)
TRAP         correlation ≠ causation ; |r|>0.8 between features = multicollinearity
```

**Seven facts to memorise**

1. Pearson `r` ranges from −1 to +1 and is unit-free.
2. `r = cov(X,Y) / (σₓ σᵧ)`.
3. `r` sees only **linear** relationships.
4. Spearman = Pearson on ranks.
5. Heat map diagonal = 1; matrix is symmetric.
6. |r| > 0.8 between two features → multicollinearity.
7. Correlation ≠ causation.

### Phase 4 completed

---

> **Next → Phase 5: Linear Regression** (the model, cost function, least squares and the normal equation derived by hand, gradient descent from scratch, regularization (Ridge/Lasso/Elastic Net), assumptions, PySpark code, reading the model summary, R², RMSE and MAE). File: `5_Unit6_Linear_Regression.md`
