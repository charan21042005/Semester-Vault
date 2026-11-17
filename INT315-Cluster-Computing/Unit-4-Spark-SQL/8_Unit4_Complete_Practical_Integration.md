# Unit IV — Phase 8: Complete Practical Integration

> **Continuing from Phase 7.** We now know every piece: RDD → DataFrame, views, SQL, wildcards, grouping, joins. In this phase we **combine everything** into complete, exam-style workflows, and learn how to attack an unseen question.
>
> All code and outputs were run in a real `spark-shell` (Spark 3.5.1). File paths use the Windows style `C:/sparkdata/...` as in earlier units.

---

## Phase 8 — Roadmap

| # | Topic |
| --- | --- |
| 1 | The universal Spark SQL pipeline |
| 2 | Preparing the data files |
| 3 | Project A — Student Result System |
| 4 | Project B — Sales Report |
| 5 | Reading files directly with `spark.read` |
| 6 | Saving results |
| 7 | RDD ↔ DataFrame ↔ SQL together |
| 8 | The "question → clause" method |
| 9 | Ten complete exam-style scenarios |
| 10 | Common integration mistakes |
| 11 | Master template |

---

### 1. The universal Spark SQL pipeline

Almost every Unit IV practical follows the same six steps:

```text
Step 1  Get the data        →  sc.textFile / Seq / spark.read
Step 2  Parse it            →  split, toInt  (RDD stage)
Step 3  Give it structure   →  toDF(...) / createDataFrame(rowRDD, schema)
Step 4  Register a view     →  createOrReplaceTempView("name")
Step 5  Query with SQL      →  spark.sql("SELECT ...")
Step 6  Show / save result  →  show() / write
```

```text
 file ──▶ RDD ──▶ DataFrame ──▶ view ──▶ SQL ──▶ result
```

---

### 2. Preparing the data files

Create a folder `C:\sparkdata` and put these files inside.

**`students.txt`** (no header)

```text
101,Alice,CSE,21,85
102,Bob,ECE,22,72
103,Charlie,CSE,21,91
104,David,IT,23,65
105,Emma,ECE,22,88
```

**`sales.txt`** (no header): `order_id,product,city,quantity,price`

```text
1,Laptop,Delhi,2,55000
2,Phone,Mumbai,5,20000
3,Laptop,Mumbai,1,55000
4,Tablet,Delhi,3,15000
5,Phone,Delhi,2,20000
6,Tablet,Mumbai,4,15000
```

**`students.csv`** (with a header row)

```text
student_id,name,department,age,marks
101,Alice,CSE,21,85
102,Bob,ECE,22,72
103,Charlie,CSE,21,91
104,David,IT,23,65
105,Emma,ECE,22,88
```

Start the shell and import everything we need once:

```scala
import spark.implicits._
import org.apache.spark.sql.functions._
import org.apache.spark.sql.Row
import org.apache.spark.sql.types._
```

---

### 3. Project A — Student Result System

**Goal:** Read student records, convert them to a DataFrame, and produce reports with SQL.

#### Step 1 — File → RDD → Rows

```scala
val lines = sc.textFile("C:/sparkdata/students.txt")

val rowRDD = lines.map(_.split(",")).map(p =>
  Row(p(0).toInt, p(1), p(2), p(3).toInt, p(4).toInt)
)
```

#### Step 2 — Define a schema and create the DataFrame

```scala
val schema = StructType(Array(
  StructField("student_id", IntegerType, false),
  StructField("name", StringType, true),
  StructField("department", StringType, true),
  StructField("age", IntegerType, true),
  StructField("marks", IntegerType, true)
))

val studentsDF = spark.createDataFrame(rowRDD, schema)
```

#### Step 3 — Register the view

```scala
studentsDF.createOrReplaceTempView("students")
```

#### Report 1 — Pass count per department (pass mark = 60)

```scala
spark.sql("""
  SELECT department,
         COUNT(*) AS total,
         SUM(CASE WHEN marks >= 60 THEN 1 ELSE 0 END) AS passed
  FROM students
  GROUP BY department
  ORDER BY department
""").show()
```

```text
+----------+-----+------+
|department|total|passed|
+----------+-----+------+
|       CSE|    2|     2|
|       ECE|    2|     2|
|        IT|    1|     1|
+----------+-----+------+
```

`SUM(CASE WHEN ... THEN 1 ELSE 0 END)` is a standard trick: it **counts rows that meet a condition**.

#### Report 2 — Grade for every student

```scala
spark.sql("""
  SELECT name, marks,
         CASE WHEN marks >= 90 THEN 'A'
              WHEN marks >= 80 THEN 'B'
              WHEN marks >= 70 THEN 'C'
              ELSE 'D' END AS grade
  FROM students
  ORDER BY marks DESC
""").show()
```

```text
+-------+-----+-----+
|   name|marks|grade|
+-------+-----+-----+
|Charlie|   91|    A|
|   Emma|   88|    B|
|  Alice|   85|    B|
|    Bob|   72|    C|
|  David|   65|    D|
+-------+-----+-----+
```

#### Report 3 — Students per grade (reuse the grade logic through a view)

Writing the same `CASE` twice is messy. Save it as a view first:

```scala
val graded = spark.sql("""
  SELECT *,
         CASE WHEN marks >= 90 THEN 'A'
              WHEN marks >= 80 THEN 'B'
              WHEN marks >= 70 THEN 'C'
              ELSE 'D' END AS grade
  FROM students
""")
graded.createOrReplaceTempView("graded")

spark.sql("""
  SELECT grade, COUNT(*) AS students
  FROM graded
  GROUP BY grade
  ORDER BY grade
""").show()
```

```text
+-----+--------+
|grade|students|
+-----+--------+
|    A|       1|
|    B|       2|
|    C|       1|
|    D|       1|
+-----+--------+
```

#### Report 4 — Topper of each department (correlated subquery)

```scala
spark.sql("""
  SELECT name, department, marks
  FROM students s
  WHERE marks = (SELECT MAX(marks) FROM students WHERE department = s.department)
  ORDER BY department
""").show()
```

```text
+-------+----------+-----+
|   name|department|marks|
+-------+----------+-----+
|Charlie|       CSE|   91|
|   Emma|       ECE|   88|
|  David|        IT|   65|
+-------+----------+-----+
```

The inner query runs **per row**, using that row's department (`s.department`).

#### Report 5 — Overall topper

```scala
spark.sql("""
  SELECT name, marks
  FROM students
  WHERE marks = (SELECT MAX(marks) FROM students)
""").show()
```

```text
+-------+-----+
|   name|marks|
+-------+-----+
|Charlie|   91|
+-------+-----+
```

#### Report 6 — Students from departments averaging above 85 (`IN` + subquery)

```scala
spark.sql("""
  SELECT name, marks
  FROM students
  WHERE department IN (
        SELECT department FROM students
        GROUP BY department HAVING AVG(marks) > 85)
  ORDER BY name
""").show()
```

```text
+-------+-----+
|   name|marks|
+-------+-----+
|  Alice|   85|
|Charlie|   91|
+-------+-----+
```

Only CSE (average 88.0) qualifies.

---

### 4. Project B — Sales Report

**Goal:** Read sales lines, convert them, join with a product-category table, and report revenue.

#### Step 1 — File → RDD → DataFrame

```scala
val sales = sc.textFile("C:/sparkdata/sales.txt")
  .map(_.split(","))
  .map(p => (p(0).toInt, p(1), p(2), p(3).toInt, p(4).toInt))
  .toDF("order_id", "product", "city", "quantity", "price")

sales.createOrReplaceTempView("sales")
sales.printSchema()
```

```text
root
 |-- order_id: integer (nullable = false)
 |-- product: string (nullable = true)
 |-- city: string (nullable = true)
 |-- quantity: integer (nullable = false)
 |-- price: integer (nullable = false)
```

#### Step 2 — A second table: product categories

```scala
val cat = Seq(
  ("Laptop", "Computing"),
  ("Tablet", "Computing"),
  ("Phone",  "Mobile")
).toDF("product", "category")

cat.createOrReplaceTempView("categories")
```

#### Report 1 — Revenue per category (join + group)

```scala
spark.sql("""
  SELECT c.category, SUM(s.quantity * s.price) AS revenue
  FROM sales s
  JOIN categories c ON s.product = c.product
  GROUP BY c.category
  ORDER BY revenue DESC
""").show()
```

```text
+---------+-------+
| category|revenue|
+---------+-------+
|Computing| 270000|
|   Mobile| 140000|
+---------+-------+
```

#### Report 2 — City + product combinations earning ₹1,00,000 or more

```scala
spark.sql("""
  SELECT city, product, SUM(quantity * price) AS revenue
  FROM sales
  GROUP BY city, product
  HAVING SUM(quantity * price) >= 100000
  ORDER BY city, revenue DESC
""").show()
```

```text
+------+-------+-------+
|  city|product|revenue|
+------+-------+-------+
| Delhi| Laptop| 110000|
|Mumbai|  Phone| 100000|
+------+-------+-------+
```

#### Report 3 — Best-selling product in Delhi

```scala
spark.sql("""
  SELECT product, SUM(quantity * price) AS revenue
  FROM sales
  WHERE city = 'Delhi'
  GROUP BY product
  ORDER BY revenue DESC
  LIMIT 1
""").show()
```

```text
+-------+-------+
|product|revenue|
+-------+-------+
| Laptop| 110000|
+-------+-------+
```

---

### 5. Reading files directly with `spark.read`

For **structured files** (CSV, JSON, Parquet), Spark can create a DataFrame **directly**, skipping the RDD step.

#### 5.1 CSV with a header and schema inference

```scala
val csvDF = spark.read
  .option("header", "true")
  .option("inferSchema", "true")
  .csv("C:/sparkdata/students.csv")

csvDF.show()
csvDF.printSchema()
```

```text
+----------+-------+----------+---+-----+
|student_id|   name|department|age|marks|
+----------+-------+----------+---+-----+
|       101|  Alice|       CSE| 21|   85|
|       102|    Bob|       ECE| 22|   72|
|       103|Charlie|       CSE| 21|   91|
|       104|  David|        IT| 23|   65|
|       105|   Emma|       ECE| 22|   88|
+----------+-------+----------+---+-----+

root
 |-- student_id: integer (nullable = true)
 |-- name: string (nullable = true)
 |-- department: string (nullable = true)
 |-- age: integer (nullable = true)
 |-- marks: integer (nullable = true)
```

| Option | Meaning |
| --- | --- |
| `header = true` | The first line holds column names |
| `inferSchema = true` | Spark scans the data and guesses the types |

#### 5.2 Without `inferSchema`: everything is a string

```scala
val csvNoInfer = spark.read.option("header", "true").csv("C:/sparkdata/students.csv")
csvNoInfer.printSchema()
```

```text
root
 |-- student_id: string (nullable = true)
 |-- name: string (nullable = true)
 |-- department: string (nullable = true)
 |-- age: string (nullable = true)
 |-- marks: string (nullable = true)
```

All columns are `string`. This is the CSV equivalent of forgetting `.toInt`.

#### 5.3 With our own schema (fastest and safest)

```scala
val csvSchema = spark.read
  .option("header", "true")
  .schema(schema)
  .csv("C:/sparkdata/students.csv")

csvSchema.printSchema()
```

The types follow our `StructType`, and Spark skips the extra pass over the data that `inferSchema` needs.

**Reader methods**

| Format | Method |
| --- | --- |
| CSV | `spark.read.csv(path)` |
| JSON | `spark.read.json(path)` |
| Parquet | `spark.read.parquet(path)` |
| Text | `spark.read.text(path)` (one column named `value`) |

**When to use RDD parsing vs `spark.read`**

| Use `sc.textFile` + `map` | Use `spark.read` |
| --- | --- |
| Messy or custom line formats (logs) | Clean CSV / JSON / Parquet |
| Needs custom parsing logic | Wants type inference or a header |

---

### 6. Saving results

A DataFrame can be written back to files.

```scala
spark.sql("""
  SELECT department, AVG(marks) AS avg_marks
  FROM students GROUP BY department ORDER BY department
""").write.mode("overwrite").option("header", "true").csv("C:/sparkdata/out_dept")
```

Read it back to check:

```scala
spark.read.option("header", "true").csv("C:/sparkdata/out_dept").show()
```

```text
+----------+---------+
|department|avg_marks|
+----------+---------+
|       CSE|     88.0|
|       ECE|     80.0|
|        IT|     65.0|
+----------+---------+
```

**Important facts**

- `out_dept` is a **folder**, not a single file. Spark writes one `part-...` file per partition, plus a `_SUCCESS` marker.
- `mode("overwrite")` replaces an existing folder. The default mode throws an error if the folder already exists (same as the `saveAsTextFile` issue in Unit III).
- Use `.coalesce(1)` before `.write` to get a single part file for small results.

| Write mode | Behaviour |
| --- | --- |
| `error` (default) | Fail if the path exists |
| `overwrite` | Replace |
| `append` | Add to existing |
| `ignore` | Do nothing if it exists |

**JSON output example**

```scala
studentsDF.write.mode("overwrite").json("C:/sparkdata/out_json")
spark.read.json("C:/sparkdata/out_json").orderBy("student_id").show()
```

```text
+---+----------+-----+-------+----------+
|age|department|marks|   name|student_id|
+---+----------+-----+-------+----------+
| 21|       CSE|   85|  Alice|       101|
| 22|       ECE|   72|    Bob|       102|
| 21|       CSE|   91|Charlie|       103|
| 23|        IT|   65|  David|       104|
| 22|       ECE|   88|   Emma|       105|
+---+----------+-----+-------+----------+
```

JSON reading sorts columns alphabetically, so the column order differs from the original. That is normal.

---

### 7. RDD ↔ DataFrame ↔ SQL together

A realistic workflow uses all three layers:

```scala
// DataFrame → RDD: use pair-RDD logic from Unit III
studentsDF.rdd
  .map(r => (r.getString(2), r.getInt(4)))
  .reduceByKey(_ + _)
  .collect()
  .sortBy(_._1)
```

```text
res21: Array[(String, Int)] = Array((CSE,176), (ECE,160), (IT,65))
```

The same answer through SQL:

```scala
spark.sql("SELECT department, SUM(marks) FROM students GROUP BY department ORDER BY department").show()
```

```text
RDD route  : reduceByKey(_ + _)    → (CSE,176) (ECE,160) (IT,65)
SQL route  : SUM(marks) GROUP BY   → CSE 176, ECE 160, IT 65
```

> **Insight:** `reduceByKey` is the RDD version of `GROUP BY` + `SUM`. If asked "which is easier / more optimized", say SQL/DataFrames, because Catalyst optimizes them automatically.

---

### 8. The "question → clause" method

Read the question and **underline keywords**. Each keyword maps to a SQL clause.

| Keyword in the question | SQL clause / function |
| --- | --- |
| "show", "display", "list" | `SELECT` |
| "from the table of students" | `FROM students` |
| "where", "only", "who scored above" | `WHERE` |
| "each", "per", "by department" | `GROUP BY` |
| "total", "sum" | `SUM()` |
| "average", "mean" | `AVG()` |
| "how many", "number of" | `COUNT()` |
| "highest", "maximum" | `MAX()` or `ORDER BY ... DESC LIMIT 1` |
| "lowest", "minimum" | `MIN()` or `ORDER BY ... ASC LIMIT 1` |
| "with average above…" (on a group) | `HAVING` |
| "sorted", "in order", "ranked" | `ORDER BY` |
| "top 3" | `ORDER BY ... DESC LIMIT 3` |
| "starting with", "ending with", "containing" | `LIKE` with `%` |
| "exactly N letters" | `LIKE` with `_` |
| "unique", "different" | `DISTINCT` |
| "between X and Y" | `BETWEEN` |
| "name of the head / details from another table" | `JOIN` |
| "even if no match", "including those without" | `LEFT JOIN` |
| "not in", "never", "missing" | `LEFT ANTI JOIN` |
| "above the class average" | Subquery with `AVG()` |

**Worked example**

> *"Display the average marks of each department that has more than one student, highest average first."*

```text
"average marks"         → AVG(marks)
"of each department"    → GROUP BY department
"more than one student" → HAVING COUNT(*) > 1
"highest average first" → ORDER BY avg_marks DESC
```

```scala
spark.sql("""
  SELECT department, AVG(marks) AS avg_marks
  FROM students
  GROUP BY department
  HAVING COUNT(*) > 1
  ORDER BY avg_marks DESC
""").show()
```

```text
+----------+---------+
|department|avg_marks|
+----------+---------+
|       CSE|     88.0|
|       ECE|     80.0|
+----------+---------+
```

---

### 9. Ten complete exam-style scenarios

Use `students` (from `students.txt`), `sales`, `categories`, and the `departments` table from Phase 7.

#### Scenario 1 — RDD to DataFrame to SQL (the basics)

**Question:** Convert an RDD of (name, marks) into a DataFrame and display students with marks above 80 using SQL.

```scala
val rdd = sc.parallelize(Seq(("Alice", 85), ("Bob", 72), ("Charlie", 91)))
val df = rdd.toDF("name", "marks")
df.createOrReplaceTempView("marks_table")
spark.sql("SELECT * FROM marks_table WHERE marks > 80").show()
```

```text
+-------+-----+
|   name|marks|
+-------+-----+
|  Alice|   85|
|Charlie|   91|
+-------+-----+
```

#### Scenario 2 — Schema with `StructType`

**Question:** Read `students.txt`, apply an explicit schema, and show the schema.

→ Project A, Steps 1–2, then `studentsDF.printSchema()`.

#### Scenario 3 — Filtering and sorting

**Question:** Show the names and marks of students from CSE, highest marks first.

```scala
spark.sql("SELECT name, marks FROM students WHERE department = 'CSE' ORDER BY marks DESC").show()
```

```text
+-------+-----+
|   name|marks|
+-------+-----+
|Charlie|   91|
|  Alice|   85|
+-------+-----+
```

#### Scenario 4 — Wildcards

**Question:** Display students whose names start with A or end with a.

```scala
spark.sql("SELECT name FROM students WHERE name LIKE 'A%' OR name LIKE '%a'").show()
```

```text
+-----+
| name|
+-----+
|Alice|
| Emma|
+-----+
```

Alice starts with `A`; Emma ends with `a`. David contains an `a` but neither starts with `A` nor ends with `a`, so he is excluded.

> Always run the query and read the result. Checking the output against the data is how you catch pattern mistakes.

#### Scenario 5 — Group and aggregate

**Question:** Find the number of students and the average marks in each department.

```scala
spark.sql("""
  SELECT department, COUNT(*) AS n, ROUND(AVG(marks), 1) AS avg_marks
  FROM students GROUP BY department ORDER BY department
""").show()
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

#### Scenario 6 — HAVING

**Question:** Display departments whose total marks exceed 150.

```scala
spark.sql("""
  SELECT department, SUM(marks) AS total
  FROM students GROUP BY department
  HAVING SUM(marks) > 150 ORDER BY department
""").show()
```

```text
+----------+-----+
|department|total|
+----------+-----+
|       CSE|  176|
|       ECE|  160|
+----------+-----+
```

#### Scenario 7 — Join (students and departments)

**Question:** Display each student with the name of their department head and building.

```scala
spark.sql("""
  SELECT s.name, d.dept_head, d.building
  FROM students s JOIN departments d ON s.department = d.department
  ORDER BY s.student_id
""").show()
```

```text
+-------+----------+--------+
|   name| dept_head|building|
+-------+----------+--------+
|  Alice|Dr. Sharma| Block A|
|    Bob| Dr. Verma| Block B|
|Charlie|Dr. Sharma| Block A|
|  David| Dr. Singh| Block C|
|   Emma| Dr. Verma| Block B|
+-------+----------+--------+
```

(This uses the five-student `students` view from Project A and the `departments` view from Phase 7.)

#### Scenario 8 — Left join with an aggregate

**Question:** Show every department with its number of students, including departments with none.

→ Phase 7, Section 14. Remember `COUNT(s.student_id)`, not `COUNT(*)`.

#### Scenario 9 — Subquery

**Question:** Display students who scored more than the average of the whole class.

```scala
spark.sql("SELECT name, marks FROM students WHERE marks > (SELECT AVG(marks) FROM students) ORDER BY marks DESC").show()
```

```text
+-------+-----+
|   name|marks|
+-------+-----+
|Charlie|   91|
|   Emma|   88|
|  Alice|   85|
+-------+-----+
```

#### Scenario 10 — Full pipeline: file → report → saved output 🔥

**Question:** Read `sales.txt`, convert it to a DataFrame, compute total revenue per product, show the result highest first, and save it as CSV.

```scala
// 1. File → RDD → DataFrame
val salesDF = sc.textFile("C:/sparkdata/sales.txt")
  .map(_.split(","))
  .map(p => (p(0).toInt, p(1), p(2), p(3).toInt, p(4).toInt))
  .toDF("order_id", "product", "city", "quantity", "price")

// 2. View
salesDF.createOrReplaceTempView("sales")

// 3. SQL report
val report = spark.sql("""
  SELECT product, SUM(quantity * price) AS revenue
  FROM sales
  GROUP BY product
  ORDER BY revenue DESC
""")

// 4. Show
report.show()

// 5. Save
report.write.mode("overwrite").option("header", "true").csv("C:/sparkdata/revenue_report")
```

```text
+-------+-------+
|product|revenue|
+-------+-------+
| Laptop| 165000|
|  Phone| 140000|
| Tablet| 105000|
+-------+-------+
```

---

### 10. Common integration mistakes

| Mistake | Effect | Fix |
| --- | --- | --- |
| Forgetting `import spark.implicits._` | `toDF` not found | Import it at the start |
| Forgetting `.toInt` while parsing | Numbers become strings | Convert numeric fields |
| Registering a view before the DataFrame exists | Error | Create the DataFrame first |
| Using a view name different from the SQL table name | `TABLE_OR_VIEW_NOT_FOUND` | Check `spark.catalog.listTables()` |
| Using `createTempView()` twice | Exception | `createOrReplaceTempView()` |
| Double quotes inside SQL strings | Scala syntax error | Single quotes for SQL text |
| CSV read without `header` | Header row becomes data | `.option("header","true")` |
| Writing to an existing output folder | Error | `.mode("overwrite")` |
| Windows backslashes | Path fails | Use `C:/sparkdata/...` |
| Forgetting `ORDER BY` | Random order | Add `ORDER BY` whenever order matters |
| Unqualified column in a join | `AMBIGUOUS_REFERENCE` | Use `alias.column` |
| `COUNT(*)` after LEFT JOIN | Wrong counts | `COUNT(right.col)` |

---

### 11. Master template — keep this in your head

```scala
// ---------- 0. SETUP ----------
import spark.implicits._
import org.apache.spark.sql.functions._
import org.apache.spark.sql.Row
import org.apache.spark.sql.types._

// ---------- 1. DATA → RDD → DATAFRAME ----------
val lines = sc.textFile("C:/sparkdata/students.txt")
val df = lines.map(_.split(","))
  .map(p => (p(0).toInt, p(1), p(2), p(3).toInt, p(4).toInt))
  .toDF("student_id", "name", "department", "age", "marks")

// ---------- 2. VIEW ----------
df.createOrReplaceTempView("students")

// ---------- 3. QUERIES ----------
spark.sql("SELECT * FROM students").show()
spark.sql("SELECT name, marks FROM students WHERE marks > 80 ORDER BY marks DESC").show()
spark.sql("SELECT name FROM students WHERE name LIKE 'A%'").show()
spark.sql("SELECT department, COUNT(*) AS n, AVG(marks) AS avg_marks FROM students GROUP BY department ORDER BY department").show()
spark.sql("SELECT department, AVG(marks) AS a FROM students GROUP BY department HAVING AVG(marks) > 70").show()
spark.sql("SELECT s.name, d.dept_head FROM students s JOIN departments d ON s.department = d.department").show()

// ---------- 4. SAVE ----------
df.write.mode("overwrite").option("header", "true").csv("C:/sparkdata/out")
```

**Phase 8 completed**

---

> **Next → Phase 9: Unit IV Final Revision**: complete syntax cheat sheet, practical templates, common errors and exactly 30 viva questions. File: `9_Unit4_Final_Revision_30_Viva.md`
