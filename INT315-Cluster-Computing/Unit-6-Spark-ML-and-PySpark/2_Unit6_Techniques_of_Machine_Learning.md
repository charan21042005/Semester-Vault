# Unit VI — Phase 2: Techniques of Machine Learning

> **Continuing from Phase 1.** We know *what* ML is. Now we learn the **families of ML techniques**, how to pick the right one for a problem, and the three ideas that decide whether a model is any good: **train/test split, overfitting and bias-variance**. This covers the syllabus topic *Techniques of Machine Learning*.
>
> All code and outputs here were run for real (PySpark 3.5 and numpy). The experiments prove the ideas with actual numbers, so you will *see* overfitting happen.

---

## Phase 2 — Roadmap

| # | Topic |
| --- | --- |
| 1 | The big picture: types of machine learning |
| 2 | Supervised learning |
| 3 | Regression vs classification |
| 4 | Unsupervised learning |
| 5 | Semi-supervised and reinforcement learning |
| 6 | Choosing a technique: the decision table |
| 7 | Which algorithms Spark MLlib provides |
| 8 | Our practice dataset (used in the whole unit) |
| 9 | Train/test split |
| 10 | Overfitting and underfitting |
| 11 | An overfitting experiment in numpy |
| 12 | An overfitting experiment in Spark (decision tree depth) |
| 13 | The bias-variance trade-off |
| 14 | Cures for overfitting |
| 15 | Cross-validation |
| 16 | Other practical traps: leakage, imbalance, scaling |
| 17 | Practical scenarios |
| 18 | Viva questions |
| 19 | Master revision |

---

### 1. The big picture: types of machine learning

```text
                         MACHINE LEARNING
        ┌──────────────┬────────┴────────┬───────────────────┐
   SUPERVISED     UNSUPERVISED     SEMI-SUPERVISED     REINFORCEMENT
   (labelled      (no labels,      (few labels +        (learn by trial,
    data)          find structure)  many unlabelled)    reward and penalty)
     │                │
 ┌───┴────┐      ┌────┴──────────────┐
 Regression  Classification   Clustering   Dimensionality   Association
 (number)    (category)       (groups)     reduction        rules
```

The deciding question is always: **do we have the correct answers (labels) for our training data?**

| Do we have labels? | Technique |
| --- | --- |
| Yes, for all data | **Supervised** |
| No labels at all | **Unsupervised** |
| Labels for a small part | **Semi-supervised** |
| No dataset; an agent learns by acting | **Reinforcement** |

---

### 2. Supervised learning

> **Supervised learning learns a mapping from inputs (features) to outputs (labels) using examples where the correct answer is already known.**

The word *supervised* means a "teacher" (the labels) supervises the learning.

```text
TRAINING                                       PREDICTION
 hours attendance prev │ final_marks (label)    new student: 6, 80, 60
  6.8    51.3     50.1 │ 63.4                          │
  7.1    94.6     39.8 │ 75.8      ──learn──▶ MODEL ───┘──▶ predicted marks = 70.5
  ...                                
```

**Steps:** show the algorithm many (input, correct answer) pairs → it adjusts its parameters to reduce its mistakes → it can now answer for inputs it has never seen.

**Examples**

| Task | Features | Label |
| --- | --- | --- |
| Predict house price | area, rooms, age | price |
| Spam filter | words in the email | spam / not spam |
| Predict exam result | hours, attendance | pass / fail |
| Disease diagnosis | test results | disease / healthy |

---

### 3. Regression vs classification

Supervised learning has **two** problem types, decided by the **type of the label**.

| | **Regression** | **Classification** |
| --- | --- | --- |
| Label type | A **continuous number** | A **category (class)** |
| Question | *How much? How many?* | *Which one?* |
| Example | Predict marks = 70.5 | Predict pass or fail |
| Output | Any real number | One of a fixed set of classes |
| Spark algorithms in this unit | **Linear Regression**, Decision Tree Regressor | **Logistic Regression, Decision Tree, SVM, Naive Bayes** |
| Typical metrics | **RMSE, MAE, R²** | **Accuracy, precision, recall, F1, confusion matrix** |

**Classification sub-types**

| Type | Classes | Example |
| --- | --- | --- |
| **Binary** | 2 | pass / fail, spam / not spam |
| **Multiclass** | 3 or more | iris species: setosa / versicolor / virginica |
| Multi-label | Several labels at once | A photo tagged *beach* and *sunset* |

> **Trap:** logistic **regression** is a **classification** algorithm, despite the name. It *regresses a probability* and then turns it into a class.

---

### 4. Unsupervised learning

> **Unsupervised learning finds hidden structure in data that has no labels.**

There is no "correct answer" to learn from; the algorithm discovers patterns on its own.

| Technique | Goal | Example | Algorithm |
| --- | --- | --- | --- |
| **Clustering** | Group similar records | Segment customers into groups | **K-means** (Phase 8) |
| **Dimensionality reduction** | Compress many features into few | 100 features → 2 for plotting | PCA |
| **Association rules** | Find items that occur together | "People who buy bread also buy butter" | FP-Growth |
| **Anomaly detection** | Find unusual records | Fraudulent transactions | Distance/density methods |

```text
Before: points with no colours        After K-means: groups discovered
      · ·    ·  ·                           A A    B  B
     ·  ·  ·  ·  ·                         A  A  B  B  B
       ·   ·     ·                           A   C     C
```

Clustering has **no accuracy score against labels**, because there are none. We judge it with measures such as WSSSE and silhouette (Phase 8).

---

### 5. Semi-supervised and reinforcement learning

**Semi-supervised learning:** labelling data is expensive (a doctor must label every X-ray). We label a **small portion** and let the algorithm use the large unlabelled portion to improve.

**Reinforcement learning (RL):** an **agent** takes **actions** in an **environment** and receives **rewards or penalties**. It learns a strategy (policy) that maximizes total reward.

```text
        action
 Agent ─────────▶ Environment
   ▲                  │
   └── reward, state ─┘
```

Examples: game-playing programs, robot walking, recommendation strategies. (Not part of Spark MLlib; know the definition for theory questions.)

---

### 6. Choosing a technique: the decision table

Read the question and look at **what you want to predict**.

| Your goal | Labels? | Technique | Algorithm (Unit VI) | Main metric |
| --- | --- | --- | --- | --- |
| Predict a **number** | Yes | Regression | Linear Regression | RMSE, MAE, R² |
| Predict a **yes/no** | Yes | Binary classification | Logistic Regression, SVM, Naive Bayes, Decision Tree | Accuracy, F1, AUC |
| Predict one of **many classes** | Yes | Multiclass classification | Logistic (multinomial), Decision Tree, Naive Bayes, SVM (One-vs-Rest) | Accuracy, F1 |
| **Group** similar records | No | Clustering | K-means | Silhouette, WSSSE |
| Explain a prediction as **rules** | Yes | Interpretable classification | Decision Tree | Accuracy |
| **Text** (spam, sentiment) | Yes | Classification | Naive Bayes | Accuracy, F1 |
| Find **important features** | Yes | Any | Trees (feature importance), Linear (coefficients) | – |

---

### 7. Which algorithms Spark MLlib provides

| Area | Algorithms |
| --- | --- |
| **Regression** | Linear Regression, Generalized Linear Regression, Decision Tree, Random Forest, Gradient-Boosted Trees, Isotonic |
| **Classification** | Logistic Regression, Decision Tree, Random Forest, Gradient-Boosted Trees, Linear SVC, Naive Bayes, Multilayer Perceptron, One-vs-Rest |
| **Clustering** | K-means, Bisecting K-means, Gaussian Mixture, LDA |
| **Recommendation** | ALS (collaborative filtering) |
| **Frequent patterns** | FP-Growth, PrefixSpan |
| **Dimensionality reduction** | PCA, SVD |
| **Feature tools** | VectorAssembler, StringIndexer, OneHotEncoder, scalers, Tokenizer, TF-IDF, Word2Vec |

Your syllabus selects six: **Linear Regression, Logistic Regression, Decision Tree, K-means, SVM, Naive Bayes.** We study each in depth.

---

### 8. Our practice dataset (used in the whole unit)

We need data we can reproduce exactly. This small generator makes **200 students** with a known relationship plus noise. Save it as `C:\sparklabs\make_data.py`:

```python
import random

def make_students(n=200, seed=42):
    random.seed(seed)
    rows = []
    for i in range(n):
        hours = round(random.uniform(1, 10), 1)           # study hours per day
        attendance = round(random.uniform(50, 100), 1)    # attendance percent
        prev = round(random.uniform(35, 90), 1)           # previous exam marks
        noise = random.gauss(0, 3)                        # random variation
        final = 4 * hours + 0.25 * attendance + 0.35 * prev + 5 + noise
        final = round(min(100.0, max(0.0, final)), 1)     # keep marks in 0..100
        passed = 1 if final >= 68 else 0                  # classification label
        rows.append((i + 1, hours, attendance, prev, final, passed))
    return rows
```

**The hidden "true" rule** (which our models must rediscover):

```text
final_marks ≈ 4 × hours + 0.25 × attendance + 0.35 × prev_marks + 5      (± noise)
passed      = 1 if final_marks ≥ 68 else 0
```

**Load it in PySpark** (start the shell in `C:\sparklabs` or add the folder to the path):

```python
import sys
sys.path.append(r"C:\sparklabs")
from make_data import make_students

df = spark.createDataFrame(make_students(),
         ["id", "hours", "attendance", "prev_marks", "final_marks", "passed"])
df.show(5)
print(df.count())
df.groupBy("passed").count().orderBy("passed").show()
```

```text
+---+-----+----------+----------+-----------+------+
| id|hours|attendance|prev_marks|final_marks|passed|
+---+-----+----------+----------+-----------+------+
|  1|  6.8|      51.3|      50.1|       63.4|     0|
|  2|  7.1|      94.6|      39.8|       75.8|     1|
|  3|  4.8|      51.5|      47.0|       52.8|     0|
|  4|  2.8|      82.5|      65.0|       59.6|     0|
|  5|  3.0|      79.5|      79.5|       70.1|     1|
+---+-----+----------+----------+-----------+------+
only showing top 5 rows

200

+------+-----+
|passed|count|
+------+-----+
|     0|  100|
|     1|  100|
+------+-----+
```

The classes are perfectly balanced: 100 fail, 100 pass.

```python
df.describe("hours", "attendance", "prev_marks", "final_marks").show()
```

```text
+-------+-----------------+------------------+------------------+------------------+
|summary|            hours|        attendance|        prev_marks|       final_marks|
+-------+-----------------+------------------+------------------+------------------+
|  count|              200|               200|               200|               200|
|   mean|5.473000000000002|            75.732|62.41849999999999|           67.9515|
| stddev|2.629399113579462|14.974278650590405|15.088536290407188|13.041341213742792|
|    min|              1.0|              50.3|              35.0|              35.9|
|    max|             10.0|             100.0|              89.8|              98.2|
+-------+-----------------+------------------+------------------+------------------+
```

**Why synthetic data?** Real datasets would need downloads. This one is tiny, reproducible on every machine (same `seed=42`) and has a **known truth**, so we can check whether our algorithms recovered the right coefficients (4, 0.25, 0.35, intercept 5).

---

### 9. Train/test split

**The central problem:** a model can *memorise* the training data. A student who memorises last year's paper scores 100 on it, but may fail a new paper. To measure real ability, we must test on data **the model never saw**.

```text
ALL DATA (200)
 ├── TRAINING SET (80% = 165)  → the model learns from this
 └── TEST SET     (20% = 35)   → used ONLY once, to measure real performance
```

**In PySpark**

```python
train, test = df.randomSplit([0.8, 0.2], seed=42)
print(train.count(), test.count())
```

```text
165 35
```

**Reading it**

| Part | Meaning |
| --- | --- |
| `randomSplit([0.8, 0.2])` | Split rows randomly in an 80:20 ratio |
| `seed=42` | Makes the split **repeatable**. Same seed gives the same split |
| `165 35` | The counts are *approximately* 80:20 (each row is assigned randomly, so not exactly 160/40) |

**Golden rules**

1. **Never** train on the test set.
2. Do all learning (including scaling statistics and feature choices) **using only the training set**.
3. Look at the test set **once**, at the end.
4. Common ratios: 80:20 or 70:30. Very large data can use 90:10.

---

### 10. Overfitting and underfitting

| | **Underfitting** | **Good fit** | **Overfitting** |
| --- | --- | --- | --- |
| Model is… | Too simple | Just right | Too complex |
| Training error | High | Low | **Very low** |
| Test error | High | Low | **High** |
| Meaning | Misses the real pattern | Captures the pattern | Memorises noise too |
| Analogy | Studied too little | Understood the subject | Memorised answers without understanding |

```text
 y                                         y                                    y
 │   · ·  ·                                │   · ·  ·                           │   · ·  ·
 │  ·       · ·     ─────── straight line  │  ·       · ·   smooth curve         │  ·       ·· ╲╱╲╱  wiggly curve
 │ ·    ·     ·       (misses the curve)   │ ·    ·     ·    (follows trend)      │ ·    ·    ╱ ·    ╲  through every point
 └──────────────── x                       └──────────────── x                    └──────────────── x
     UNDERFITTING                               GOOD FIT                              OVERFITTING
```

**How to detect it: compare training and test performance**

| Training score | Test score | Diagnosis |
| --- | --- | --- |
| Poor | Poor | **Underfitting** |
| Excellent | Much worse | **Overfitting** |
| Good | Similarly good | **Good fit** |

---

### 11. An overfitting experiment in numpy

We fit polynomials of increasing complexity to data that is *truly a straight line plus noise*.

```python
import numpy as np

rng = np.random.default_rng(0)
xs = np.linspace(0, 1, 10);      ys = 2 * xs + 1 + rng.normal(0, 0.2, 10)     # training data
xt = np.linspace(0.05, 0.95, 10); yt = 2 * xt + 1 + rng.normal(0, 0.2, 10)    # NEW data

for deg in (1, 3, 9):
    coef = np.polyfit(xs, ys, deg)                 # fit a polynomial of this degree
    train_mse = np.mean((np.polyval(coef, xs) - ys) ** 2)
    test_mse  = np.mean((np.polyval(coef, xt) - yt) ** 2)
    print("degree", deg, "train MSE", round(train_mse, 4), "test MSE", round(test_mse, 4))
```

**Output**

```text
degree 1 train MSE 0.0206 test MSE 0.049
degree 3 train MSE 0.0095 test MSE 0.0633
degree 9 train MSE 0.0    test MSE 0.1024
```

**What the numbers say**

| Degree | Train MSE | Test MSE | Verdict |
| --- | --- | --- | --- |
| 1 (a line = the truth) | 0.0206 | **0.0490** | Best on new data |
| 3 | 0.0095 | 0.0633 | Fits training better, new data worse |
| 9 | **0.0000** | **0.1024** | Memorised all 10 points (zero error!), but twice as bad on new data |

**More complexity always lowers *training* error but eventually raises *test* error.** That is overfitting, and it is why we judge models on test data.

---

### 12. An overfitting experiment in Spark (decision tree depth)

A decision tree's `maxDepth` controls its complexity. We train trees of growing depth on our students (predicting `passed`) and compare accuracy on train vs test data. (Decision trees are fully explained in Phase 7; here we only look at the pattern.)

```python
from pyspark.ml.feature import VectorAssembler
from pyspark.ml.classification import DecisionTreeClassifier
from pyspark.ml.evaluation import MulticlassClassificationEvaluator
from pyspark.sql import functions as F

va = VectorAssembler(inputCols=["hours", "attendance", "prev_marks"], outputCol="features")
data = va.transform(df).select("features", F.col("passed").alias("label"))
train, test = data.randomSplit([0.8, 0.2], seed=42)

evaluator = MulticlassClassificationEvaluator(labelCol="label", metricName="accuracy")

for depth in (1, 2, 3, 5, 8, 12):
    model = DecisionTreeClassifier(featuresCol="features", labelCol="label", maxDepth=depth, seed=42).fit(train)
    tr_acc = evaluator.evaluate(model.transform(train))
    te_acc = evaluator.evaluate(model.transform(test))
    print("maxDepth", depth, "train", round(tr_acc, 4), "test", round(te_acc, 4))
```

**Output**

```text
maxDepth 1 train 0.8545 test 0.7429
maxDepth 2 train 0.8606 test 0.7143
maxDepth 3 train 0.9333 test 0.8857
maxDepth 5 train 0.9879 test 0.8857
maxDepth 8 train 1.0 test 0.8286
maxDepth 12 train 1.0 test 0.8286
```

**Reading the table**

```text
maxDepth    train   test
   1        0.855   0.743   ← underfitting (too simple; both low)
   2        0.861   0.714
   3        0.933   0.886   ← sweet spot begins
   5        0.988   0.886
   8        1.000   0.829   ← overfitting: perfect on training, worse on test
  12        1.000   0.829
```

- Training accuracy **only goes up** with depth (it reaches a perfect 100%).
- Test accuracy rises, **peaks around depth 3 to 5**, then falls.
- A gap between train (1.000) and test (0.829) is the **signature of overfitting**.

---

### 13. The bias-variance trade-off

The expected error of a model on new data splits into three parts:

```text
Total error  =  Bias²  +  Variance  +  Irreducible noise
```

| Term | Meaning | Caused by |
| --- | --- | --- |
| **Bias** | Error from **wrong assumptions**; the model is too simple to capture the truth | Underfitting |
| **Variance** | Error from **sensitivity to the training sample**; the model changes a lot if the data changes | Overfitting |
| **Noise** | Randomness in the data itself | Cannot be reduced |

```text
 Error
  │ ╲                              ╱  ← total test error
  │  ╲   bias² (falls)           ╱
  │   ╲__                      ╱
  │      ╲___       ________╱     variance (rises)
  │          ╲_____╱
  │       best complexity ↓
  └────────────────────────────── model complexity →
     simple (high bias)        complex (high variance)
```

| | **High bias** | **High variance** |
| --- | --- | --- |
| Model | Too simple (linear for curved data; shallow tree) | Too flexible (deep tree; high-degree polynomial) |
| Training error | High | Low |
| Test error | High | High |
| Fix | More features, a more complex model | More data, simpler model, regularization |

> **Viva line:** *The bias-variance trade-off is the balance between a model that is too simple (high bias, underfits) and too complex (high variance, overfits). We want the complexity that minimizes test error.*

---

### 14. Cures for overfitting

| Technique | How it helps | Where in Unit VI |
| --- | --- | --- |
| **More training data** | Harder to memorise | – |
| **Simpler model** | Fewer parameters to memorise noise | Lower `maxDepth`, fewer features |
| **Regularization** | Penalises large coefficients | `regParam` in Linear/Logistic/SVM (Phases 5, 6, 9) |
| **Pruning / limits on trees** | Stops a tree growing too deep | `maxDepth`, `minInstancesPerNode` (Phase 7) |
| **Feature selection** | Removes useless features | Correlation analysis (Phase 4) |
| **Cross-validation** | Measures real performance reliably | Section 15, Phase 11 |
| **Ensembles** | Averaging many models lowers variance | Random Forest |

---

### 15. Cross-validation

One train/test split can be lucky or unlucky. **K-fold cross-validation** uses *every* row for testing exactly once.

**3-fold example**

```text
Data split into 3 equal parts (folds):  [ F1 | F2 | F3 ]

Round 1: train on F2+F3, test on F1  →  score 1
Round 2: train on F1+F3, test on F2  →  score 2
Round 3: train on F1+F2, test on F3  →  score 3

Final score = average of score 1, 2, 3
```

| Advantage | Detail |
| --- | --- |
| More reliable estimate | Averages over several splits |
| Uses all data for both training and testing | Important for small datasets |
| Basis of tuning | Compare hyperparameters by their average score |

Typical choice: **k = 5 or 10** (we use 3 in small demos). Spark's `CrossValidator` does this automatically (Phase 3 and Phase 11).

---

### 16. Other practical traps

#### 16.1 Data leakage

> **Leakage means information from the test set (or from the future) accidentally influences training**, making results look better than reality.

Examples:

- Scaling the *whole* dataset (using test-set statistics) before splitting.
- Including a feature that is a **copy of the label** (for example `final_marks` as a feature when predicting `passed`).
- Using a feature that is only known **after** the event.

> In our dataset, never put `final_marks` into the features when predicting `passed`. It *defines* `passed`, so the model would be 100% accurate and 100% useless.

#### 16.2 Class imbalance

If 95% of transactions are genuine, a model that always says "genuine" has **95% accuracy** and catches **zero** fraud. Accuracy misleads; use **precision, recall, F1, AUC** (Phase 11).

#### 16.3 Feature scaling

Features on very different scales (income in lakhs vs age in years) can dominate distance- or gradient-based algorithms (K-means, SVM, regularized regression). Standardize them (Phase 3). Tree-based and Naive Bayes methods do not need it.

#### 16.4 Curse of dimensionality

With very many features, data becomes sparse and models overfit more easily. Remove irrelevant features or reduce dimensions.

---

### 17. Phase 2 — practical scenarios

1. Save `make_data.py` and **load the 200 students** into a DataFrame; check the class balance.
2. **Split** the data 80:20 with `seed=42`; confirm 165 / 35.
3. Split again with a **different seed**; compare the counts.
4. Run the **numpy polynomial experiment** and explain why degree 9 has zero training error.
5. Run the **decision-tree depth experiment** and mark the underfitting and overfitting zones.
6. **Classify** each task as regression, classification or clustering: *predict rainfall in mm; detect spam; group shoppers; predict loan default; predict a taxi fare*.
7. For each task in 6, **name the algorithm** you would try.
8. Explain why using `final_marks` as a feature to predict `passed` is **leakage**.
9. A model has 99% training and 70% test accuracy. **Diagnose** it and list three fixes.
10. Draw the **3-fold cross-validation** diagram.

---

### 18. Phase 2 — viva questions and answers

#### Q1. What are the main types of machine learning?

Supervised, unsupervised, semi-supervised and reinforcement learning.

#### Q2. What is supervised learning?

Learning a mapping from features to labels using examples with known correct answers.

#### Q3. What is the difference between regression and classification?

Regression predicts a continuous number; classification predicts a category.

#### Q4. What is unsupervised learning? Give two techniques.

Finding hidden structure in unlabelled data, for example clustering (K-means) and dimensionality reduction (PCA).

#### Q5. What is reinforcement learning?

An agent learns by taking actions in an environment and receiving rewards or penalties to maximize total reward.

#### Q6. Which algorithms from your syllabus are classification algorithms?

Logistic Regression, Decision Tree, SVM and Naive Bayes.

#### Q7. Which algorithm is a clustering algorithm in your syllabus?

K-means.

#### Q8. Why is logistic regression called a classification algorithm?

It models the probability of a class and outputs a class label, even though it uses a regression-style equation.

#### Q9. Why do we split data into training and test sets?

To measure performance on unseen data and avoid being fooled by memorisation.

#### Q10. What does `seed` do in `randomSplit`?

It makes the random split reproducible.

#### Q11. What is overfitting?

When a model fits the training data (including noise) so closely that it performs poorly on new data.

#### Q12. What is underfitting?

When a model is too simple to capture the pattern, performing poorly on both training and test data.

#### Q13. How do you detect overfitting?

Training performance is very high while test performance is much lower.

#### Q14. Explain the bias-variance trade-off.

Simple models have high bias (underfit); complex models have high variance (overfit). We choose a complexity that minimizes total test error.

#### Q15. Name four ways to reduce overfitting.

More data, a simpler model, regularization, cross-validation (and pruning for trees).

#### Q16. What is k-fold cross-validation?

Splitting the data into k folds; each fold serves once as the test set while the others train; the k scores are averaged.

#### Q17. What is data leakage?

Information from the test set or the future leaking into training, making results unrealistically good.

#### Q18. Why is accuracy misleading for imbalanced data?

A model predicting only the majority class can score high accuracy while never detecting the minority class.

#### Q19. Which algorithms need feature scaling?

Distance- and gradient-based ones such as K-means, SVM and regularized regression; trees and Naive Bayes do not.

#### Q20. In our experiment, what did a deep decision tree show?

100% training accuracy but lower test accuracy than a moderate-depth tree, a classic sign of overfitting.

---

### 19. Phase 2 — final master recap

```text
TECHNIQUES
  Supervised    labels known     → Regression (number) | Classification (category)
  Unsupervised  no labels        → Clustering | Dimensionality reduction | Association
  Semi-supervised few labels
  Reinforcement agent + reward

SPLIT       train 80% / test 20%  (randomSplit([0.8,0.2], seed=42))
FIT         underfit (high bias) ←→ good ←→ overfit (high variance)
DETECT      compare train score vs test score
CURE        more data · simpler model · regularization · pruning · cross-validation
TRAPS       leakage · class imbalance · unscaled features
```

**Seven facts to memorise**

1. Labels present → supervised; absent → unsupervised.
2. Number → regression; category → classification.
3. Logistic regression is classification.
4. Never train on the test set.
5. Training error always improves with complexity; **test** error decides.
6. Overfitting = low train error, high test error.
7. Cross-validation averages several train/test splits.

### Phase 2 completed

---

> **Next → Phase 3: Spark MLlib and Key Concepts of Spark ML** (MLlib vs `spark.ml`, DataFrames and Vectors, Transformer, Estimator, Pipeline, feature transformers, VectorAssembler, scaling, encoding, hyperparameter tuning, saving models). File: `3_Unit6_Spark_MLlib_Key_Concepts.md`
