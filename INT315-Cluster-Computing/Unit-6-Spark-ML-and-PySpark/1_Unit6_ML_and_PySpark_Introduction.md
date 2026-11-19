# LET'S ACE UNIT VI, MATE! 🔥

Scala, RDDs, GraphX, Spark SQL, Kafka and Streaming are done. Now the final unit: **machine learning with Spark, written in Python (PySpark)**.

Same strategy as before, with one big upgrade for this unit, because it is full of algorithms:

- Start from absolute zero.
- **Every algorithm gets the full treatment: intuition → mathematics → a hand-worked example → PySpark code → how to read the output.**
- Simple, human-typed code with real, verified outputs.
- Practical scenarios, common errors and viva questions in every phase.

---

## Unit VI — Spark ML Programming and PySpark

Your syllabus line: **Introduction to Machine Learning and PySpark, Techniques of Machine Learning, Introduction to Spark MLlib, Key Concepts of Spark ML, Spark ML Algorithms using PySpark (Linear Regression, Logistic Regression, Decision Tree, K-means, SVM, Naive Bayes), Evaluation and performance metrics: Confusion matrix, R², RMSE, MAE, Correlation heat map.**

### Our phase-wise roadmap

| Phase | File | Topic |
| --- | --- | --- |
| **1** | `1_Unit6_ML_and_PySpark_Introduction.md` | What is ML, what is PySpark, setup *(starting now)* |
| 2 | `2_Unit6_Techniques_of_Machine_Learning.md` | Supervised, unsupervised, workflow, overfitting, bias-variance |
| 3 | `3_Unit6_Spark_MLlib_Key_Concepts.md` | MLlib, Transformer, Estimator, Pipeline, features, tuning |
| 4 | `4_Unit6_Data_Understanding_and_Correlation_Heatmap.md` | Statistics, correlation and the correlation heat map |
| 5 | `5_Unit6_Linear_Regression.md` | Linear regression: maths, gradient descent, regularization |
| 6 | `6_Unit6_Logistic_Regression.md` | Logistic regression: sigmoid, log-loss, odds |
| 7 | `7_Unit6_Decision_Tree.md` | Decision trees: entropy, Gini, information gain |
| 8 | `8_Unit6_KMeans_Clustering.md` | K-means: objective, iterations, choosing K |
| 9 | `9_Unit6_SVM.md` | SVM: margin, hinge loss, regularization |
| 10 | `10_Unit6_Naive_Bayes.md` | Naive Bayes: Bayes theorem, smoothing |
| 11 | `11_Unit6_Model_Evaluation_Metrics.md` | Confusion matrix, precision/recall, ROC, R², RMSE, MAE, CV |
| 12 | `12_Unit6_Complete_Practical_Integration.md` | End-to-end projects, algorithm choice, exam scenarios |
| 13 | `13_Unit6_Final_Revision_30_Viva.md` | Cheat sheets and exactly 30 viva questions |

---

## Phase 1 — Introduction to Machine Learning and PySpark

### Your environment

| Component | Version | Role in Unit VI |
| --- | --- | --- |
| Python | 3.10.11 | The language for this whole unit |
| Spark / PySpark | 3.5.9 | `pyspark.ml` (Spark MLlib) |
| Java | 1.8.0_501 | Runs the Spark engine underneath |
| numpy, pandas | latest compatible | Arrays and small tables |
| matplotlib, seaborn | latest compatible | Charts and the **correlation heat map** |

Install the Python libraries once (CMD):

```powershell
pip install numpy pandas matplotlib seaborn
```

### How these notes were verified (honest note)

Every PySpark snippet and output in Unit VI was **run in a real PySpark 3.5 session** (the API is identical in your 3.5.9). Where a number comes from an algorithm, it is the real output. Two harmless differences you may see on your machine:

1. Printing a pandas DataFrame's *type* shows `pandas.core.frame.DataFrame` on older pandas and `pandas.DataFrame` on pandas 3.
2. Printing a list of coefficients may show `np.float64(4.06)` on numpy 2 and `4.06` on numpy 1. We print Spark vectors directly, which are always clean.

---

### 1. What is machine learning?

> **Machine learning (ML) is the field of building programs that learn patterns from data and use them to make predictions or decisions, without being explicitly programmed with rules.**

**Traditional programming vs machine learning**

```text
TRADITIONAL PROGRAMMING                 MACHINE LEARNING
 Rules  ──┐                              Data    ──┐
          ├──▶ Program ──▶ Answers       Answers ──┴──▶ ML algorithm ──▶ Rules (the MODEL)
 Data   ──┘                                                                  │
                                                  new Data ──▶ MODEL ──▶ predictions
```

**Example.** To decide whether a student will pass:

- *Traditional:* a programmer writes `if hours > 5 and attendance > 75: pass`. The rule is guessed by a human.
- *Machine learning:* we give 200 past students (hours, attendance, previous marks, result). The algorithm **finds the rule itself** and can handle combinations a human would never write.

---

### 2. AI vs ML vs Deep Learning

```text
┌───────────────────────────────────────────────┐
│ Artificial Intelligence (machines acting smart)│
│   ┌───────────────────────────────────────┐   │
│   │ Machine Learning (learn from data)    │   │
│   │   ┌───────────────────────────────┐   │   │
│   │   │ Deep Learning (neural networks)│   │   │
│   │   └───────────────────────────────┘   │   │
│   └───────────────────────────────────────┘   │
└───────────────────────────────────────────────┘
```

| Term | Meaning | Example |
| --- | --- | --- |
| **AI** | Any technique that makes machines behave intelligently | Chess programs, chatbots |
| **ML** | AI that **learns from data** | Spam filters, price prediction |
| **Deep learning** | ML using many-layered neural networks | Image and speech recognition |

Unit VI covers **classical ML** (regression, trees, clustering, SVM, Naive Bayes), which is exactly what Spark MLlib provides.

---

### 3. Why machine learning on Spark?

| Problem with single-machine ML | How Spark helps |
| --- | --- |
| Data too large for one computer's memory | Data is **distributed** across a cluster |
| Training takes too long | Training runs **in parallel** on many cores/machines |
| Data lives in Hadoop/Kafka/files at scale | Spark reads all of them directly |
| Separate tools for cleaning and modelling | One platform: SQL + DataFrames + ML (Units IV, V, VI) |

**Honest guidance:** for a small dataset (thousands of rows), scikit-learn on one machine is simpler. Spark ML shines when data is **big** (millions to billions of rows) and when ML must be part of a larger Spark pipeline. For learning, small datasets are perfect because the *concepts and API are identical*.

---

### 4. The ML vocabulary (learn this table now)

| Term | Meaning | In our student dataset |
| --- | --- | --- |
| **Dataset** | The whole collection of data | 200 students |
| **Sample / record / row** | One example | One student |
| **Feature** (input, attribute, `X`) | A measurable property used to predict | `hours`, `attendance`, `prev_marks` |
| **Label / target** (output, `y`) | The thing we want to predict | `final_marks` (a number) or `passed` (0/1) |
| **Model** | The learned rule/function `y = f(X)` | The fitted equation |
| **Training** | Letting the algorithm learn from data | `model = algorithm.fit(train)` |
| **Prediction / inference** | Using the model on new data | `model.transform(new_data)` |
| **Training set** | Data used to learn | 80% of the rows |
| **Test set** | Data held back to check the model | 20% of the rows |
| **Parameter** | A number the model **learns** | Coefficients (weights) |
| **Hyperparameter** | A setting **we choose** before training | `maxDepth`, `k`, `regParam` |
| **Loss / cost function** | A number measuring how wrong the model is | Mean squared error |

```text
 Features (X)                          Label (y)
 hours attendance prev_marks   ────▶   final_marks
  6.8     51.3       50.1                 63.4
  7.1     94.6       39.8                 75.8
```

> **Memory line:** *Features go in, the label comes out. Parameters are learned; hyperparameters are chosen.*

---

### 5. The machine learning workflow

Every ML project, in any tool, follows these steps:

```text
 1. Define the problem        (predict marks? pass/fail? group customers?)
        ↓
 2. Collect data              (files, databases, Kafka)
        ↓
 3. Explore & clean data      (missing values, outliers, correlations)   ← Phase 4
        ↓
 4. Prepare features          (encode, scale, assemble into a vector)    ← Phase 3
        ↓
 5. Split data                (train / test)
        ↓
 6. Choose & train a model    (fit)                                       ← Phases 5–10
        ↓
 7. Evaluate                  (metrics on the TEST set)                   ← Phase 11
        ↓
 8. Tune & improve            (hyperparameters, cross-validation)
        ↓
 9. Deploy / predict          (save the model, use on new data)
```

Steps 3 to 8 are often repeated many times. We will walk this path for every algorithm.

---

### 6. What is PySpark?

> **PySpark is the Python API of Apache Spark. It lets us write Spark programs (DataFrames, SQL, streaming and MLlib) in Python.**

Why Python for machine learning?

- The data-science world uses Python (numpy, pandas, matplotlib, scikit-learn).
- Simple, readable syntax.
- Your syllabus text book is *Advanced Analytics with PySpark*.

**How it works inside: the Py4J bridge**

```text
┌─────────────────────┐        ┌──────────────────────────────────────────┐
│  Your Python code   │        │              Spark (JVM)                 │
│  (driver process)   │  Py4J  │  Driver JVM  →  Executors (tasks)        │
│  spark.read..       │◀──────▶│  Catalyst optimizer, MLlib algorithms    │
│  LogisticRegression │ bridge │  (Scala/Java code, runs distributed)     │
└─────────────────────┘        └──────────────────────────────────────────┘
```

1. Your Python statements become **commands sent to the Spark JVM** through **Py4J**.
2. DataFrame and MLlib operations run **inside the JVM**, so they are as fast as Scala.
3. Only Python-specific pieces (Python UDFs, RDD `lambda` functions) need extra Python worker processes and are slower.

> **Key idea:** with DataFrames and `pyspark.ml`, Python is just the *remote control*; the heavy work runs in the JVM. That is why PySpark ML performance is similar to Scala.

---

### 7. Starting PySpark

There are three ways to run PySpark code.

#### 7.1 The PySpark shell (interactive)

```powershell
pyspark
```

You get a Python prompt `>>>` with `spark` (SparkSession) and `sc` (SparkContext) already created, like `spark-shell` in Scala.

```python
>>> spark.version
'3.5.9'
```

#### 7.2 A Python script run with `spark-submit`

Save code in `C:\sparklabs\lab1.py` and run:

```powershell
spark-submit C:\sparklabs\lab1.py
```

#### 7.3 Any Python environment (VS Code, Jupyter, IDLE)

Make sure Python can import `pyspark` (`pip install pyspark==3.5.9`, matching your Spark) and write:

```python
from pyspark.sql import SparkSession

spark = (SparkSession.builder
         .master("local[2]")
         .appName("unit6")
         .config("spark.sql.shuffle.partitions", "4")
         .getOrCreate())

spark.sparkContext.setLogLevel("ERROR")      # hide noisy INFO/WARN messages
```

| Line | Meaning |
| --- | --- |
| `SparkSession.builder` | Start configuring a session |
| `.master("local[2]")` | Run on this laptop with 2 cores |
| `.appName("unit6")` | A name shown in the Spark UI |
| `shuffle.partitions = 4` | Small value: faster on small data |
| `.getOrCreate()` | Create the session, or reuse the existing one |
| `setLogLevel("ERROR")` | Print only errors |

**Pointing Spark at the right Python (Windows)**

If Spark picks a different Python from the one you want (3.10.11), set it before starting:

```powershell
set PYSPARK_PYTHON=python
set PYSPARK_DRIVER_PYTHON=python
pyspark
```

---

### 8. PySpark DataFrame basics

Unit IV taught DataFrames in Scala. The ideas are identical; only syntax changes. ✅ **All outputs below are from a real run.**

**Create a DataFrame**

```python
rows = [(101, "Alice", "CSE", 21, 85),
        (102, "Bob", "ECE", 22, 72),
        (103, "Charlie", "CSE", 21, 91),
        (104, "David", "IT", 23, 65),
        (105, "Emma", "ECE", 22, 88)]

df = spark.createDataFrame(rows, ["student_id", "name", "department", "age", "marks"])
df.printSchema()
df.show()
```

```text
root
 |-- student_id: long (nullable = true)
 |-- name: string (nullable = true)
 |-- department: string (nullable = true)
 |-- age: long (nullable = true)
 |-- marks: long (nullable = true)

+----------+-------+----------+---+-----+
|student_id|   name|department|age|marks|
+----------+-------+----------+---+-----+
|       101|  Alice|       CSE| 21|   85|
|       102|    Bob|       ECE| 22|   72|
|       103|Charlie|       CSE| 21|   91|
|       104|  David|        IT| 23|   65|
|       105|   Emma|       ECE| 22|   88|
+----------+-------+----------+---+-----+
```

> **Difference from Scala:** Python integers become `long` (64-bit), while Scala `Int` became `integer`. Python floats become `double`.

**Select, filter, group, add a column**

```python
from pyspark.sql import functions as F

df.select("name", "marks").filter(F.col("marks") > 80).show()
```

```text
+-------+-----+
|   name|marks|
+-------+-----+
|  Alice|   85|
|Charlie|   91|
|   Emma|   88|
+-------+-----+
```

```python
df.groupBy("department").agg(F.count("*").alias("n"), F.round(F.avg("marks"), 1).alias("avg_marks")).orderBy("department").show()
```

```text
+----------+---+---------+
|department|  n|avg_marks|
+----------+---+---------+
|       CSE|  2|     88.0|
|       ECE|  2|     80.0|
|        IT|  1|     65.0|
+----------+---+---------+
```

```python
df.withColumn("grade", F.when(F.col("marks") >= 90, "A")
                        .when(F.col("marks") >= 80, "B")
                        .otherwise("C")).select("name", "marks", "grade").show()
```

```text
+-------+-----+-----+
|   name|marks|grade|
+-------+-----+-----+
|  Alice|   85|    B|
|    Bob|   72|    C|
|Charlie|   91|    A|
|  David|   65|    C|
|   Emma|   88|    B|
+-------+-----+-----+
```

**SQL still works (Unit IV)**

```python
df.createOrReplaceTempView("students")
spark.sql("SELECT department, MAX(marks) AS top FROM students GROUP BY department ORDER BY department").show()
```

```text
+----------+---+
|department|top|
+----------+---+
|       CSE| 91|
|       ECE| 88|
|        IT| 65|
+----------+---+
```

**Quick statistics**

```python
df.describe("marks").show()
```

```text
+-------+------------------+
|summary|             marks|
+-------+------------------+
|  count|                 5|
|   mean|              80.2|
| stddev|11.166915420114902|
|    min|                65|
|    max|                91|
+-------+------------------+
```

**Moving between Spark and pandas**

```python
pdf = df.toPandas()          # Spark DataFrame → pandas (pulls data to the driver)
print(pdf["marks"].mean())   # 80.2

sdf = spark.createDataFrame(pdf)   # pandas → Spark DataFrame
print(sdf.count())                 # 5
```

> **Warning:** `toPandas()` collects *everything* into the driver's memory. Use it only for **small results** (summaries, a sample, the correlation matrix), never on a huge dataset.

---

### 9. Scala vs PySpark syntax table

You already know Scala from Units III to V. This table converts your knowledge.

| Idea | Scala | PySpark (Python) |
| --- | --- | --- |
| Variable | `val df = ...` | `df = ...` |
| Column | `col("x")` or `$"x"` | `F.col("x")` or `df["x"]` |
| Imports | `import org.apache.spark.sql.functions._` | `from pyspark.sql import functions as F` |
| Equal | `===` | `==` |
| AND / OR / NOT | `&&`, `\|\|`, `!` | `&`, `\|`, `~` (with brackets!) |
| Alias | `.as("n")` | `.alias("n")` |
| If-else column | `when(...).otherwise(...)` | `F.when(...).otherwise(...)` |
| Lambda | `x => x + 1` | `lambda x: x + 1` |
| Print | `println(x)` | `print(x)` |
| Multi-line chain | `:paste` needed in the shell | Wrap in `( ... )` or use `\` |
| Tuples | `(1, "a")` | `(1, "a")` |
| Class `Row` | `Row(1, "a")` | `Row(id=1, name="a")` |

**Boolean conditions need brackets**

```python
df.filter((F.col("marks") > 70) & (F.col("department") == "CSE"))    # ✅
df.filter(F.col("marks") > 70 & F.col("department") == "CSE")        # ❌ error
```

The ❌ version fails because `&` binds tighter than `>` in Python. Always bracket each condition.

---

### 10. PySpark vs pandas vs scikit-learn

| | pandas | scikit-learn | **PySpark / Spark ML** |
| --- | --- | --- | --- |
| Data size | Fits in one machine's memory | Fits in memory | **Distributed (billions of rows)** |
| Execution | Immediate | Immediate | **Lazy** (runs on an action) |
| Algorithms | – | Very large library | The most important ones |
| Speed on small data | Fastest | Fast | Slower (startup cost) |
| Speed on big data | Fails | Fails | **Scales** |
| Data type | `DataFrame` | numpy arrays | Spark `DataFrame` + `Vector` column |
| Typical use | Exploration | Prototyping | **Production on big data** |

Practical pattern: use **Spark** for heavy data and **pandas/matplotlib** only for the *small final results* (for example plotting a correlation matrix).

---

### 11. Common PySpark errors

| Symptom | Cause | Fix |
| --- | --- | --- |
| *Python in worker has different version than that in driver* | Two different Pythons | Set `PYSPARK_PYTHON` and `PYSPARK_DRIVER_PYTHON` to the same Python |
| *Java gateway process exited before sending its port number* | Java missing or `JAVA_HOME` wrong | Check `java -version` and `JAVA_HOME` |
| `ModuleNotFoundError: No module named 'pyspark'` | PySpark not importable | `pip install pyspark==3.5.9` or run with `spark-submit` |
| `ValueError: Cannot convert column into bool ...` | Used `and`/`or`/`&` without brackets | Use `&`, `\|`, `~` and bracket every condition |
| `ModuleNotFoundError: No module named 'numpy'` | `pyspark.ml` needs numpy | `pip install numpy` |
| Output flooded with INFO/WARN | Default log level | `spark.sparkContext.setLogLevel("ERROR")` |
| `toPandas()` crashes with out-of-memory | Collected too much | Aggregate or sample first |
| Wrong results after `col("a") == None` | Comparing to None | Use `F.col("a").isNull()` |
| Column not found | Typo or wrong case | `df.columns` |
| Very slow on a laptop | 200 shuffle partitions | `spark.sql.shuffle.partitions=4` |

---

### 12. Phase 1 — practical scenarios

1. Check **Python and Spark versions** (`python --version`, `spark.version`).
2. Start the **`pyspark` shell** and confirm `spark` and `sc` exist.
3. Create the 5-student DataFrame and print its **schema**.
4. **Filter** students with marks above 80.
5. **Group by** department and compute the average marks.
6. Add a **grade column** with `F.when`.
7. Register a view and run a **SQL** query.
8. Convert the DataFrame to **pandas** and back.
9. Write the **boolean filter** `(marks > 70) & (department == "CSE")` correctly.
10. Run a script with **`spark-submit`**.

---

### 13. Phase 1 — viva questions and answers

#### Q1. What is machine learning?

Building programs that learn patterns from data to make predictions or decisions without being explicitly programmed with rules.

#### Q2. How does machine learning differ from traditional programming?

Traditional programming takes rules and data to produce answers; machine learning takes data and answers to produce the rules (the model).

#### Q3. What is the difference between AI, ML and deep learning?

AI is the broad goal of intelligent machines; ML is AI that learns from data; deep learning is ML using deep neural networks.

#### Q4. What are features and labels?

Features are the input variables used for prediction; the label (target) is the value we want to predict.

#### Q5. What is the difference between a parameter and a hyperparameter?

A parameter is learned from data (for example coefficients); a hyperparameter is chosen before training (for example `maxDepth`, `k`).

#### Q6. What are training and test sets?

The training set is used to learn the model; the test set, kept aside, checks how well the model works on unseen data.

#### Q7. What are the steps of an ML workflow?

Define the problem, collect data, explore and clean, prepare features, split, train, evaluate, tune, and deploy.

#### Q8. What is PySpark?

The Python API of Apache Spark for DataFrames, SQL, streaming and MLlib.

#### Q9. How does PySpark communicate with Spark?

Through the Py4J bridge, which sends Python commands to the Spark JVM where the work executes.

#### Q10. Why is PySpark DataFrame/ML code about as fast as Scala?

Because DataFrame and MLlib operations execute inside the JVM; Python only sends the commands.

#### Q11. Why use Spark for machine learning?

To train on data too big for one machine, in parallel, and to integrate ML with data processing and streaming in one platform.

#### Q12. How do you start PySpark?

Run the `pyspark` shell, use `spark-submit script.py`, or build a `SparkSession` in any Python environment.

#### Q13. What does `toPandas()` do and what is the risk?

It converts a Spark DataFrame to pandas by collecting all data on the driver; large data can exhaust driver memory.

#### Q14. How do you combine conditions in a PySpark filter?

With `&`, `|` and `~`, wrapping each condition in brackets.

#### Q15. Name one environment variable that fixes a Python-version mismatch.

`PYSPARK_PYTHON` (together with `PYSPARK_DRIVER_PYTHON`).

---

### 14. Phase 1 — final memory map

```text
ML = learn the RULE from DATA
  features (X) ──▶ MODEL ──▶ label (y)
  parameter = learned ; hyperparameter = chosen
  workflow: problem → data → clean → features → split → train → evaluate → tune → deploy

PYSPARK = Python API of Spark
  Python ──Py4J──▶ Spark JVM (DataFrames + MLlib run here)
  start: pyspark | spark-submit | SparkSession.builder...getOrCreate()
  small results → toPandas() → matplotlib/seaborn
```

**One-line summary**

> *Machine learning lets Spark learn rules from big data, and PySpark lets us drive all of it from Python.*

### Phase 1 completed

---

> **Next → Phase 2: Techniques of Machine Learning** (supervised, unsupervised, semi-supervised, reinforcement learning; regression vs classification vs clustering; overfitting, underfitting, bias-variance; train/test split and cross-validation; a numpy overfitting experiment). File: `2_Unit6_Techniques_of_Machine_Learning.md`
