# Unit VI — Phase 13: Final Revision

> **Final phase of Unit VI.** This is exam mode, not another teaching pass. It contains the syllabus coverage map, the one-page mental model, formula and code cheat sheets for **every algorithm and metric**, the "numbers to remember" table, a combined error table, **exactly 30 viva questions with answers**, and a last-10-minute sheet.
>
> **How the notes were verified:** all PySpark code in Unit VI was run in a real PySpark 3.5 session, every hand calculation was cross-checked against numpy and Spark, and the outputs shown are the real ones. Your Python 3.10.11 + Spark 3.5.9 uses the identical API.

---

## 1. Syllabus coverage map

Your Unit VI syllabus line:

> *Spark ML Programming and PySpark: Introduction to Machine Learning and PySpark, Techniques of Machine Learning, Introduction to Spark MLlib, Key Concepts of Spark ML, Spark ML Algorithms using PySpark (Linear Regression, Logistic Regression, Decision Tree, K-means, SVM, Naive Bayes), Evaluation and performance metrics (Confusion matrix, R², RMSE, MAE, Correlation heat map).*

| Syllabus topic | Where it is covered |
| --- | --- |
| Introduction to Machine Learning and PySpark | **Phase 1** |
| Techniques of Machine Learning | **Phase 2** (supervised, unsupervised, overfitting, bias-variance, CV) |
| Introduction to Spark MLlib | **Phase 3** |
| Key Concepts of Spark ML | **Phase 3** (Transformer, Estimator, Pipeline, parameters, tuning) |
| **Correlation heat map** | **Phase 4** (maths, Spark, seaborn) |
| **Linear Regression** | **Phase 5** (least squares, normal equation, gradient descent, regularization) |
| **Logistic Regression** | **Phase 6** (sigmoid, odds, log-loss, softmax) |
| **Decision Tree** | **Phase 7** (entropy, Gini, information gain, regression trees) |
| **K-means** | **Phase 8** (WCSS, hand iteration, elbow, silhouette) |
| **SVM** | **Phase 9** (margin, hinge loss, soft margin, OvR) |
| **Naive Bayes** | **Phase 10** (Bayes theorem, smoothing, variants, text) |
| **Confusion matrix, R², RMSE, MAE** | **Phase 11** (+ precision, recall, F1, ROC-AUC, CV) |
| Practical integration | **Phase 12** |
| Revision and viva | **Phase 13** (this file) |

**File index**

| File | Phase |
| --- | --- |
| `1_Unit6_ML_and_PySpark_Introduction.md` | 1 |
| `2_Unit6_Techniques_of_Machine_Learning.md` | 2 |
| `3_Unit6_Spark_MLlib_Key_Concepts.md` | 3 |
| `4_Unit6_Data_Understanding_and_Correlation_Heatmap.md` | 4 |
| `5_Unit6_Linear_Regression.md` | 5 |
| `6_Unit6_Logistic_Regression.md` | 6 |
| `7_Unit6_Decision_Tree.md` | 7 |
| `8_Unit6_KMeans_Clustering.md` | 8 |
| `9_Unit6_SVM.md` | 9 |
| `10_Unit6_Naive_Bayes.md` | 10 |
| `11_Unit6_Model_Evaluation_Metrics.md` | 11 |
| `12_Unit6_Complete_Practical_Integration.md` | 12 |
| `13_Unit6_Final_Revision_30_Viva.md` | 13 |

---

## 2. Your environment

| Tool | Version | Role |
| --- | --- | --- |
| Python | 3.10.11 | The language of Unit VI |
| Spark / PySpark | 3.5.9 | `pyspark.ml` |
| Java | 1.8.0_501 | Spark engine |
| numpy, pandas, matplotlib, seaborn | latest | Arrays, tables, **heat maps** |

```powershell
pip install numpy pandas matplotlib seaborn
pyspark                        REM start the shell ;  spark-submit script.py  for scripts
```

---

## 3. The Unit VI mental model

```text
 DATA ──▶ UNDERSTAND ──▶ PREPARE ──▶ SPLIT ──▶ PIPELINE(fit) ──▶ PREDICT ──▶ EVALUATE ──▶ TUNE ──▶ SAVE
         describe,         Imputer,    train/    prep stages +    transform     metrics     CV +      write().
         correlation       Indexer,    test      ALGORITHM                      on TEST     ParamGrid save()
         heat map          Assembler,
                           Scaler
 SUPERVISED:   label known  → REGRESSION (number): Linear Regression
                            → CLASSIFICATION (category): Logistic Regression · Decision Tree · SVM · Naive Bayes
 UNSUPERVISED: no label     → CLUSTERING: K-means
```

**One-line summary**

> *Machine learning learns a rule from data. Spark ML expresses the whole workflow as Pipelines of Transformers and Estimators on DataFrames, and PySpark lets us run it from Python at scale.*

---

## 4. Cheat sheets

### 4.1 Spark ML grammar

| Concept | Definition |
| --- | --- |
| **DataFrame** | The dataset |
| **Transformer** | `transform()` adds columns (VectorAssembler, Bucketizer, any **Model**) |
| **Estimator** | `fit()` learns → returns a **Model** (LinearRegression, StandardScaler, StringIndexer) |
| **Pipeline** | Ordered chain of stages; `fit()` → `PipelineModel` |
| **Evaluator** | Computes a metric from predictions |
| **ParamGrid + CrossValidator** | Hyperparameter tuning |

```python
va  = VectorAssembler(inputCols=[...], outputCol="features")          # features → one Vector column
model = Estimator(featuresCol="features", labelCol="label").fit(train)  # LEARN
pred  = model.transform(test)                                           # PREDICT
metric = Evaluator(labelCol="label", metricName="...").evaluate(pred)   # EVALUATE
```

| Feature tool | Use |
| --- | --- |
| `Imputer` | Fill missing values (mean/median/mode) |
| `StringIndexer` | Text category → index (most frequent = 0) |
| `OneHotEncoder` | Index → 0/1 vector (default drops last; `dropLast=False` for Naive Bayes) |
| `VectorAssembler` | Combine columns into `features` |
| `StandardScaler` | `z = (x − μ)/σ` |
| `MinMaxScaler` | `x' = (x − min)/(max − min)` |
| `Tokenizer`, `CountVectorizer` | Text → word counts |

### 4.2 Algorithm formulas

| Algorithm | Core formulas |
| --- | --- |
| **Linear Regression** | `ŷ = β₀ + β₁x₁ + … + βₖxₖ` ; loss `SSE = Σ(y − ŷ)²` ; simple: `β₁ = Sxy/Sxx`, `β₀ = ȳ − β₁x̄` ; normal equation `β = (XᵀX)⁻¹Xᵀy` ; gradient descent `β ← β − η·∂MSE/∂β` ; Ridge `SSE + λΣβ²` (L2), Lasso `SSE + λΣ|β|` (L1), `elasticNetParam` 0 = Ridge, 1 = Lasso |
| **Logistic Regression** | `z = β₀ + β·x` ; `p = σ(z) = 1/(1+e⁻ᶻ)` ; `ln(p/(1−p)) = z` ; odds ratio `e^β` ; log-loss `−(1/n)Σ[y ln p + (1−y) ln(1−p)]` ; gradient `(1/n)Σ(p − y)x` ; softmax `e^zₖ/Σe^zⱼ` |
| **Decision Tree** | Entropy `−Σ p log₂ p` ; Gini `1 − Σp²` ; Gain = `Impurity(parent) − Σ(\|child\|/\|parent\|)·Impurity(child)` ; regression: SSE reduction, leaf = mean |
| **K-means** | `d = √Σ(pᵢ−qᵢ)²` ; `WCSS = Σ_clusters Σ_points ‖x−μ‖²` ; repeat: assign to nearest centroid → centroid = mean ; silhouette `s = (b−a)/max(a,b)` |
| **SVM** | Hyperplane `w·x + b = 0` ; distance `|w·x+b|/‖w‖` ; support vectors on `w·x+b = ±1` ; margin `= 2/‖w‖` ; hard margin `min ½‖w‖²  s.t.  y(w·x+b) ≥ 1` ; soft margin `½‖w‖² + CΣξ` ; hinge `max(0, 1 − y·f)` |
| **Naive Bayes** | `P(C\|X) = P(X\|C)P(C)/P(X)` ; naive: `P(X\|C) = ΠP(xᵢ\|C)` ; rule `argmax P(C)ΠP(xᵢ\|C)` ; Laplace `(count+α)/(N+αk)` ; Gaussian density `(1/√(2πσ²))·exp(−(x−μ)²/2σ²)` |

### 4.3 Metric formulas

| Metric | Formula |
| --- | --- |
| Accuracy | `(TP+TN)/(TP+TN+FP+FN)` |
| Precision | `TP/(TP+FP)` |
| Recall (TPR) | `TP/(TP+FN)` |
| Specificity | `TN/(TN+FP)` |
| FPR | `FP/(FP+TN)` |
| F1 | `2PR/(P+R)` |
| AUC | P(score(positive) > score(negative)); 0.5 = random, 1 = perfect |
| MSE / RMSE | `Σ(y−ŷ)²/n` / `√MSE` |
| MAE | `Σ|y−ŷ|/n` |
| R² | `1 − SSE/SST` ; adjusted `1 − (1−R²)(n−1)/(n−k−1)` |
| Correlation | `r = Σ(x−x̄)(y−ȳ)/√[Σ(x−x̄)²Σ(y−ȳ)²]` ∈ [−1, 1] |
| Covariance | `Σ(x−x̄)(y−ȳ)/(n−1)` |

### 4.4 PySpark classes and key parameters

| Class | Module | Key parameters |
| --- | --- | --- |
| `LinearRegression` | `ml.regression` | `regParam`, `elasticNetParam`, `maxIter`, `solver` |
| `LogisticRegression` | `ml.classification` | `regParam`, `elasticNetParam`, `threshold`, `family`, `maxIter` |
| `DecisionTreeClassifier` / `Regressor` | `ml.classification` / `ml.regression` | `maxDepth`, `impurity`, `minInstancesPerNode`, `maxBins`, `seed` |
| `RandomForestClassifier` | `ml.classification` | `numTrees`, `maxDepth` |
| `KMeans` | `ml.clustering` | `k`, `maxIter`, `initMode`, `seed` |
| `LinearSVC` | `ml.classification` | `regParam`, `maxIter` (binary, linear only) |
| `OneVsRest` | `ml.classification` | `classifier=` (for multiclass SVM) |
| `NaiveBayes` | `ml.classification` | `modelType` (multinomial / bernoulli / gaussian / complement), `smoothing` |
| Evaluators | `ml.evaluation` | `RegressionEvaluator` (`rmse`,`mae`,`r2`), `BinaryClassificationEvaluator` (`areaUnderROC`), `MulticlassClassificationEvaluator` (`accuracy`,`f1`,…), `ClusteringEvaluator` (`silhouette`) |
| Tuning | `ml.tuning` | `ParamGridBuilder`, `CrossValidator(numFolds)`, `TrainValidationSplit(trainRatio)` |
| Correlation | `ml.stat` | `Correlation.corr(df, "vec", "pearson"/"spearman")` |

### 4.5 Heat maps

```python
import pandas as pd, matplotlib.pyplot as plt, seaborn as sns
from pyspark.ml.stat import Correlation

# correlation heat map
corr = pd.DataFrame(Correlation.corr(vec_df, "v").head()[0].toArray(), index=cols, columns=cols)
sns.heatmap(corr, annot=True, fmt=".2f", cmap="coolwarm", vmin=-1, vmax=1)

# confusion-matrix heat map
sns.heatmap([[33, 2], [1, 31]], annot=True, fmt="d", cmap="Blues",
            xticklabels=["Pred 0", "Pred 1"], yticklabels=["Actual 0", "Actual 1"])
plt.show()
```

---

## 5. Numbers to remember (all verified)

| Topic | Key numbers |
| --- | --- |
| **Correlation (5 points)** | cov = 21.25 ; r = 0.9948 |
| **Linear regression (5 points)** | `marks = 31.5 + 8.5·hours` ; SSE 7.5 ; MSE 1.5 ; RMSE 1.2247 ; MAE 1.0 ; R² 0.9897 ; first GD step: slope 7.52, intercept 2.28 |
| **Students: regression** | coef `[4.0612, 0.2479, 0.34]`, intercept 5.867 (truth 4, 0.25, 0.35, 5) ; test RMSE 2.8413, MAE 2.2595, R² 0.9291 |
| **Students: heat map** | r(hours, final) = 0.834 ; r(prev, final) = 0.419 ; r(attendance, final) = 0.318 ; features mutually ≈ 0 |
| **Logistic (8 students)** | w = 1.2823, b = −5.7703 ; boundary 4.5 h ; P(6 h) = 0.8725 ; odds ratio 3.605 ; start log-loss 0.6931 → 0.3129 |
| **Students: classification** | Logistic acc 0.9143, AUC 0.988 ; SVM 0.9143 (regParam 0.01) / 0.9429 (0.1) ; Tree depth 4 0.8857 ; Gaussian NB 0.8286 ; baseline 0.7143 |
| **Weather tree** | H(S) = 0.9403 ; gains: Outlook 0.2467, Humidity 0.1518, Wind 0.0481, Temp 0.0292 ; root = Outlook |
| **Regression tree (5 points)** | best split hours ≤ 3.5 ; leaves 48.33 and 70.0 |
| **K-means (7 points)** | WCSS 12.208 → **8.525** ; centroids (1.25, 1.5), (3.9, 5.1) ; silhouette **0.8315** |
| **K-means (customers)** | K = 4: WSSSE 271.8, silhouette 0.988 ; unscaled sizes [5,6,3,2] ✗ vs scaled [4,4,4,4] ✓ |
| **SVM (8 points)** | w = (0.62, 0.772), b = −3.1784, ‖w‖ = 0.9901, margin 2.0199 ; support vectors (3,3) and (1,2) |
| **Naive Bayes** | Weather query: P(Yes) = 0.2046, P(No) = 0.7954 → **No** ; spam "free money meeting" 0.6927 |
| **Confusion example** | TP 40, FP 10, FN 5, TN 45 → acc 0.85, prec 0.80, rec 0.8889, spec 0.8182, F1 0.8421 |
| **Students confusion** | [[33, 2], [1, 31]] → acc 0.9552, prec 0.9394, rec 0.9688, F1 0.9538, AUC 0.9866 |
| **AUC by pairs** | positives (0.9, 0.8, 0.4) vs negatives (0.7, 0.3, 0.2) → 8/9 = 0.8889 |
| **Softmax** | scores (2.0, 1.0, 0.1) → (0.659, 0.242, 0.099) |
| **Bayes (disease)** | prior 1%, sensitivity 99%, false positive 5% → P(ill \| +) = 16.7% |

---

## 6. Combined error table

| Symptom | Cause | Fix |
| --- | --- | --- |
| `Column features must be of type VectorUDT` | No `VectorAssembler` | Assemble features |
| `label does not exist` | Label not named `label` | `.alias("label")` / `labelCol=` |
| `Encountered null while assembling a row` | Missing values | `Imputer` / `na.drop()` / `handleInvalid` |
| `Unseen label` (StringIndexer) | New category | `handleInvalid="keep"` |
| `Labels MUST be in [0, ...)` | Negative/text label | `StringIndexer` |
| `Vector values MUST NOT be Negative` | Naive Bayes with negatives | MinMax scale or `gaussian` |
| Perfect accuracy | **Leakage** (label-derived feature) | Remove it |
| Train ≫ test | **Overfitting** | Regularize, `maxDepth`↓, more data |
| Low AUC | Used `prediction` not scores | `rawPrediction` / `probability` |
| K-means dominated by one feature | Unscaled | `StandardScaler` |
| Accuracy 99% but useless | Imbalance | Confusion matrix, recall, F1 |
| `f1` ≠ hand F1 | Spark `f1` is weighted | `fMeasureByLabel` |
| `LinearSVC` fails on 3 classes | Binary only | `OneVsRest` |
| Different results each run | No seeds | `seed=42` |
| Python version mismatch | Two Pythons | `PYSPARK_PYTHON` |
| Boolean filter fails | `and`/`or` | `&`, `|`, `~` with brackets |
| `seaborn` not found | Not installed | `pip install seaborn matplotlib` |
| Heat map invisible | No `plt.show()` | Add it or `savefig` |

---

## 7. Exactly 30 viva questions with answers

#### Q1. What is machine learning, and how does it differ from traditional programming?

Machine learning builds programs that learn patterns from data to make predictions without explicit rules. Traditional programming takes rules and data to produce answers; ML takes data and answers to produce the rules (the model).

#### Q2. What is PySpark and why is its ML performance close to Scala's?

PySpark is Spark's Python API. Python sends commands through Py4J to the Spark JVM, where DataFrame and MLlib operations execute, so the heavy work runs at JVM speed.

#### Q3. What are features, labels, parameters and hyperparameters?

Features are inputs; the label is the target to predict; parameters (coefficients) are learned from data; hyperparameters (`maxDepth`, `k`, `regParam`) are chosen before training.

#### Q4. What are the types of machine learning? Explain regression vs classification.

Supervised (labels known), unsupervised (no labels), semi-supervised and reinforcement learning. Within supervised learning, regression predicts a continuous number and classification predicts a category.

#### Q5. What are overfitting, underfitting and the bias-variance trade-off?

Underfitting: too simple, high bias, poor on train and test. Overfitting: too complex, high variance, excellent on train but poor on test. We choose complexity that minimizes test error, using regularization, simpler models, more data or cross-validation.

#### Q6. Why split data into train and test sets, and what is data leakage?

To measure performance on unseen data. Leakage is information from the test set or the label leaking into training (for example a feature derived from the label), giving unrealistically good results.

#### Q7. What is Spark MLlib, and how do `spark.ml` and `spark.mllib` differ?

MLlib is Spark's scalable ML library. `spark.ml` is the DataFrame-based primary API with Pipelines; `spark.mllib` is the older RDD-based API in maintenance mode.

#### Q8. Explain Transformer, Estimator and Pipeline.

A Transformer has `transform()` and adds columns; an Estimator has `fit()` and returns a Model (which is a Transformer); a Pipeline chains stages and, when fitted, becomes a `PipelineModel`, avoiding leakage and enabling reuse.

#### Q9. Why is `VectorAssembler` needed, and what do `StringIndexer`, `OneHotEncoder` and `StandardScaler` do?

Spark ML algorithms read features from one Vector column, which `VectorAssembler` builds. `StringIndexer` maps text to indices, `OneHotEncoder` makes 0/1 vectors, and `StandardScaler` computes `z = (x − μ)/σ`.

#### Q10. How do you tune hyperparameters in Spark ML?

With `ParamGridBuilder` to list values and `CrossValidator` (k-fold) or `TrainValidationSplit` to evaluate each combination with an evaluator and keep the best model; tune on the training data, test once.

#### Q11. What is Pearson correlation and what is a correlation heat map?

`r = cov(X,Y)/(σₓσᵧ)`, a unit-free measure of linear association from −1 to +1. A heat map is a colour-coded correlation matrix (`Correlation.corr` + `sns.heatmap`) revealing strong relationships and multicollinearity (|r| > 0.8 between features).

#### Q12. Explain linear regression and the least-squares principle.

It fits `ŷ = β₀ + β₁x₁ + …` by choosing coefficients that minimize the sum of squared residuals. For simple regression `β₁ = Sxy/Sxx` and `β₀ = ȳ − β₁x̄`; in matrix form `β = (XᵀX)⁻¹Xᵀy`, or by gradient descent.

#### Q13. What are Ridge and Lasso regularization, and what do `regParam` and `elasticNetParam` mean?

They add penalties to the loss: Ridge (L2) `λΣβ²` shrinks coefficients; Lasso (L1) `λΣ|β|` can set some to zero. `regParam` is λ; `elasticNetParam` is 0 for Ridge and 1 for Lasso.

#### Q14. How does logistic regression work?

It computes a linear score `z`, applies the sigmoid `p = 1/(1 + e⁻ᶻ)` to get a probability, and predicts class 1 if `p ≥ threshold`. The log-odds `ln(p/(1−p))` are linear in the features, and `e^β` is the odds ratio.

#### Q15. What loss does logistic regression minimize, and what is its gradient?

Log-loss (binary cross-entropy) `−(1/n)Σ[y ln p + (1−y) ln(1−p)]`, derived from maximum likelihood; its gradient is `(1/n)Σ(p − y)x`. For many classes it uses softmax.

#### Q16. How does a decision tree choose splits? Define entropy, Gini and information gain.

It picks the split with the highest impurity reduction. Entropy = `−Σp log₂p`; Gini = `1 − Σp²`; information gain = parent impurity minus the weighted child impurity. In the weather data Outlook (gain 0.2467) is the root.

#### Q17. How do you control overfitting in decision trees, and what are regression trees?

Limit `maxDepth`, raise `minInstancesPerNode`/`minInfoGain`, prune or use Random Forest. Regression trees split by SSE reduction and predict the mean of each leaf.

#### Q18. Describe the K-means algorithm and its objective.

Choose K, initialize centroids, then repeat: assign each point to the nearest centroid and recompute each centroid as the cluster mean until assignments stop changing. It minimizes WCSS, `Σ‖x − μ‖²`, and converges to a local minimum.

#### Q19. How do you choose K and why must features be scaled for K-means?

Use the elbow (bend in WCSS) and the silhouette score `s = (b − a)/max(a, b)` (higher is better). K-means is distance-based, so unscaled features with large ranges dominate the clusters.

#### Q20. Explain the SVM: hyperplane, margin and support vectors.

SVM finds the hyperplane `w·x + b = 0` that maximizes the margin `2/‖w‖`. Support vectors are the points on the margin lines `w·x + b = ±1`; only they determine the boundary.

#### Q21. What are hinge loss, soft margin and the effect of C/`regParam`? What are Spark's SVM limits?

Hinge loss = `max(0, 1 − y·f(x))`. The soft margin allows violations via slack with objective `½‖w‖² + CΣξ`: large C gives a narrow margin and overfitting; small C a wide margin and underfitting (Spark's `regParam` acts inversely). `LinearSVC` is linear and binary only (use `OneVsRest` for multiclass) and gives no probabilities.

#### Q22. State Bayes' theorem and the Naive Bayes assumption.

`P(C|X) = P(X|C)P(C)/P(X)`. Naive Bayes assumes features are conditionally independent given the class, so `P(X|C) = ΠP(xᵢ|C)` and the predicted class maximizes `P(C)ΠP(xᵢ|C)`.

#### Q23. What is Laplace smoothing and which Naive Bayes variants does Spark offer?

Adding α (usually 1) to counts so no probability is zero: `(count + α)/(N + αk)`. Variants: multinomial (counts/text, non-negative), Bernoulli (0/1), Gaussian (continuous) and complement.

#### Q24. What is a confusion matrix? Define TP, TN, FP, FN.

A table of actual vs predicted classes. TP: predicted positive, actually positive; TN: predicted negative, actually negative; FP: predicted positive, actually negative; FN: predicted negative, actually positive.

#### Q25. Define accuracy, precision, recall and F1, and say when accuracy misleads.

Accuracy = (TP+TN)/all; precision = TP/(TP+FP); recall = TP/(TP+FN); F1 = 2PR/(P+R). Accuracy misleads on imbalanced data (always predicting the majority class can score 99%).

#### Q26. What are the ROC curve and AUC?

ROC plots TPR against FPR over all thresholds. AUC is the area under it, equal to the probability that a random positive outranks a random negative; 0.5 is random and 1.0 perfect.

#### Q27. Define MSE, RMSE, MAE and R².

MSE = `Σ(y−ŷ)²/n`; RMSE = `√MSE` (label units, penalizes large errors); MAE = `Σ|y−ŷ|/n`; R² = `1 − SSE/SST` is the share of variance explained (1 perfect, 0 = predicting the mean, negative = worse than the mean).

#### Q28. How does the threshold affect precision and recall?

Lowering the threshold predicts positive more often: recall rises, precision falls. Raising it does the opposite. Choose by the cost of false negatives vs false positives.

#### Q29. How are clustering results evaluated without labels?

With internal measures: WSSSE (`summary.trainingCost`, lower is better but always falls with K) and the silhouette score (`ClusteringEvaluator`, higher is better).

#### Q30. Describe an end-to-end Spark ML workflow and how a trained model is deployed.

Load data, understand it (describe, correlation heat map), prepare features (Imputer, indexer, assembler, scaler), split, build a Pipeline ending in an algorithm, fit on train, evaluate on test with suitable metrics and a baseline, tune with cross-validation, then save the `PipelineModel` and load it later with `PipelineModel.load` to score new batch or streaming data.

---

## 8. Last 10-minute sheet

**Ten things to remember**

1. **Features in, label out.** Parameters are learned; hyperparameters are chosen.
2. **Fit on train, evaluate on test**; tune with CV; compare with a baseline.
3. Estimator → `fit()` → Model; Model → `transform()`; **Pipeline** chains them.
4. Features must be **one Vector column** (`VectorAssembler`); scale for K-means/SVM/regularized models.
5. **Linear regression:** minimize SSE; `β₁ = Sxy/Sxx`; metrics **RMSE, MAE, R²**.
6. **Logistic regression:** sigmoid → probability; log-loss; odds ratio `e^β`.
7. **Decision tree:** highest **information gain**; entropy `−Σp log₂p`, Gini `1 − Σp²`.
8. **K-means:** assign → recompute means; WCSS; elbow + silhouette; scale first.
9. **SVM:** margin `2/‖w‖`, support vectors, hinge loss, `LinearSVC` is binary and linear. **Naive Bayes:** Bayes + independence + smoothing.
10. **Confusion matrix** → precision, recall, F1; **AUC**; **correlation heat map** for features.

**Five-minute rescue code**

```python
from pyspark.ml.feature import VectorAssembler
from pyspark.ml.classification import LogisticRegression
from pyspark.ml.evaluation import MulticlassClassificationEvaluator

data = VectorAssembler(inputCols=feats, outputCol="features").transform(df).select("features", F.col("y").alias("label"))
train, test = data.randomSplit([0.8, 0.2], seed=42)
pred = LogisticRegression(maxIter=100).fit(train).transform(test)
print(MulticlassClassificationEvaluator(metricName="accuracy").evaluate(pred))
pred.groupBy("label", "prediction").count().show()
```

---

## 9. Unit VI status

```text
Phase 1   ML and PySpark introduction                       ✅
Phase 2   Techniques of machine learning                    ✅
Phase 3   Spark MLlib and key concepts                      ✅
Phase 4   Data understanding and correlation heat map       ✅
Phase 5   Linear regression                                 ✅
Phase 6   Logistic regression                               ✅
Phase 7   Decision tree                                     ✅
Phase 8   K-means clustering                                ✅
Phase 9   Support vector machine                            ✅
Phase 10  Naive Bayes                                       ✅
Phase 11  Model evaluation and metrics                      ✅
Phase 12  Complete practical integration                    ✅
Phase 13  Final revision + 30 viva                          ✅
```

## ✅ UNIT VI — COMPLETE

### 🎓 INT315: Cluster Computing: where your notes stand

```text
Unit III  RDDs and GraphX                       ✅ your notes
Unit IV   Spark SQL                             ✅ your Phases 1-2 + Phases 3-9
Unit V    Spark Streaming with Apache Kafka     ✅ 10 files
Unit VI   Spark ML and PySpark                  ✅ 13 files (this unit)
```

Your practical list is covered by these units: **MLlib** (Unit VI), **Spark SQL** (Unit IV), **GraphX** (Unit III), **Spark Streaming** (Unit V) and **Kafka producer and consumer** (Unit V). Good luck, mate. 🔥
