# Unit IV — Phase 3: Converting RDDs into DataFrames

> **Continuing from Phase 2.** In Phase 2 we learned DataFrames from scratch. Now we cover the syllabus line **"methods to convert RDDs to DataFrames"**, which is an explicit examinable topic.
>
> Every command in this file was run in a real `spark-shell` (Spark 3.5.1, Scala 2.12), so the outputs shown are genuine. Minor formatting can differ slightly on other Spark versions.

---

## Phase 3 — Roadmap

| # | Topic |
| --- | --- |
| 1 | Why do we convert RDD to DataFrame? |
| 2 | Our starting RDD |
| 3 | Method 1 — `toDF()` without column names |
| 4 | Method 2 — `toDF()` with column names |
| 5 | Method 3 — `createDataFrame(rdd)` |
| 6 | Method 4 — Case class (reflection) |
| 7 | Method 5 — `Row` + `StructType` (programmatic schema) |
| 8 | Which method should I use? |
| 9 | Converting a text file → RDD → DataFrame |
| 10 | DataFrame → RDD (the reverse direction) |
| 11 | Working with `Row` objects |
| 12 | Schema inference and `nullable` |
| 13 | Common errors |
| 14 | Practical scenarios |
| 15 | Viva questions |
| 16 | Master revision |

---

### 1. Why do we convert RDD to DataFrame?

In Unit III we worked with RDDs:

```scala
val rdd = sc.parallelize(List(("Alice", 85), ("Bob", 72)))
```

An RDD is just a distributed collection of **objects**. Spark has no idea what is inside those objects. To Spark, `("Alice", 85)` is an opaque tuple.

A DataFrame is different. It has:

```text
Named columns     → name, marks
Data types        → string, integer
A schema          → structure Spark understands
```

Because Spark understands the structure, it can:

1. Let us write **SQL** on the data.
2. Use the **Catalyst optimizer** to plan queries smartly.
3. Store data more compactly and process it faster.
4. Give readable column-based operations (`select`, `filter`, `groupBy`).

**Typical real-world flow**

```text
Raw text / log / file
        |
        ↓
      RDD         (parse lines with map, split, filter)
        |
        ↓
   DataFrame      (give it structure)
        |
        ↓
   SQL queries    (analysis and reports)
```

> **Exam point:** RDD gives low-level control but no structure. DataFrame adds structure (schema), which enables SQL and optimization. Converting between them lets us use the best of both.

---

### 2. Our starting RDD

Open Spark Shell:

```powershell
spark-shell
```

Import what we need:

```scala
import spark.implicits._
import org.apache.spark.sql.functions._
```

Create an RDD of student tuples. Every tuple is `(student_id, name, department, age, marks)`:

```scala
val studentRDD = sc.parallelize(Seq(
  (101, "Alice", "CSE", 21, 85),
  (102, "Bob", "ECE", 22, 72),
  (103, "Charlie", "CSE", 21, 91),
  (104, "David", "IT", 23, 65),
  (105, "Emma", "ECE", 22, 88)
))
```

Check it:

```scala
studentRDD.take(2)
```

**Output**

```text
res0: Array[(Int, String, String, Int, Int)] = Array((101,Alice,CSE,21,85), (102,Bob,ECE,22,72))
```

Notice the type: `RDD[(Int, String, String, Int, Int)]`. This is an RDD of 5-field tuples. There are no column names anywhere.

---

### 3. Method 1 — `toDF()` without column names

```scala
val df1 = studentRDD.toDF()
```

**Why does this work?** Because we ran `import spark.implicits._`. That import adds a `toDF()` method to RDDs of tuples and case classes.

```scala
df1.show(2)
```

**Output**

```text
+---+-----+---+---+---+
| _1|   _2| _3| _4| _5|
+---+-----+---+---+---+
|101|Alice|CSE| 21| 85|
|102|  Bob|ECE| 22| 72|
+---+-----+---+---+---+
only showing top 2 rows
```

```scala
df1.printSchema()
```

**Output**

```text
root
 |-- _1: integer (nullable = false)
 |-- _2: string (nullable = true)
 |-- _3: string (nullable = true)
 |-- _4: integer (nullable = false)
 |-- _5: integer (nullable = false)
```

**Observation**

- Spark automatically names the columns `_1`, `_2`, `_3`, ... following tuple field positions.
- Data types were **inferred** from the tuple types.
- This works but is not readable. Real work needs proper column names.

---

### 4. Method 2 — `toDF("col1", "col2", ...)` with column names

```scala
val df2 = studentRDD.toDF("student_id", "name", "department", "age", "marks")
```

```scala
df2.show()
```

**Output**

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
```

```scala
df2.printSchema()
```

**Output**

```text
root
 |-- student_id: integer (nullable = false)
 |-- name: string (nullable = true)
 |-- department: string (nullable = true)
 |-- age: integer (nullable = false)
 |-- marks: integer (nullable = false)
```

**Syntax to remember**

```scala
val df = rdd.toDF("column1", "column2", "column3")
```

**Rules**

1. The RDD must contain **tuples** (or case classes).
2. The number of names must **exactly match** the number of tuple fields.
3. `import spark.implicits._` must be present.

> **This is the fastest and most common method in practicals.**

---

### 5. Method 3 — `spark.createDataFrame(rdd)`

`createDataFrame()` is a method of `SparkSession`:

```scala
val df3 = spark.createDataFrame(studentRDD)
```

Just like `toDF()` without names, this gives `_1`, `_2`, ... columns. To rename them:

```scala
val df3 = spark.createDataFrame(studentRDD)
  .toDF("student_id", "name", "department", "age", "marks")
```

```scala
df3.show(2)
```

**Output**

```text
+----------+-----+----------+---+-----+
|student_id| name|department|age|marks|
+----------+-----+----------+---+-----+
|       101|Alice|       CSE| 21|   85|
|       102|  Bob|       ECE| 22|   72|
+----------+-----+----------+---+-----+
only showing top 2 rows
```

**Difference from `toDF()`**

| `rdd.toDF()` | `spark.createDataFrame(rdd)` |
| --- | --- |
| Needs `spark.implicits._` | Does not need implicits for this form |
| Shorter and more convenient | Slightly longer |
| Called on the RDD | Called on the SparkSession |

`createDataFrame()` also has more powerful forms (with `Row` + schema) which we will see in Method 5.

---

### 6. Method 4 — Using a case class (reflection)

A **case class** lets us name each field directly in code, so Spark can read the column names automatically.

**Step 1 — Define the case class**

```scala
case class Student(student_id: Int, name: String, department: String, age: Int, marks: Int)
```

**Step 2 — Convert tuples into Student objects**

```scala
val caseRDD = studentRDD.map(t => Student(t._1, t._2, t._3, t._4, t._5))
```

The type is now `RDD[Student]`.

**Step 3 — Convert to DataFrame**

```scala
val df4 = caseRDD.toDF()
```

```scala
df4.printSchema()
```

**Output**

```text
root
 |-- student_id: integer (nullable = false)
 |-- name: string (nullable = true)
 |-- department: string (nullable = true)
 |-- age: integer (nullable = false)
 |-- marks: integer (nullable = false)
```

```scala
df4.show(2)
```

**Output**

```text
+----------+-----+----------+---+-----+
|student_id| name|department|age|marks|
+----------+-----+----------+---+-----+
|       101|Alice|       CSE| 21|   85|
|       102|  Bob|       ECE| 22|   72|
+----------+-----+----------+---+-----+
only showing top 2 rows
```

The column names came **automatically** from the case class field names. We did not pass any names to `toDF()`.

The same works with `createDataFrame`:

```scala
val df4b = spark.createDataFrame(caseRDD)
df4b.columns
```

**Output**

```text
res8: Array[String] = Array(student_id, name, department, age, marks)
```

**Why is it called reflection-based?** Spark uses Scala reflection to look inside the case class and discover its field names and types automatically.

> **Exam point:** Case class method = schema is inferred using **reflection** on the case class. Good when the structure is known and fixed before running.

---

### 7. Method 5 — `Row` + `StructType` (programmatic schema)

This is the **most flexible** method. We build the schema ourselves, in code. It is also the method examiners love asking about.

It has three steps:

```text
Step 1 → Convert RDD elements into Row objects
Step 2 → Define the schema using StructType
Step 3 → Combine them with spark.createDataFrame(rowRDD, schema)
```

**Imports**

```scala
import org.apache.spark.sql.Row
import org.apache.spark.sql.types._
```

**Step 1 — Create an RDD[Row]**

```scala
val rowRDD = studentRDD.map(t => Row(t._1, t._2, t._3, t._4, t._5))
```

```scala
rowRDD.take(1)
```

**Output**

```text
res9: Array[org.apache.spark.sql.Row] = Array([101,Alice,CSE,21,85])
```

A `Row` is a generic record: just values in order, with no names yet.

**Step 2 — Define the schema**

```scala
val schema = StructType(Array(
  StructField("student_id", IntegerType, false),
  StructField("name", StringType, true),
  StructField("department", StringType, true),
  StructField("age", IntegerType, true),
  StructField("marks", IntegerType, true)
))
```

**Understand the pieces**

| Piece | Meaning |
| --- | --- |
| `StructType` | The whole schema: a list of fields |
| `StructField` | Describes one column |
| `"student_id"` | Column name |
| `IntegerType` | Column data type |
| `false` / `true` | `nullable`: can the column contain null? |

```text
StructType
   |
   +-- StructField("student_id", IntegerType, false)
   +-- StructField("name",       StringType,  true)
   +-- StructField("department", StringType,  true)
   +-- StructField("age",        IntegerType, true)
   +-- StructField("marks",      IntegerType, true)
```

**Common data types**

| Spark type | Scala meaning |
| --- | --- |
| `IntegerType` | Int |
| `LongType` | Long |
| `DoubleType` | Double |
| `StringType` | String |
| `BooleanType` | Boolean |
| `DateType` | Date |
| `TimestampType` | Timestamp |

**Step 3 — Create the DataFrame**

```scala
val df5 = spark.createDataFrame(rowRDD, schema)
```

```scala
df5.show()
```

**Output**

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
```

```scala
df5.printSchema()
```

**Output**

```text
root
 |-- student_id: integer (nullable = false)
 |-- name: string (nullable = true)
 |-- department: string (nullable = true)
 |-- age: integer (nullable = true)
 |-- marks: integer (nullable = true)
```

Notice that `age` and `marks` are now `nullable = true` because **we** declared them so in the schema. This is the difference between inferred and explicit schemas.

**When to use this method**

- The structure is decided at runtime (for example, column names come from a file).
- You need exact control over types and nullability.
- Your data does not fit a case class.

> **Viva line:** The programmatic schema method uses `Row` objects, a `StructType` schema, and `spark.createDataFrame(rowRDD, schema)`.

---

### 8. Which method should I use?

| Method | Code | Column names | Best for |
| --- | --- | --- | --- |
| 1. `toDF()` | `rdd.toDF()` | `_1`, `_2`... | Quick checking only |
| 2. `toDF(names)` | `rdd.toDF("a","b")` | You give them | **Fastest everyday practical** |
| 3. `createDataFrame(rdd)` | `spark.createDataFrame(rdd)` | `_1`, `_2`... | Equivalent to Method 1 |
| 4. Case class | `rdd.map(...).toDF()` | From the class | Fixed, known structure |
| 5. `Row` + `StructType` | `createDataFrame(rowRDD, schema)` | From the schema | Full control, runtime schema |

```text
Quick practical?             → toDF("names")
Structure fixed in code?     → case class
Need exact control of types? → Row + StructType
```

---

### 9. Text file → RDD → DataFrame (the classic practical)

Suppose `C:/sparkdata/students.txt` contains:

```text
101,Alice,CSE,21,85
102,Bob,ECE,22,72
103,Charlie,CSE,21,91
104,David,IT,23,65
105,Emma,ECE,22,88
```

**Step 1 — Read as an RDD of lines**

```scala
val lines = sc.textFile("C:/sparkdata/students.txt")
```

Each element is one `String` like `"101,Alice,CSE,21,85"`.

**Step 2 — Split and convert types**

*Option A — parse into tuples, then `toDF`*

```scala
val tupleRDD = lines.map(_.split(",")).map(p =>
  (p(0).toInt, p(1), p(2), p(3).toInt, p(4).toInt)
)
val df7 = tupleRDD.toDF("student_id", "name", "department", "age", "marks")
```

```scala
df7.show(2)
```

**Output**

```text
+----------+-----+----------+---+-----+
|student_id| name|department|age|marks|
+----------+-----+----------+---+-----+
|       101|Alice|       CSE| 21|   85|
|       102|  Bob|       ECE| 22|   72|
+----------+-----+----------+---+-----+
only showing top 2 rows
```

*Option B — parse into Rows, apply the schema*

```scala
val parsed = lines.map(_.split(",")).map(p =>
  Row(p(0).toInt, p(1), p(2), p(3).toInt, p(4).toInt)
)
val df6 = spark.createDataFrame(parsed, schema)
df6.show()
```

**Output**

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
```

**What happens line by line**

```text
"101,Alice,CSE,21,85"
        |
        | split(",")
        ↓
Array("101","Alice","CSE","21","85")      ← all Strings
        |
        | p(0).toInt, p(3).toInt, p(4).toInt
        ↓
(101, "Alice", "CSE", 21, 85)             ← proper types
        |
        | toDF(...)
        ↓
DataFrame row
```

**Important: `.toInt` matters**

If we forget `.toInt`:

```scala
val dfBad = lines.map(_.split(",")).map(p => (p(0), p(1))).toDF("id", "name")
dfBad.printSchema()
```

**Output**

```text
root
 |-- id: string (nullable = true)
 |-- name: string (nullable = true)
```

`id` became a **string**. Sorting, comparing and averaging would then behave like text, not numbers.

> **Windows reminder:** In Spark Shell on Windows, use `C:/sparkdata/students.txt` (forward slashes) or `C:\\sparkdata\\students.txt` (double backslash).

---

### 10. DataFrame → RDD (the reverse direction)

Every DataFrame has an `.rdd` property:

```scala
val backRDD = df7.rdd
```

```scala
backRDD.first()
```

**Output**

```text
res15: org.apache.spark.sql.Row = [101,Alice,CSE,21,85]
```

The result is `RDD[Row]`, not an RDD of tuples. To get normal Scala values, we extract fields from each `Row`:

```scala
backRDD.map(r => r.getString(1)).collect()
```

**Output**

```text
res16: Array[String] = Array(Alice, Bob, Charlie, David, Emma)
```

By column name:

```scala
backRDD.map(r => r.getAs[Int]("marks")).collect()
```

**Output**

```text
res17: Array[Int] = Array(85, 72, 91, 65, 88)
```

Convert to a pair RDD, so that Unit III operations apply again:

```scala
val pairRDD = df7.select("name", "marks").rdd.map(r => (r.getString(0), r.getInt(1)))
pairRDD.collect()
```

**Output**

```text
res21: Array[(String, Int)] = Array((Alice,85), (Bob,72), (Charlie,91), (David,65), (Emma,88))
```

**When do we go back to RDD?** When we need a low-level RDD operation that is awkward in DataFrames, such as custom pair-RDD logic with `reduceByKey()`.

```text
RDD  ──── toDF() / createDataFrame() ───▶  DataFrame
RDD  ◀──────────── .rdd ──────────────────  DataFrame
```

---

### 11. Working with `Row` objects

A `Row` is one record. We can pick its values in several ways:

```scala
val r0 = df7.first()
```

```scala
r0.getInt(0)
```

```text
res18: Int = 101
```

```scala
r0(1)
```

```text
res19: Any = Alice
```

| Access style | Example | Returns |
| --- | --- | --- |
| By position, typed | `r.getInt(0)`, `r.getString(1)` | Exact type |
| By position, untyped | `r(1)` | `Any` |
| By name, typed | `r.getAs[Int]("marks")` | Exact type |

> **Remember:** Row positions start from **0**.

---

### 12. Schema inference and `nullable`

Spark can get a schema in two ways:

| Inferred schema | Explicit schema |
| --- | --- |
| Spark works out types from data (tuples, case class) | We define it with `StructType` |
| Quick | Exact and controlled |
| Primitive tuple fields come out `nullable = false` | We choose `nullable` for every field |

**Observation from our runs**

- Tuple fields such as `Int` appeared as `nullable = false`, while `String` was `nullable = true`.
- With an explicit schema we declared `age` and `marks` as `nullable = true`.

For a beginner practical, you rarely need to worry about `nullable`. Just know that it exists and that `StructField`'s third parameter controls it.

---

### 13. Common errors

#### Error 1 — Wrong number of column names

```scala
studentRDD.toDF("a", "b").show()
```

**Error**

```text
java.lang.IllegalArgumentException: requirement failed: The number of columns doesn't match.
Old column names (5): _1, _2, _3, _4, _5
New column names (2): a, b
```

**Fix:** Provide exactly as many names as there are fields (5 here).

#### Error 2 — Forgetting `import spark.implicits._`

Without it, `rdd.toDF(...)` gives an error such as *"value toDF is not a member of org.apache.spark.rdd.RDD"*.

**Fix:** Run `import spark.implicits._` first. (Spark Shell often does this for you, but always write it in practicals.)

#### Error 3 — Forgetting `.toInt`

Numbers stay strings, so sorting and arithmetic misbehave. See Section 9.

#### Error 4 — Schema does not match the Row data

If a `Row` holds a `String` where the schema says `IntegerType`, the error appears **when an action runs** (such as `show()`), not when you create the DataFrame. This is lazy evaluation at work.

**Fix:** Make sure that `Row(...)` values match the order and types of the `StructType`.

#### Error 5 — `ArrayIndexOutOfBoundsException` while parsing

This happens when a line has fewer fields than expected (blank lines or bad records). Filter such lines first:

```scala
val clean = lines.filter(_.trim.nonEmpty)
```

---

### 14. Practical scenarios — Phase 3 practice

Use `studentRDD` from Section 2.

#### Practical 1 — Convert the RDD to a DataFrame with proper names

```scala
val df = studentRDD.toDF("student_id", "name", "department", "age", "marks")
df.show()
```

#### Practical 2 — Display the schema

```scala
df.printSchema()
```

#### Practical 3 — Convert using a case class

```scala
case class Student(student_id: Int, name: String, department: String, age: Int, marks: Int)
val sDF = studentRDD.map(t => Student(t._1, t._2, t._3, t._4, t._5)).toDF()
sDF.show()
```

#### Practical 4 — Convert using `Row` + `StructType`

```scala
import org.apache.spark.sql.Row
import org.apache.spark.sql.types._

val rowRDD = studentRDD.map(t => Row(t._1, t._2, t._3, t._4, t._5))
val schema = StructType(Array(
  StructField("student_id", IntegerType, false),
  StructField("name", StringType, true),
  StructField("department", StringType, true),
  StructField("age", IntegerType, true),
  StructField("marks", IntegerType, true)
))
val rDF = spark.createDataFrame(rowRDD, schema)
rDF.show()
```

#### Practical 5 — Text file → DataFrame

```scala
val lines = sc.textFile("C:/sparkdata/students.txt")
val fileDF = lines.map(_.split(","))
  .map(p => (p(0).toInt, p(1), p(2), p(3).toInt, p(4).toInt))
  .toDF("student_id", "name", "department", "age", "marks")
fileDF.show()
```

#### Practical 6 — DataFrame back to an RDD and word-count-style aggregation

```scala
val deptMarks = fileDF.rdd.map(r => (r.getString(2), r.getInt(4)))
deptMarks.reduceByKey(_ + _).collect()
```

**Output (order may vary)**

```text
Array((CSE,176), (ECE,160), (IT,65))
```

#### Practical 7 — Convert, filter and show

```scala
fileDF.filter(col("marks") > 80).show()
```

---

### 15. Phase 3 — viva questions and answers

#### Q1. Why do we convert an RDD into a DataFrame?

A DataFrame has a schema, which lets Spark use SQL and the Catalyst optimizer. An RDD has no structure that Spark can understand.

#### Q2. What are the methods to convert an RDD into a DataFrame?

1. `toDF()` (with or without column names)
2. `spark.createDataFrame(rdd)`
3. Case class (reflection-based)
4. `Row` + `StructType` with `spark.createDataFrame(rowRDD, schema)`

#### Q3. What does `import spark.implicits._` do?

It enables implicit conversions, including the `toDF()` method on RDDs of tuples and case classes.

#### Q4. What column names do we get from `toDF()` without arguments on a tuple RDD?

`_1`, `_2`, `_3`, and so on.

#### Q5. What is a case class?

A Scala class designed for data modelling. Spark uses reflection on its fields to infer the DataFrame schema.

#### Q6. What is `StructType`?

A class representing the full schema of a DataFrame as a list of `StructField` objects.

#### Q7. What is `StructField`?

It describes a single column: its name, data type and whether it can be null.

#### Q8. What is a `Row`?

A generic object that represents one record of a DataFrame.

#### Q9. When would you use the `Row` + `StructType` method?

When we need explicit control over the schema, or when the schema is only known at runtime.

#### Q10. How do you convert a DataFrame back to an RDD?

Using the `.rdd` property, which gives an `RDD[Row]`.

#### Q11. What is the difference between `toDF()` and `createDataFrame()`?

`toDF()` is called on an RDD (needs `spark.implicits._`); `createDataFrame()` is called on the `SparkSession` and also supports the `Row` + schema form.

#### Q12. What is the third parameter of `StructField`?

The `nullable` flag, which says whether the column may contain null values.

#### Q13. Why does `toDF("a","b")` fail on a 5-field RDD?

The number of names must match the number of fields exactly.

#### Q14. What happens if we forget `.toInt` while parsing a file?

The numeric column becomes a string column, so numeric operations behave incorrectly.

#### Q15. Is schema checking immediate when creating a DataFrame from `Row`s?

Often not. Mismatches usually appear when an action such as `show()` is executed.

---

### 16. Phase 3 — final master code revision

```scala
// Setup
import spark.implicits._
import org.apache.spark.sql.functions._
import org.apache.spark.sql.Row
import org.apache.spark.sql.types._

// 1. RDD of tuples
val studentRDD = sc.parallelize(Seq(
  (101, "Alice", "CSE", 21, 85),
  (102, "Bob", "ECE", 22, 72),
  (103, "Charlie", "CSE", 21, 91),
  (104, "David", "IT", 23, 65),
  (105, "Emma", "ECE", 22, 88)
))

// 2. toDF with names (fastest)
val df2 = studentRDD.toDF("student_id", "name", "department", "age", "marks")

// 3. createDataFrame
val df3 = spark.createDataFrame(studentRDD)
  .toDF("student_id", "name", "department", "age", "marks")

// 4. Case class
case class Student(student_id: Int, name: String, department: String, age: Int, marks: Int)
val df4 = studentRDD.map(t => Student(t._1, t._2, t._3, t._4, t._5)).toDF()

// 5. Row + StructType
val rowRDD = studentRDD.map(t => Row(t._1, t._2, t._3, t._4, t._5))
val schema = StructType(Array(
  StructField("student_id", IntegerType, false),
  StructField("name", StringType, true),
  StructField("department", StringType, true),
  StructField("age", IntegerType, true),
  StructField("marks", IntegerType, true)
))
val df5 = spark.createDataFrame(rowRDD, schema)

// 6. File → RDD → DataFrame
val lines = sc.textFile("C:/sparkdata/students.txt")
val df7 = lines.map(_.split(","))
  .map(p => (p(0).toInt, p(1), p(2), p(3).toInt, p(4).toInt))
  .toDF("student_id", "name", "department", "age", "marks")

// 7. DataFrame → RDD
val backRDD = df7.rdd
backRDD.map(r => r.getString(1)).collect()
```

**Quick-recall card**

| Need | Syntax |
| --- | --- |
| Tuple RDD → DF | `rdd.toDF("a","b",...)` |
| RDD → DF (session) | `spark.createDataFrame(rdd)` |
| Case class RDD → DF | `rdd.map(...).toDF()` |
| Custom schema | `spark.createDataFrame(rowRDD, schema)` |
| DF → RDD | `df.rdd` |
| Read a Row value | `r.getString(1)`, `r.getInt(0)`, `r.getAs[Int]("marks")` |

**Phase 3 completed**

---

> **Next → Phase 4: Running SQL Queries Using Spark SQL** (temporary views, global temporary views, `spark.sql()` and query execution). File: `4_Unit4_Temp_Views_SQL_Queries.md`
