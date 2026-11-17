# Unit IV — Phase 4: Running SQL Queries Using Spark SQL

> **Continuing from Phase 3.** We now have DataFrames from several sources. In this phase we learn the core idea of the unit: **how to run real SQL queries on a DataFrame** using temporary views and `spark.sql()`.
>
> All commands and outputs were verified in a real `spark-shell` (Spark 3.5.1). In Spark versions before 3.4, null values print as `null` instead of `NULL`.

---

## Phase 4 — Roadmap

| # | Topic |
| --- | --- |
| 1 | Why do we need a view? |
| 2 | Creating a temporary view |
| 3 | Running SQL with `spark.sql()` |
| 4 | The result is a DataFrame |
| 5 | SELECT basics: columns, aliases, expressions |
| 6 | Writing multi-line SQL |
| 7 | `createTempView()` vs `createOrReplaceTempView()` |
| 8 | Global temporary views |
| 9 | Session scope: temp view vs global temp view |
| 10 | Inspecting and managing views |
| 11 | Chaining: SQL result → new view |
| 12 | Using Scala variables inside SQL |
| 13 | SQL vs DataFrame API |
| 14 | Common errors |
| 15 | Practical scenarios |
| 16 | Viva questions |
| 17 | Master revision |

---

### 1. Why do we need a view?

SQL queries run on **tables**. A DataFrame is a Scala object, so SQL does not know it by name.

```scala
spark.sql("SELECT * FROM studentsDF")   // ❌ SQL cannot see the Scala variable
```

We must **register** the DataFrame under a name that SQL can use. That registered name is a **view**.

```text
Scala variable            SQL name
studentsDF   ──────▶   "students"   (temporary view)
                              |
                              ↓
                  spark.sql("SELECT ... FROM students")
```

A **temporary view** is:

- a *name* given to a DataFrame so SQL can query it,
- **not** a physical copy of the data,
- **not** saved permanently. It disappears when the Spark session ends.

---

### 2. Creating a temporary view

**Setup**

```scala
import spark.implicits._
import org.apache.spark.sql.functions._

val studentsDF = Seq(
  (101, "Alice", "CSE", 21, 85),
  (102, "Bob", "ECE", 22, 72),
  (103, "Charlie", "CSE", 21, 91),
  (104, "David", "IT", 23, 65),
  (105, "Emma", "ECE", 22, 88)
).toDF("student_id", "name", "department", "age", "marks")
```

**Register the view**

```scala
studentsDF.createOrReplaceTempView("students")
```

Now `students` is a table-like name that SQL understands.

**Syntax**

```scala
dataFrame.createOrReplaceTempView("view_name")
```

---

### 3. Running SQL with `spark.sql()`

```scala
val result = spark.sql("SELECT * FROM students")
```

```scala
result.show()
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

**What happened?**

```text
spark.sql("...")
      |
      | parse the SQL text
      ↓
Catalyst analyses and optimizes
      |
      ↓
A new DataFrame is returned  (still lazy)
      |
      | show()  ← action triggers actual execution
      ↓
Output on screen
```

> **Key point:** `spark.sql()` is a **transformation-like** call. It returns a DataFrame and does not run the job until an action (`show`, `collect`, `count`) is called.

---

### 4. The result of `spark.sql()` is a DataFrame

Because the result is a DataFrame, everything from Phase 2 still works on it:

```scala
val r = spark.sql("SELECT * FROM students")
r.printSchema()
r.count()
r.collect()
```

```text
root
 |-- student_id: integer (nullable = false)
 |-- name: string (nullable = true)
 |-- department: string (nullable = true)
 |-- age: integer (nullable = false)
 |-- marks: integer (nullable = false)
```

This is why Spark SQL is powerful: **SQL and the DataFrame API are interchangeable**.

---

### 5. SELECT basics

#### 5.1 Select specific columns

```scala
spark.sql("SELECT name, marks FROM students").show()
```

**Output**

```text
+-------+-----+
|   name|marks|
+-------+-----+
|  Alice|   85|
|    Bob|   72|
|Charlie|   91|
|  David|   65|
|   Emma|   88|
+-------+-----+
```

#### 5.2 Column aliases using `AS`

```scala
spark.sql("SELECT name AS student_name, marks AS score FROM students").show()
```

**Output**

```text
+------------+-----+
|student_name|score|
+------------+-----+
|       Alice|   85|
|         Bob|   72|
|     Charlie|   91|
|       David|   65|
|        Emma|   88|
+------------+-----+
```

#### 5.3 A simple WHERE

```scala
spark.sql("SELECT name, marks FROM students WHERE marks > 80").show()
```

**Output**

```text
+-------+-----+
|   name|marks|
+-------+-----+
|  Alice|   85|
|Charlie|   91|
|   Emma|   88|
+-------+-----+
```

> We study WHERE, ORDER BY and wildcards in depth in Phase 5.

#### 5.4 Expressions in SELECT

```scala
spark.sql("SELECT name, marks, marks + 5 AS marks_with_bonus FROM students").show()
```

**Output**

```text
+-------+-----+----------------+
|   name|marks|marks_with_bonus|
+-------+-----+----------------+
|  Alice|   85|              90|
|    Bob|   72|              77|
|Charlie|   91|              96|
|  David|   65|              70|
|   Emma|   88|              93|
+-------+-----+----------------+
```

#### 5.5 Counting rows

```scala
spark.sql("SELECT COUNT(*) AS total FROM students").show()
```

**Output**

```text
+-----+
|total|
+-----+
|    5|
+-----+
```

> **SQL keywords are not case-sensitive:** `select`, `SELECT`, `Select` all work. Convention: keywords in UPPERCASE, names in lowercase.

---

### 6. Writing multi-line SQL

Long queries are hard to read on one line. Use Scala's **triple quotes**.

```scala
val q = """
  SELECT name, department, marks
  FROM students
  WHERE department = 'CSE'
"""
spark.sql(q).show()
```

**Output**

```text
+-------+----------+-----+
|   name|department|marks|
+-------+----------+-----+
|  Alice|       CSE|   85|
|Charlie|       CSE|   91|
+-------+----------+-----+
```

**Using `stripMargin` for tidy formatting**

```scala
val q2 = """
  |SELECT name, marks
  |FROM students
  |WHERE marks >= 85
""".stripMargin

spark.sql(q2).show()
```

**Output**

```text
+-------+-----+
|   name|marks|
+-------+-----+
|  Alice|   85|
|Charlie|   91|
|   Emma|   88|
+-------+-----+
```

**Quote rules**

| In SQL | Use |
| --- | --- |
| Text values like `'CSE'` | **Single** quotes inside the SQL |
| The whole SQL string in Scala | **Double** quotes `"..."` or triple quotes `"""..."""` |

```scala
spark.sql("SELECT * FROM students WHERE department = 'CSE'")   // ✅
spark.sql("SELECT * FROM students WHERE department = "CSE"")   // ❌ breaks the Scala string
```

---

### 7. `createTempView()` vs `createOrReplaceTempView()`

| Method | If the view name already exists |
| --- | --- |
| `createTempView("name")` | ❌ **Throws an error** |
| `createOrReplaceTempView("name")` | ✅ **Replaces** the old view |

```scala
studentsDF.createTempView("students")
```

**Error**

```text
org.apache.spark.sql.catalyst.analysis.TempTableAlreadyExistsException:
[TEMP_TABLE_OR_VIEW_ALREADY_EXISTS] Cannot create the temporary view `students` because it already exists.
```

> **Practical advice:** In `spark-shell` you re-run commands often, so **always use `createOrReplaceTempView()`**.

---

### 8. Global temporary views

A normal temp view belongs to **one SparkSession**. A **global temporary view** can be shared across sessions of the same Spark application.

**Create**

```scala
studentsDF.createOrReplaceGlobalTempView("global_students")
```

**Query: you must use the `global_temp.` prefix**

```scala
spark.sql("SELECT * FROM global_temp.global_students").show(2)
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

> `global_temp` is a special system database that Spark uses for global temporary views.

---

### 9. Session scope: temp view vs global temp view

Let's prove the difference by creating a second session.

```scala
val s2 = spark.newSession()
```

**Global view → visible in the new session**

```scala
s2.sql("SELECT COUNT(*) AS c FROM global_temp.global_students").show()
```

**Output**

```text
+---+
|  c|
+---+
|  5|
+---+
```

**Normal temp view → NOT visible in the new session**

```scala
s2.sql("SELECT COUNT(*) AS c FROM students").show()
```

**Error**

```text
org.apache.spark.sql.catalyst.ExtendedAnalysisException:
[TABLE_OR_VIEW_NOT_FOUND] The table or view `students` cannot be found.
```

**Comparison table**

| Temporary view | Global temporary view |
| --- | --- |
| Created by `createOrReplaceTempView()` | Created by `createOrReplaceGlobalTempView()` |
| Visible only in the **same session** | Visible across **all sessions** of the application |
| Queried as `students` | Queried as `global_temp.view_name` |
| Dies when the session ends | Dies when the **Spark application** ends |

```text
Spark Application
 ├── Session 1 (spark)   → temp view "students" (private)
 ├── Session 2 (s2)      → cannot see "students"
 └── global_temp         → shared by all sessions
```

> **Viva line:** Temporary views are session-scoped; global temporary views are application-scoped and live in the `global_temp` database.

---

### 10. Inspecting and managing views

**List what exists**

```scala
spark.catalog.listTables().show()
```

**Output**

```text
+--------+-------+---------+-----------+---------+-----------+
|    name|catalog|namespace|description|tableType|isTemporary|
+--------+-------+---------+-----------+---------+-----------+
|students|   NULL|       []|       NULL|TEMPORARY|       true|
+--------+-------+---------+-----------+---------+-----------+
```

**Using SQL**

```scala
spark.sql("SHOW TABLES").show()
```

**Output**

```text
+---------+---------+-----------+
|namespace|tableName|isTemporary|
+---------+---------+-----------+
|         | students|       true|
+---------+---------+-----------+
```

**Describe a view's structure**

```scala
spark.sql("DESCRIBE students").show()
```

**Output**

```text
+----------+---------+-------+
|  col_name|data_type|comment|
+----------+---------+-------+
|student_id|      int|   NULL|
|      name|   string|   NULL|
|department|   string|   NULL|
|       age|      int|   NULL|
|     marks|      int|   NULL|
+----------+---------+-------+
```

**Check if a view exists**

```scala
spark.catalog.tableExists("students")
```

```text
res18: Boolean = true
```

**Drop a view**

```scala
spark.catalog.dropTempView("students")
```

```text
res19: Boolean = true
```

After dropping, a query fails:

```scala
spark.sql("SELECT * FROM students").show()
```

```text
[TABLE_OR_VIEW_NOT_FOUND] The table or view `students` cannot be found.
```

Register it again for the rest of the phase:

```scala
studentsDF.createOrReplaceTempView("students")
```

**Quick card**

| Task | Command |
| --- | --- |
| List views | `spark.catalog.listTables().show()` |
| List via SQL | `spark.sql("SHOW TABLES").show()` |
| Structure | `spark.sql("DESCRIBE students").show()` |
| Exists? | `spark.catalog.tableExists("students")` |
| Drop | `spark.catalog.dropTempView("students")` |

---

### 11. Chaining: SQL result → new view

Since `spark.sql()` returns a DataFrame, we can register **that** as another view and query it again.

```scala
val toppers = spark.sql("SELECT * FROM students WHERE marks >= 85")
toppers.createOrReplaceTempView("toppers")

spark.sql("SELECT name FROM toppers ORDER BY name").show()
```

**Output**

```text
+-------+
|   name|
+-------+
|  Alice|
|Charlie|
|   Emma|
+-------+
```

```text
students ──SQL──▶ toppers (DataFrame) ──view──▶ "toppers" ──SQL──▶ final result
```

**Why this is useful:** It lets us build a multi-step analysis, with each step a readable query.

---

### 12. Using Scala variables inside SQL

Use **string interpolation** with the `s` prefix:

```scala
val minMarks = 80
spark.sql(s"SELECT name, marks FROM students WHERE marks > $minMarks").show()
```

**Output**

```text
+-------+-----+
|   name|marks|
+-------+-----+
|  Alice|   85|
|Charlie|   91|
|   Emma|   88|
+-------+-----+
```

For text values, keep the SQL single quotes around the variable:

```scala
val dept = "CSE"
spark.sql(s"SELECT name FROM students WHERE department = '$dept'").show()
```

**Output**

```text
+-------+
|   name|
+-------+
|  Alice|
|Charlie|
+-------+
```

> **Note:** `s"..."` injects the variable value into the text *before* Spark parses it. Do not build SQL this way from untrusted user input in real applications (SQL injection risk). For practicals it is fine.

---

### 13. SQL vs DataFrame API

Both give the same result because both go through Catalyst.

| Task | SQL | DataFrame API |
| --- | --- | --- |
| All rows | `SELECT * FROM students` | `studentsDF` |
| Columns | `SELECT name, marks FROM students` | `studentsDF.select("name","marks")` |
| Filter | `WHERE marks > 80` | `.filter(col("marks") > 80)` |
| Count | `SELECT COUNT(*) FROM students` | `studentsDF.count()` |
| Alias | `marks AS score` | `col("marks").alias("score")` |

**Proof they use the same engine**

```scala
spark.sql("SELECT name, marks FROM students WHERE marks > 80").explain()
```

The printed plan shows a `Filter` and a `Project` over the scan. The DataFrame version gives the same plan.

**Which should I use?**

| Use SQL when | Use the DataFrame API when |
| --- | --- |
| Queries are complex (joins, grouping) | You need programmatic, step-by-step logic |
| You know SQL well | You want compile-time checks of method names |
| The exam asks for "SQL queries" | The exam asks for DataFrame methods |

---

### 14. Common errors

#### Error 1 — Misspelled column

```scala
spark.sql("SELECT nme FROM students").show()
```

```text
[UNRESOLVED_COLUMN.WITH_SUGGESTION] A column or function parameter with name `nme` cannot be resolved.
Did you mean one of the following? [`name`, ...]
```

**Fix:** Check `df.columns` or `DESCRIBE students`.

#### Error 2 — Misspelled SQL keyword

```scala
spark.sql("SELEC * FROM students").show()
```

```text
org.apache.spark.sql.catalyst.parser.ParseException:
[PARSE_SYNTAX_ERROR] Syntax error at or near 'SELEC'.
```

#### Error 3 — View does not exist

```scala
spark.sql("SELECT * FROM student").show()    // missing the final 's'
```

```text
[TABLE_OR_VIEW_NOT_FOUND] The table or view `student` cannot be found.
```

**Fix:** Run `spark.catalog.listTables().show()` to see the real names.

#### Error 4 — Forgetting the action

```scala
spark.sql("SELECT * FROM students")      // returns a DataFrame, shows nothing
```

**Fix:** Add `.show()`.

#### Error 5 — Double quotes around SQL text values

Use `'CSE'` inside SQL, not `"CSE"`. See Section 6.

#### Error 6 — Re-creating a view with `createTempView()`

Use `createOrReplaceTempView()`. See Section 7.

---

### 15. Practical scenarios — Phase 4 practice

#### Practical 1 — Register a view and view all data

```scala
studentsDF.createOrReplaceTempView("students")
spark.sql("SELECT * FROM students").show()
```

#### Practical 2 — Names and marks only

```scala
spark.sql("SELECT name, marks FROM students").show()
```

#### Practical 3 — Students with marks above 80

```scala
spark.sql("SELECT name, marks FROM students WHERE marks > 80").show()
```

#### Practical 4 — Add 5 bonus marks as a new column

```scala
spark.sql("SELECT name, marks, marks + 5 AS marks_with_bonus FROM students").show()
```

#### Practical 5 — Count the students

```scala
spark.sql("SELECT COUNT(*) AS total FROM students").show()
```

#### Practical 6 — CSE students using a multi-line query

```scala
val q = """
  SELECT name, department, marks
  FROM students
  WHERE department = 'CSE'
"""
spark.sql(q).show()
```

#### Practical 7 — Create a global view and query it

```scala
studentsDF.createOrReplaceGlobalTempView("global_students")
spark.sql("SELECT * FROM global_temp.global_students").show()
```

#### Practical 8 — Two-step analysis using chaining

```scala
spark.sql("SELECT * FROM students WHERE marks >= 85").createOrReplaceTempView("toppers")
spark.sql("SELECT name FROM toppers ORDER BY name").show()
```

#### Practical 9 — Show all registered views

```scala
spark.catalog.listTables().show()
```

#### Practical 10 — Use a Scala variable in a query

```scala
val limit = 80
spark.sql(s"SELECT name FROM students WHERE marks > $limit").show()
```

---

### 16. Phase 4 — viva questions and answers

#### Q1. What is a temporary view?

A named, session-scoped handle for a DataFrame that lets us query it with SQL. It does not copy the data.

#### Q2. Why do we need to create a view before running SQL?

SQL needs a table-like name, and a Scala DataFrame variable is not visible to SQL.

#### Q3. Which method creates a temporary view?

`createOrReplaceTempView("name")` (or `createTempView("name")`).

#### Q4. What does `spark.sql()` return?

A DataFrame.

#### Q5. Is `spark.sql()` lazy?

Yes. Execution happens only when an action like `show()`, `count()` or `collect()` is called.

#### Q6. Difference between `createTempView()` and `createOrReplaceTempView()`?

`createTempView()` throws an exception if the view already exists; `createOrReplaceTempView()` replaces it.

#### Q7. What is a global temporary view?

A view shared across all sessions of one Spark application, stored in the `global_temp` database.

#### Q8. How do we query a global temporary view?

With the prefix `global_temp.`, for example `SELECT * FROM global_temp.global_students`.

#### Q9. What is the lifetime of a temporary view?

It lasts as long as the SparkSession that created it.

#### Q10. Does a temporary view store a copy of the data?

No. It is just a name for the DataFrame's logical plan.

#### Q11. How do we list all views?

`spark.catalog.listTables().show()` or `spark.sql("SHOW TABLES").show()`.

#### Q12. How do we remove a temporary view?

`spark.catalog.dropTempView("name")`.

#### Q13. How do we write a multi-line SQL query in Scala?

Using triple quotes `"""..."""`, optionally with `stripMargin`.

#### Q14. How do we use a Scala variable inside SQL?

With string interpolation: `spark.sql(s"... $variable ...")`.

#### Q15. Is the SQL and the DataFrame API result different?

No. Both are compiled by Catalyst into the same kind of optimized plan.

---

### 17. Phase 4 — final master code revision

```scala
// Setup
import spark.implicits._
import org.apache.spark.sql.functions._

val studentsDF = Seq(
  (101, "Alice", "CSE", 21, 85),
  (102, "Bob", "ECE", 22, 72),
  (103, "Charlie", "CSE", 21, 91),
  (104, "David", "IT", 23, 65),
  (105, "Emma", "ECE", 22, 88)
).toDF("student_id", "name", "department", "age", "marks")

// Register view
studentsDF.createOrReplaceTempView("students")

// Basic queries
spark.sql("SELECT * FROM students").show()
spark.sql("SELECT name, marks FROM students").show()
spark.sql("SELECT name AS student_name, marks AS score FROM students").show()
spark.sql("SELECT name, marks FROM students WHERE marks > 80").show()
spark.sql("SELECT COUNT(*) AS total FROM students").show()

// Multi-line
val q = """
  SELECT name, department, marks
  FROM students
  WHERE department = 'CSE'
"""
spark.sql(q).show()

// Global temp view
studentsDF.createOrReplaceGlobalTempView("global_students")
spark.sql("SELECT * FROM global_temp.global_students").show()

// Manage views
spark.catalog.listTables().show()
spark.sql("DESCRIBE students").show()
spark.catalog.dropTempView("students")

// Chaining
studentsDF.createOrReplaceTempView("students")
spark.sql("SELECT * FROM students WHERE marks >= 85").createOrReplaceTempView("toppers")
spark.sql("SELECT name FROM toppers").show()
```

**The 4-line flow to remember**

```scala
val df = data.toDF("a", "b")
df.createOrReplaceTempView("t")
spark.sql("SELECT ... FROM t ...").show()
```

**Phase 4 completed**

---

> **Next → Phase 5: SQL Filtering, Sorting and Wildcards** (WHERE, DISTINCT, ORDER BY, LIKE, `%`, `_`, LIMIT). File: `5_Unit4_SQL_Filtering_Sorting_Wildcards.md`
