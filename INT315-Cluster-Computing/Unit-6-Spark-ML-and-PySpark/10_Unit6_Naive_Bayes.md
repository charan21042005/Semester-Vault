# Unit VI — Phase 10: Naive Bayes

> **Continuing from Phase 9.** SVM drew a boundary; trees asked questions. **Naive Bayes classifies with probability theory.** It asks: *given what I observe, which class is most probable?* It uses **Bayes' theorem** plus one bold simplification, the "naive" assumption that features are independent. We build the maths from scratch, work the weather example **completely by hand** (and watch Spark reproduce it digit for digit), learn **Laplace smoothing**, the three Spark variants (Multinomial, Bernoulli, Gaussian), and classify text.
>
> All numbers are real. Spark's output for the weather query (0.2046 vs 0.7954) matches our hand calculation exactly, and the text-classification example matches by hand too (0.6927).

---

## Phase 10 — Roadmap

| # | Topic |
| --- | --- |
| 1 | The idea in plain words |
| 2 | Probability basics |
| 3 | Bayes' theorem |
| 4 | From Bayes to the Naive Bayes classifier |
| 5 | The "naive" assumption |
| 6 | A complete hand-worked example: weather data |
| 7 | The zero-frequency problem and Laplace smoothing |
| 8 | Avoiding underflow: log probabilities |
| 9 | Variants: Multinomial, Bernoulli, Gaussian |
| 10 | PySpark code: weather data (Multinomial) |
| 11 | Reading the model: `pi` and `theta` |
| 12 | Gaussian Naive Bayes: continuous features |
| 13 | PySpark code: flowers (Gaussian) |
| 14 | PySpark code: students, and choosing the right variant |
| 15 | Text classification with Naive Bayes |
| 16 | Important parameters |
| 17 | Strengths and weaknesses |
| 18 | Common errors |
| 19 | Practical scenarios |
| 20 | Viva questions |
| 21 | Master revision |

---

### 1. The idea in plain words

Think of how you decide whether an email is spam. You notice words: *"free"*, *"win"*, *"money"*. Each word raises your suspicion a little. You combine all these small clues into one belief.

> **Naive Bayes computes, for each class, the probability of that class *given* the observed features, and predicts the class with the highest probability.**

```text
 features seen: "free", "win", "money"
      │
      ├── how likely are these words in SPAM emails?   high
      ├── how likely are these words in NORMAL emails? low
      └── how common is spam overall?                  (prior)
              ▼
   combine with Bayes' theorem → P(spam | words) = 0.97   → predict SPAM
```

---

### 2. Probability basics

| Notation | Meaning | Example |
| --- | --- | --- |
| `P(A)` | Probability of event A | P(Play = Yes) = 9/14 |
| `P(A, B)` | Probability that **both** A and B happen (joint) | |
| `P(A | B)` | Probability of A **given that B happened** (conditional) | P(Humidity = High | Play = No) = 4/5 |

**Product rule**

```text
 P(A, B) = P(A | B) · P(B) = P(B | A) · P(A)
```

**Independence.** Events A and B are **independent** if knowing one tells nothing about the other:

```text
 P(A, B) = P(A) · P(B)         (and P(A | B) = P(A))
```

---

### 3. Bayes' theorem

From the product rule, `P(A | B) P(B) = P(B | A) P(A)`. Divide both sides by `P(B)`:

```text
                  P(B | A) · P(A)
 P(A | B)   =   ─────────────────
                       P(B)
```

In classification, A is the **class** (C) and B is the **evidence** (the features X):

```text
                    P(X | C) · P(C)
 P(C | X)   =   ─────────────────────
                         P(X)
```

| Term | Name | Meaning |
| --- | --- | --- |
| `P(C | X)` | **Posterior** | Probability of the class **after** seeing the features: *what we want* |
| `P(C)` | **Prior** | How common the class is **before** seeing features |
| `P(X | C)` | **Likelihood** | How likely these features are **if** the class is C |
| `P(X)` | **Evidence** | Overall probability of the features (same for all classes) |

**A classic numeric example (why priors matter).** A disease affects 1% of people. A test detects it 99% of the time (sensitivity) but also gives a false positive for 5% of healthy people. If you test **positive**, what is the chance you are ill?

```text
 P(ill) = 0.01     P(+ | ill) = 0.99     P(+ | healthy) = 0.05     P(healthy) = 0.99

 P(+) = 0.99 × 0.01 + 0.05 × 0.99 = 0.0099 + 0.0495 = 0.0594

                  0.99 × 0.01
 P(ill | +) = ─────────────── = 0.0099 / 0.0594 = 0.1667   (only 16.7%!)
                    0.0594
```

Despite the 99% accurate test, a positive result means only a **1 in 6** chance of illness, because the disease is rare (a small **prior**). Bayes' theorem combines prior and likelihood correctly.

---

### 4. From Bayes to the Naive Bayes classifier

For classes `C₁, C₂, …` and features `X = (x₁, x₂, …, xₙ)`:

```text
 P(Cₖ | x₁ … xₙ)  ∝  P(Cₖ) · P(x₁, x₂, …, xₙ | Cₖ)
```

The symbol `∝` means "proportional to": we can **drop the evidence `P(X)`**, because it is the same for every class and does not change which class wins.

The joint likelihood `P(x₁, …, xₙ | Cₖ)` is hard to estimate (we would need examples of every feature *combination*). Naive Bayes simplifies it with the independence assumption (next section).

**Prediction rule**

```text
 predicted class = argmax over k of   P(Cₖ) · Π P(xᵢ | Cₖ)
```

---

### 5. The "naive" assumption

> **Naive Bayes assumes the features are conditionally independent given the class:**

```text
 P(x₁, x₂, …, xₙ | C) = P(x₁ | C) · P(x₂ | C) · … · P(xₙ | C)
```

So instead of one impossible joint probability, we estimate **one simple probability per feature per class**, by counting.

| Why "naive"? | Because it is rarely exactly true |
| --- | --- |
| Example | In emails, the words "free" and "win" often appear *together*, so they are **not** independent given "spam". |
| Yet it works | Even with imperfect independence, the **ranking** of classes is often still right. Naive Bayes is a famously strong baseline, especially for **text**. |

**Benefits of the assumption:** training is just counting (extremely fast), needs little data, scales to huge datasets and thousands of features.

---

### 6. A complete hand-worked example: weather data

Use the 14-day weather table from Phase 7. **Task:** for a new day — **Outlook = Sunny, Temp = Cool, Humidity = High, Wind = Strong** — predict **Play = Yes or No**.

**Step 1: priors** (count the classes)

```text
 P(Yes) = 9/14 = 0.6429          P(No) = 5/14 = 0.3571
```

**Step 2: frequency tables** (count each attribute value within each class)

| Outlook | Yes (9) | No (5) |
| --- | --- | --- |
| Sunny | 2 | 3 |
| Overcast | 4 | 0 |
| Rain | 3 | 2 |

| Temp | Yes (9) | No (5) |
| --- | --- | --- |
| Hot | 2 | 2 |
| Mild | 4 | 2 |
| Cool | 3 | 1 |

| Humidity | Yes (9) | No (5) |
| --- | --- | --- |
| High | 3 | 4 |
| Normal | 6 | 1 |

| Wind | Yes (9) | No (5) |
| --- | --- | --- |
| Weak | 6 | 2 |
| Strong | 3 | 3 |

**Step 3: likelihoods for our query** (count ÷ class size)

| Feature value | P(· | Yes) | P(· | No) |
| --- | --- | --- |
| Outlook = Sunny | 2/9 = 0.2222 | 3/5 = 0.6 |
| Temp = Cool | 3/9 = 0.3333 | 1/5 = 0.2 |
| Humidity = High | 3/9 = 0.3333 | 4/5 = 0.8 |
| Wind = Strong | 3/9 = 0.3333 | 3/5 = 0.6 |

**Step 4: multiply (prior × likelihoods)**

```text
 Score(Yes) = P(Yes) × P(Sunny|Yes) × P(Cool|Yes) × P(High|Yes) × P(Strong|Yes)
            = (9/14) × (2/9) × (3/9) × (3/9) × (3/9)
            = 0.6429 × 0.2222 × 0.3333 × 0.3333 × 0.3333 = 0.00529

 Score(No)  = P(No) × P(Sunny|No) × P(Cool|No) × P(High|No) × P(Strong|No)
            = (5/14) × (3/5) × (1/5) × (4/5) × (3/5)
            = 0.3571 × 0.6 × 0.2 × 0.8 × 0.6 = 0.02057
```

**Step 5: normalize to get proper probabilities**

```text
 P(Yes | X) = 0.00529 / (0.00529 + 0.02057) = 0.2046
 P(No  | X) = 0.02057 / (0.00529 + 0.02057) = 0.7954
```

**Decision: No** (79.5%). *Do not play tennis on a sunny, cool, humid, windy day.*

**Reading it:** high humidity (4/5 of the "No" days) and sunny (3/5) both point to "No", outweighing the higher prior for "Yes".

> The scores 0.00529 and 0.02057 are not probabilities yet (they do not sum to 1). **Normalizing** by their sum converts them. The *winner* is the same either way.

---

### 7. The zero-frequency problem and Laplace smoothing

**The problem.** Suppose a new day has **Outlook = Overcast**. Look at the table: Overcast appeared **0** times with "No". Then:

```text
 P(Overcast | No) = 0/5 = 0   →   Score(No) = … × 0 × … = 0
```

A **single zero wipes out** the whole product. The model would say "No is impossible" just because we never *happened* to see Overcast on a "No" day in 14 samples.

**The fix: Laplace (add-one) smoothing.** Add a small count `α` (usually 1) to every count, and add `α × (number of possible values)` to the denominator:

```text
                   count(value, class) + α
 P(value | class) = ────────────────────────────
                      count(class) + α × k          k = number of distinct values of the feature
```

**Example:** Outlook has k = 3 values (Sunny, Overcast, Rain). With α = 1:

```text
 P(Overcast | No) = (0 + 1) / (5 + 3) = 1/8 = 0.125     (no longer zero)
 P(Sunny    | No) = (3 + 1) / (5 + 3) = 4/8 = 0.5
```

**Our query again, with smoothing** (k = 3 for Outlook and Temp, k = 2 for Humidity and Wind):

| Value | P(· | Yes) | P(· | No) |
| --- | --- | --- |
| Sunny | (2+1)/(9+3) = 0.2500 | (3+1)/(5+3) = 0.5000 |
| Cool | (3+1)/(9+3) = 0.3333 | (1+1)/(5+3) = 0.2500 |
| High | (3+1)/(9+2) = 0.3636 | (4+1)/(5+2) = 0.7143 |
| Strong | (3+1)/(9+2) = 0.3636 | (3+1)/(5+2) = 0.5714 |

```text
 Score(Yes) = 0.6429 × 0.25 × 0.3333 × 0.3636 × 0.3636 = 0.00709
 Score(No)  = 0.3571 × 0.50 × 0.2500 × 0.7143 × 0.5714 = 0.01822
 P(Yes | X) = 0.00709 / (0.00709 + 0.01822) = 0.2799      → still predicts No
```

Smoothing slightly moved the probability (0.2046 → 0.2799) but not the decision.

| `α` (Spark's `smoothing`) | Effect |
| --- | --- |
| 0 | No smoothing: zero-frequency problem possible |
| **1.0 (default)** | Laplace smoothing |
| Between 0 and 1 | Lidstone smoothing (gentler) |

---

### 8. Avoiding underflow: log probabilities

With hundreds of features (words), multiplying many small probabilities gives numbers like `10⁻³⁰⁰` that a computer **underflows to 0**. The fix is to **add logarithms** instead of multiplying:

```text
 log Score(C) = log P(C) + Σ log P(xᵢ | C)
```

Since `log` is increasing, the class with the largest log-score is the same as the class with the largest product. Spark stores `log P(C)` in the model attribute **`pi`** and the log conditional probabilities in **`theta`** (Section 11).

---

### 9. Variants: Multinomial, Bernoulli, Gaussian

The "likelihood" `P(xᵢ | C)` depends on the **type of feature**. Spark offers four `modelType` values.

| Model type | Features it expects | Likelihood | Typical use |
| --- | --- | --- | --- |
| **multinomial** (default) | **Non-negative counts** (word counts, TF-IDF), also one-hot | `(N_wc + α) / (N_c + α·V)` | Text, categorical data |
| **bernoulli** | **Binary 0/1** (word present or absent) | Probability that the feature is 1 in the class | Short text, yes/no attributes |
| **gaussian** | **Continuous** numbers | Normal density `N(x; μ_c, σ²_c)` | Measurements (height, marks) |
| **complement** | Non-negative counts | Uses the *complement* of each class | Imbalanced text data |

**The Gaussian likelihood**

```text
                           1               ⎛  (x − μ)²  ⎞
 P(x | C) = ──────────────────  · exp ⎜ − ────────────⎟
              √( 2π σ²_C )              ⎝    2 σ²_C    ⎠

 μ_C = mean of the feature within class C       σ²_C = variance of the feature within class C
```

For each feature and class, Spark simply learns **one mean and one variance**.

> **Choosing the variant:** counts → Multinomial; yes/no → Bernoulli; real-valued → Gaussian. A wrong choice (for example Multinomial on continuous marks) gives poor results (Section 14).

---

### 10. PySpark code: weather data (Multinomial)

The weather features are **categorical text**. We index them, then **one-hot encode** them (so they become non-negative 0/1 features), then use Multinomial Naive Bayes.

```python
from pyspark.ml.feature import StringIndexer, OneHotEncoder, VectorAssembler
from pyspark.ml.classification import NaiveBayes
from pyspark.ml import Pipeline
from pyspark.ml.evaluation import MulticlassClassificationEvaluator

W = [("Sunny","Hot","High","Weak","No"), ("Sunny","Hot","High","Strong","No"),
     ("Overcast","Hot","High","Weak","Yes"), ("Rain","Mild","High","Weak","Yes"),
     ("Rain","Cool","Normal","Weak","Yes"), ("Rain","Cool","Normal","Strong","No"),
     ("Overcast","Cool","Normal","Strong","Yes"), ("Sunny","Mild","High","Weak","No"),
     ("Sunny","Cool","Normal","Weak","Yes"), ("Rain","Mild","Normal","Weak","Yes"),
     ("Sunny","Mild","Normal","Strong","Yes"), ("Overcast","Mild","High","Strong","Yes"),
     ("Overcast","Hot","Normal","Weak","Yes"), ("Rain","Mild","High","Strong","No")]

cols = ["outlook", "temp", "humidity", "wind"]
wdf = spark.createDataFrame(W, cols + ["play"])

indexers = [StringIndexer(inputCol=c, outputCol=c + "_i") for c in cols]
encoder = OneHotEncoder(inputCols=[c + "_i" for c in cols],
                        outputCols=[c + "_v" for c in cols], dropLast=False)   # keep ALL categories
assembler = VectorAssembler(inputCols=[c + "_v" for c in cols], outputCol="features")
label_idx = StringIndexer(inputCol="play", outputCol="label")

nb = NaiveBayes(smoothing=0.0, modelType="multinomial")      # smoothing 0 → to match our hand calculation

model = Pipeline(stages=indexers + [encoder, assembler, label_idx, nb]).fit(wdf)
```

> **Why `dropLast=False`?** Earlier we dropped the last category to avoid a redundant column (for regression). For Naive Bayes we want **every category** as its own feature so that each gets a likelihood.

**Predict our query day**

```python
query = spark.createDataFrame([("Sunny", "Cool", "High", "Strong")], cols)
r = model.transform(query).select("probability", "prediction").first()
print(r.probability, r.prediction)
```

```text
[0.2046, 0.7954]   1.0
```

✔ **Spark gives P(Yes) = 0.2046 and P(No) = 0.7954, exactly our hand-calculated values.** The prediction `1.0` is the index of "No" (the label indexer put `Yes = 0.0`, `No = 1.0`).

**Why does Multinomial NB on one-hot features equal the categorical hand calculation?** Each one-hot row has exactly **four** 1s (one per attribute). Multinomial NB divides by the *total* count of ones in the class (4 × class size), not by the class size. That denominator is a **constant factor of 4⁴** for *every* class, so it cancels when we compare classes and normalize. (With `smoothing=0` the match is exact.)

**With the default Laplace smoothing** (`smoothing=1.0`):

```text
 probability [Yes, No] = [0.2657, 0.7343]    prediction: No
```

This differs slightly from our per-attribute Laplace result (0.2799) because Spark's multinomial smoothing adds `α` to **each of the 10 one-hot features** and `α × 10` to the denominator, rather than smoothing each attribute separately. The **decision (No) is the same**.

**Training accuracy**

```python
pred = model.transform(wdf)
print(MulticlassClassificationEvaluator(labelCol="label", metricName="accuracy").evaluate(pred))
```

```text
0.9286
```

13 of 14 days are classified correctly. The independence assumption is not exactly true here (for example, temperature and outlook are related), which causes the single mistake.

---

### 11. Reading the model: `pi` and `theta`

```python
import math
nbm = model.stages[-1]
print("priors     :", [round(math.exp(x), 4) for x in nbm.pi])
print("theta shape:", nbm.theta.numRows, "x", nbm.theta.numCols)
```

```text
priors     : [0.6429, 0.3571]
theta shape: 2 x 10
```

| Attribute | Meaning |
| --- | --- |
| **`pi`** | **Log of the class priors** `log P(C)`. `exp(pi)` = `[9/14, 5/14]` = `[0.6429, 0.3571]` ✔ |
| **`theta`** | Matrix of **log conditional probabilities** `log P(feature | class)` (one row per class, one column per feature). For Gaussian NB it holds the **means** |
| `sigma` | (Gaussian only) the **variances** |

**Look inside `theta`** (exponentiated, class `Yes` first, then `No`; the 10 columns are the one-hot features Rain, Sunny, Overcast, Mild, Cool, Hot, High, Normal, Weak, Strong):

```text
 Yes: [0.083, 0.056, 0.111, 0.111, 0.083, 0.056, 0.083, 0.167, 0.167, 0.083]
 No:  [0.100, 0.150, 0.000, 0.100, 0.050, 0.100, 0.200, 0.050, 0.100, 0.150]
```

Check one value: for class **Yes**, feature **Rain** = 3 Rain days out of the 36 total feature-ones (4 attributes × 9 Yes days) = 3/36 = **0.083** ✔. For class **No**, **Overcast** = 0/20 = **0.000** (the zero-frequency problem, visible!).

---

### 12. Gaussian Naive Bayes: continuous features

For numeric features we assume each feature is **normally distributed within each class** (Section 9). Training = computing the **mean and variance of every feature in every class**. Prediction = multiply the priors by the **Gaussian densities**.

**Single-feature density by hand.** Iris-style flowers: the petal length of **Virginica** has mean 5.68 and variance 0.1016; **Versicolor** has mean 4.54 and variance 0.0904. A flower with **petal length 5.0**:

```text
 Virginica:  1/√(2π·0.1016) × exp(−(5.0 − 5.68)² / (2·0.1016)) = 1.2516 × exp(−2.2756) = 0.1286
 Versicolor: 1/√(2π·0.0904) × exp(−(5.0 − 4.54)² / (2·0.0904)) = 1.3268 × exp(−1.1704) = 0.4117
```

On petal length alone, Versicolor looks more likely (density 0.41 vs 0.13). The classifier multiplies such densities for **all** features and by the prior, so other features (petal width, sepal length…) can change the verdict, as the next example shows.

---

### 13. PySpark code: flowers (Gaussian)

The 15-row Iris sample from Phase 6 (3 species × 5 flowers).

```python
fl = [(5.1,3.5,1.4,0.2,0),(4.9,3.0,1.4,0.2,0),(4.7,3.2,1.3,0.2,0),(4.6,3.1,1.5,0.2,0),(5.0,3.6,1.4,0.2,0),
      (7.0,3.2,4.7,1.4,1),(6.4,3.2,4.5,1.5,1),(6.9,3.1,4.9,1.5,1),(5.5,2.3,4.0,1.3,1),(6.5,2.8,4.6,1.5,1),
      (6.3,3.3,6.0,2.5,2),(5.8,2.7,5.1,1.9,2),(7.1,3.0,5.9,2.1,2),(6.3,2.9,5.6,1.8,2),(6.5,3.0,5.8,2.2,2)]

fdf = spark.createDataFrame(fl, ["sl", "sw", "pl", "pw", "label"])
fdf = VectorAssembler(inputCols=["sl", "sw", "pl", "pw"], outputCol="features").transform(fdf)

g = NaiveBayes(modelType="gaussian").fit(fdf)

print("accuracy:", MulticlassClassificationEvaluator(metricName="accuracy").evaluate(g.transform(fdf)))
print("class means:")
for row in g.theta.toArray():
    print([round(float(v), 3) for v in row])
```

```text
accuracy: 1.0
class means:
[4.86, 3.28, 1.4, 0.2]
[6.46, 2.92, 4.54, 1.44]
[6.4, 2.98, 5.68, 2.1]
```

**Reading the learned means** (rows = species 0, 1, 2; columns = sepal length, sepal width, petal length, petal width):

| Species | Sepal L | Sepal W | **Petal L** | **Petal W** |
| --- | --- | --- | --- | --- |
| 0 (setosa) | 4.86 | 3.28 | **1.40** | **0.20** |
| 1 (versicolor) | 6.46 | 2.92 | **4.54** | **1.44** |
| 2 (virginica) | 6.40 | 2.98 | **5.68** | **2.10** |

The petal measurements clearly separate the species. The class variances (`g.sigma`) are the other half of the model; for example, setosa's petal length variance is only **0.004** (very consistent), and its petal width variance is essentially 0.

**Priors** are equal (`[0.3333, 0.3333, 0.3333]`, five flowers each).

**Classify a new flower** (sepal 6.0×3.0, petal 5.0×1.8):

```python
from pyspark.ml.linalg import Vectors
x = Vectors.dense([6.0, 3.0, 5.0, 1.8])
print(g.predict(x), g.predictProbability(x))
```

```text
2.0   [0.0, 0.0004, 0.9996]
```

The model says **species 2 (virginica) with probability 0.9996**. Although petal length alone (Section 12) leaned to versicolor, the **petal width of 1.8** is much closer to virginica's mean (2.10) than versicolor's (1.44), and combining all four features gives virginica.

---

### 14. PySpark code: students, and choosing the right variant

The student features (`hours`, `attendance`, `prev_marks`) are **continuous**, so **Gaussian** NB is the right variant. We also try **Multinomial** to see what a wrong choice costs.

```python
import sys
sys.path.append(r"C:\sparklabs")
from make_data import make_students
from pyspark.sql import functions as F

df = spark.createDataFrame(make_students(),
        ["id", "hours", "attendance", "prev_marks", "final_marks", "passed"])
va = VectorAssembler(inputCols=["hours", "attendance", "prev_marks"], outputCol="features")
data = va.transform(df).select("features", F.col("passed").alias("label"))
train, test = data.randomSplit([0.8, 0.2], seed=42)

ev = MulticlassClassificationEvaluator(metricName="accuracy")

gnb = NaiveBayes(modelType="gaussian").fit(train)
print("Gaussian    accuracy:", round(ev.evaluate(gnb.transform(test)), 4))

mnb = NaiveBayes(modelType="multinomial").fit(train)
print("Multinomial accuracy:", round(ev.evaluate(mnb.transform(test)), 4))
```

```text
Gaussian    accuracy: 0.8286
Multinomial accuracy: 0.6857
```

**What the Gaussian model learned** (class means; fail = 0, pass = 1):

```text
 priors [fail, pass] = [0.4545, 0.5455]

 class means [hours, attendance, prev_marks]:
   fail (0):  [3.53, 70.67, 57.77]
   pass (1):  [7.47, 79.15, 67.47]
```

Failing students study 3.5 hours on average; passing students 7.5. The model is simply comparing a new student with these two profiles.

**Comparison on the same split**

| Model | Test accuracy |
| --- | --- |
| Logistic Regression | **0.9143** |
| Linear SVM | **0.9143** |
| Decision Tree (depth 4) | 0.8857 |
| **Gaussian Naive Bayes** | 0.8286 |
| Multinomial NB (wrong variant!) | 0.6857 |

**Lessons**

1. **Naive Bayes is weaker here.** The class depends on a *sum* of the features, and a model that treats features as independent evidence handles that less precisely than logistic regression. (Also, the true class boundary is not a clean two-Gaussians structure.)
2. **Choosing the right variant matters.** Multinomial NB assumes **count data** and does badly on continuous marks (0.69 vs 0.83).
3. Naive Bayes shines in **different** settings: text and very high-dimensional sparse data, tiny training sets, and when speed matters.

---

### 15. Text classification with Naive Bayes

The classic NB application is **spam filtering**. We turn text into **word-count vectors** and use Multinomial NB.

**Training texts** (label 1 = spam, 0 = normal)

| Text | Label |
| --- | --- |
| win free money now | 1 |
| free money offer win | 1 |
| win prize free | 1 |
| meeting report today | 0 |
| project report meeting | 0 |
| lunch meeting today | 0 |

```python
from pyspark.ml.feature import Tokenizer, CountVectorizer

docs = [("win free money now", 1), ("free money offer win", 1), ("win prize free", 1),
        ("meeting report today", 0), ("project report meeting", 0), ("lunch meeting today", 0)]

tk = Tokenizer(inputCol="text", outputCol="words")
ddf = tk.transform(spark.createDataFrame(docs, ["text", "label"]))

cvm = CountVectorizer(inputCol="words", outputCol="features", vocabSize=20).fit(ddf)
print(cvm.vocabulary)

cdf = cvm.transform(ddf)
tnb = NaiveBayes(smoothing=1.0, modelType="multinomial").fit(cdf)
```

```text
['free', 'meeting', 'win', 'today', 'report', 'money', 'project', 'offer', 'lunch', 'prize', 'now']
```

| Step | What it does |
| --- | --- |
| `Tokenizer` | Splits text into lowercase words |
| `CountVectorizer` | Builds the vocabulary and turns each text into a vector of **word counts** |
| `NaiveBayes(multinomial)` | Learns word frequencies per class |

**Classify new messages**

```python
new = spark.createDataFrame([("free money meeting",), ("report today",), ("win free prize",)], ["text"])
newd = cvm.transform(tk.transform(new))
tnb.transform(newd).select("text", "probability", "prediction").show(truncate=False)
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

**Hand verification of "free money meeting"**

Word counts per class (vocabulary size V = 11):

```text
 SPAM words (total 11): win 3, free 3, money 2, now 1, offer 1, prize 1
 HAM  words (total  9): meeting 3, report 2, today 2, project 1, lunch 1
```

With Laplace smoothing α = 1, `P(word | class) = (count + 1) / (total words in class + V)`:

```text
 SPAM:  denominator = 11 + 11 = 22
        P(free|spam) = (3+1)/22 = 4/22     P(money|spam) = (2+1)/22 = 3/22     P(meeting|spam) = (0+1)/22 = 1/22
        Score(spam) = 0.5 × (4/22)(3/22)(1/22) = 0.000563

 HAM:   denominator = 9 + 11 = 20
        P(free|ham) = 1/20     P(money|ham) = 1/20     P(meeting|ham) = (3+1)/20 = 4/20
        Score(ham)  = 0.5 × (1/20)(1/20)(4/20) = 0.000250

 P(spam | text) = 0.000563 / (0.000563 + 0.000250) = 0.6927
```

✔ **Matches Spark's 0.6927** exactly. "free money" are strong spam words, but "meeting" is a strong ham word, so the verdict is spam but not overwhelming.

**In practice:** use many more documents; optionally weight words with **TF-IDF** (`HashingTF` + `IDF`) before Naive Bayes; and put `Tokenizer → CountVectorizer → NaiveBayes` into a `Pipeline`.

---

### 16. Important parameters

| Parameter | Default | Meaning |
| --- | --- | --- |
| `modelType` | `multinomial` | `multinomial`, `bernoulli`, `gaussian`, `complement` |
| `smoothing` | 1.0 | Laplace/Lidstone smoothing α (Multinomial, Bernoulli, Complement) |
| `thresholds` | none | Per-class thresholds |
| `weightCol` | none | Row weights |
| `featuresCol`, `labelCol`, `predictionCol`, `probabilityCol`, `rawPredictionCol` | standard | Column names |

**Model attributes:** `pi` (log priors), `theta` (log conditional probabilities, or means for Gaussian), `sigma` (variances, Gaussian only), `numClasses`, `numFeatures`.

---

### 17. Strengths and weaknesses

| Strengths | Weaknesses |
| --- | --- |
| **Very fast** to train and predict (just counting) | The **independence assumption** is rarely true |
| Works with **little training data** | Probabilities can be poorly calibrated (over-confident) |
| Excellent for **text** and high-dimensional data | Multinomial/Bernoulli need **non-negative** features |
| Handles **multiclass** naturally | Cannot learn feature **interactions** |
| Easy to understand and implement | Zero-frequency issue without smoothing |
| Scales well; robust to irrelevant features | Often beaten by logistic regression/SVM on dense numeric data |

---

### 18. Common errors

| Symptom | Cause | Fix |
| --- | --- | --- |
| `Vector values MUST NOT be Negative, NaN or Infinity, but got [-1.0]` | Negative feature for Multinomial/Bernoulli NB | Scale to [0, 1] (`MinMaxScaler`), or use `modelType="gaussian"` |
| Terrible accuracy on continuous data | Used Multinomial | Use **Gaussian** |
| Probability 0 for some class | Zero-frequency, `smoothing=0` | Keep `smoothing=1.0` |
| Categorical text passed directly | Needs numbers | `StringIndexer` + `OneHotEncoder(dropLast=False)` |
| `dropLast=True` for NB one-hot | Loses a category's feature | Use `dropLast=False` |
| Standardized features (negative values) with Multinomial | StandardScaler creates negatives | Do not standardize for Multinomial; use MinMax or Gaussian |
| Bernoulli with counts | Bernoulli needs 0/1 | Binarize (`Binarizer`) or use Multinomial |
| Misreading `theta` | It holds **log** probabilities | `math.exp(theta)` |
| Over-trusting probabilities | NB probabilities are over-confident | Use them for ranking, not as exact chances |
| Result changes with correlated duplicate features | Violates independence, double counts | Remove duplicate features |

---

### 19. Phase 10 — practical scenarios

1. Use **Bayes' theorem** for the disease-test example; confirm 16.7%.
2. Build the **frequency tables** for the weather data.
3. Compute **Score(Yes) and Score(No)** for (Sunny, Cool, High, Strong) by hand; normalize.
4. Compute the same with **Laplace smoothing** and compare (0.2799).
5. Show how **Overcast with class No** creates the zero-frequency problem and fix it.
6. Run the weather **Pipeline** in Spark with `smoothing=0`; confirm 0.2046 / 0.7954.
7. Print `pi` and `theta`; **exponentiate** and interpret.
8. Fit **Gaussian NB** on the flower sample; print the class means.
9. Compute the **Gaussian density by hand** for petal length 5.0 and one class.
10. Train **Gaussian vs Multinomial** NB on the students; explain the gap.
11. Build the **spam classifier** with `Tokenizer → CountVectorizer → NaiveBayes`.
12. Verify "free money meeting" by hand (0.6927).

---

### 20. Phase 10 — viva questions and answers

#### Q1. What is Naive Bayes?

A probabilistic classifier based on Bayes' theorem with the assumption that features are conditionally independent given the class.

#### Q2. State Bayes' theorem.

`P(C | X) = P(X | C) · P(C) / P(X)`.

#### Q3. Define prior, likelihood, evidence and posterior.

Prior: `P(C)`, class frequency before observing features; likelihood: `P(X | C)`; evidence: `P(X)`; posterior: `P(C | X)`, the probability of the class after observing features.

#### Q4. Why is it called "naive"?

Because it assumes all features are independent given the class, which is rarely exactly true.

#### Q5. What does the independence assumption allow?

The joint likelihood factorizes into a product of simple per-feature probabilities, so training is just counting.

#### Q6. What is the prediction rule?

Choose the class that maximizes `P(C) · Π P(xᵢ | C)`.

#### Q7. Why can we ignore the evidence `P(X)`?

It is the same for every class, so it does not change which class has the highest posterior.

#### Q8. What is the zero-frequency problem?

A feature value never seen with a class gives probability zero and wipes out the entire product.

#### Q9. What is Laplace smoothing?

Adding a small count α (usually 1) to every count and α·k to the denominator so no probability is zero.

#### Q10. Why use log probabilities?

Multiplying many small probabilities underflows to zero; adding logs is numerically stable and keeps the same ranking.

#### Q11. Name the Naive Bayes variants in Spark and when to use each.

Multinomial (counts, text), Bernoulli (binary features), Gaussian (continuous features), Complement (imbalanced text).

#### Q12. Why does Multinomial NB require non-negative features?

Its likelihood is based on counts/frequencies, which cannot be negative; Spark rejects negative values.

#### Q13. How does Gaussian NB model a feature?

As a normal distribution with a separate mean and variance for each class.

#### Q14. What do `pi` and `theta` store?

`pi`: log class priors; `theta`: log conditional probabilities (or class means for Gaussian).

#### Q15. In the weather example, what did the classifier predict and with what probability?

For (Sunny, Cool, High, Strong) it predicts **No** with probability 0.7954.

#### Q16. How do you apply Naive Bayes to text in Spark?

`Tokenizer → CountVectorizer (or HashingTF+IDF) → NaiveBayes(modelType="multinomial")`, ideally in a Pipeline.

#### Q17. What are the strengths of Naive Bayes?

Very fast, needs little data, works well for text and high dimensions, handles multiclass.

#### Q18. What are its weaknesses?

The independence assumption, poorly calibrated probabilities, and inability to model feature interactions.

#### Q19. Why did Naive Bayes score lower than logistic regression on our students?

The class depends on a combination (sum) of features and the data is dense and numeric, which suits a discriminative linear model better than independent Gaussian evidence.

#### Q20. Why do we use `OneHotEncoder(dropLast=False)` for Naive Bayes on categorical data?

Each category needs its own feature so it can have its own likelihood; dropping one would lose information.

---

### 21. Phase 10 — final master recap

```text
NAIVE BAYES   probabilistic classifier ; picks the class with the highest posterior
BAYES         P(C|X) = P(X|C)·P(C) / P(X)         (posterior = likelihood × prior / evidence)
NAIVE         features independent given class :  P(X|C) = Π P(xᵢ|C)
RULE          class = argmax  P(C) · Π P(xᵢ|C)      (drop P(X); normalize to get probabilities)
SMOOTHING     P = (count + α) / (N_class + α·k)     (Spark smoothing=1.0 default) → fixes zero frequency
LOG FORM      log P(C) + Σ log P(xᵢ|C)             (Spark: pi, theta)
VARIANTS      multinomial (counts/text) · bernoulli (0/1) · gaussian (continuous) · complement
WEATHER       Score Yes 0.00529 , No 0.02057 → P(Yes)=0.2046 , P(No)=0.7954 → predict NO  (Spark identical)
TEXT          "free money meeting" → spam 0.6927 (hand = Spark)
STUDENTS      Gaussian 0.8286 ; Multinomial 0.6857 (wrong variant) ; Logistic 0.9143
PYSPARK       NaiveBayes(modelType, smoothing).fit(train) ; model.pi ; model.theta ; model.sigma
```

**Seven facts to memorise**

1. Posterior ∝ **prior × likelihood**; the evidence cancels.
2. "Naive" = **conditional independence** of features given the class.
3. Training is **counting**, so it is extremely fast.
4. **Laplace smoothing** fixes zero probabilities.
5. **Multinomial** = counts (text), **Bernoulli** = 0/1, **Gaussian** = continuous.
6. Multinomial/Bernoulli **reject negative features**.
7. Spark's `pi` and `theta` are **log** values.

### Phase 10 completed

---

> **Next → Phase 11: Model Evaluation and Performance Metrics** (the confusion matrix with TP/FP/TN/FN, accuracy, precision, recall, F1, specificity, ROC and AUC, threshold trade-offs; regression metrics MSE, RMSE, MAE, R² with hand calculations; clustering metrics; cross-validation and hyperparameter tuning; confusion-matrix heat map). File: `11_Unit6_Model_Evaluation_Metrics.md`
