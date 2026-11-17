# Unit IV — Phase 9: Final Revision

> **Final phase of Unit IV.** This is exam-mode revision, not another teaching pass. It contains the syllabus coverage map, the complete syntax cheat sheet, practical templates, the error table, **exactly 30 viva questions with answers**, and a last-minute sheet.
>
> All code in this unit was verified in a real `spark-shell` (Spark 3.5.1, Scala 2.12).

---

## 1. Syllabus coverage map

Your Unit IV syllabus line:

> *Running SQL queries using Spark SQL: importance and features of Spark SQL, methods to convert RDDs to DataFrames, concepts of Spark SQL, joins, groupby, orderby, aggregate functions, SQL wildcards.*

| Syllabus topic | Where it is covered |
| --- | --- |
| Importance and features of Spark SQL | Phase 1 |
| Concepts of Spark SQL (SparkSession, DataFrame, Dataset, Catalyst) | Phase 1, Phase 2 |
| Methods to convert RDDs to DataFrames | **Phase 3** |
| Running SQL queries (views, `spark.sql`) | **Phase 4** |
| ORDER BY | **Phase 5** |
| SQL wildcards | **Phase 5** |
| GROUP BY | **Phase 6** |
| Aggregate functions | **Phase 6** |
| Joins | **Phase 7** |
| Complete practical flow | **Phase 8** |
| Revision and viva | **Phase 9** (this file) |

**File index**

| File | Content |
| --- | --- |
| `1_Unit4_Spark_SQL_Fundamentals.md` | Phase 1 |
| `2_Unit4_DataFrames_Practical_Operations.md` | Phase 2 |
| `3_Unit4_RDD_to_DataFrame_Conversion.md` | Phase 3 |
| `4_Unit4_Temp_Views_SQL_Queries.md` | Phase 4 |
| `5_Unit4_SQL_Filtering_Sorting_Wildcards.md` | Phase 5 |
| `6_Unit4_GroupBy_Aggregate_Functions.md` | Phase 6 |
| `7_Unit4_Joins.md` | Phase 7 |
| `8_Unit4_Complete_Practical_Integration.md` | Phase 8 |
| `9_Unit4_Final_Revision_30_Viva.md` | Phase 9 |

---

## 2. The Unit IV mental model

```text
                         SPARK SQL
                             |
     ┌───────────────┬───────┴────────┬────────────────┐
     |               |                |                |
  SparkSession    DataFrame      Temp View         Catalyst
  (spark)       (rows+schema)   (SQL name)       (optimizer)
     |               |                |                |
  entry point   from RDD / Seq /   spark.sql("...")  plans the
                file (read)         returns DataFrame   fastest way
```

```text
 RDD ──toDF/createDataFrame──▶ DataFrame ──createOrReplaceTempView──▶ SQL view
                                   ▲                                     |
                                   └──────────── spark.sql() ◀───────────┘
```

**One-line summary**

> Spark SQL lets us process **structured data** using **SQL** or the **DataFrame API**, with the **Catalyst optimizer** planning execution, through the entry point **SparkSession**.

---

## 3. Complete syntax cheat sheet

### 3.1 Setup

```scala
import spark.implicits._
import org.apache.spark.sql.functions._
import org.apache.spark.sql.Row
import org.apache.spark.sql.types._
```

### 3.2 Creating DataFrames

```scala
val df = Seq((1,"A"),(2,"B")).toDF("id","name")              // from Seq
val df = rdd.toDF("a","b")                                    // from tuple RDD
val df = spark.createDataFrame(rdd)                           // _1, _2, ...
val df = caseClassRDD.toDF()                                  // from case class
val df = spark.createDataFrame(rowRDD, schema)                // Row + StructType
val df = spark.read.option("header","true")
                   .option("inferSchema","true").csv("path")  // from CSV
val rdd2 = df.rdd                                             // back to RDD[Row]
```

### 3.3 Schema

```scala
val schema = StructType(Array(
  StructField("id",   IntegerType, false),
  StructField("name", StringType,  true)
))
df.printSchema()
df.columns
```

### 3.4 Views

```scala
df.createOrReplaceTempView("t")              // session-scoped
df.createOrReplaceGlobalTempView("g")        // query as global_temp.g
spark.catalog.listTables().show()
spark.sql("DESCRIBE t").show()
spark.catalog.dropTempView("t")
```

### 3.5 SQL query skeleton

```sql
SELECT   [DISTINCT] columns / expressions / aggregates
FROM     table [alias]
[JOIN    other [alias] ON condition]
[WHERE   row condition]
[GROUP BY columns]
[HAVING  group condition]
[ORDER BY column [ASC|DESC]]
[LIMIT   n]
```

```scala
spark.sql("SELECT ... ").show()
```

### 3.6 Conditions

| Need | SQL |
| --- | --- |
| Equal / not equal | `=`  `<>` |
| Compare | `>` `<` `>=` `<=` |
| Combine | `AND` `OR` `NOT` |
| In a list | `IN ('a','b')` / `NOT IN` |
| Range (inclusive) | `BETWEEN 70 AND 90` |
| NULL check | `IS NULL` / `IS NOT NULL` |
| Default for NULL | `COALESCE(col, 0)` |
| If-else | `CASE WHEN ... THEN ... ELSE ... END` |

### 3.7 Wildcards

| Pattern | Meaning |
| --- | --- |
| `%` | any number of characters |
| `_` | exactly one character |
| `'A%'` | starts with A |
| `'%e'` | ends with e |
| `'%li%'` | contains li |
| `'_ob'` | 3 letters, ends in ob |
| `'D____'` | D + 4 more letters |
| `NOT LIKE` | negation |
| `LOWER(col) LIKE ...` / `ILIKE` | case-insensitive |

### 3.8 Aggregates and grouping

```sql
COUNT(*)  COUNT(col)  COUNT(DISTINCT col)
SUM(col)  AVG(col)  MIN(col)  MAX(col)

SELECT dept, COUNT(*), AVG(marks)
FROM students
WHERE marks > 50              -- filters rows (before grouping)
GROUP BY dept
HAVING AVG(marks) > 70        -- filters groups (after grouping)
ORDER BY dept;
```

### 3.9 Joins

```sql
FROM a JOIN b            ON a.k = b.k    -- INNER
FROM a LEFT JOIN b       ON a.k = b.k
FROM a RIGHT JOIN b      ON a.k = b.k
FROM a FULL OUTER JOIN b ON a.k = b.k
FROM a LEFT SEMI JOIN b  ON a.k = b.k
FROM a LEFT ANTI JOIN b  ON a.k = b.k
FROM a CROSS JOIN b
```

### 3.10 DataFrame API equivalents

```scala
df.select("a","b")                       df.filter(col("x") > 5)
df.where(col("x") === "CSE")             df.withColumn("new", col("x") + 1)
df.withColumnRenamed("old","new")        df.drop("c")
df.distinct()                            df.orderBy(col("x").desc)
df.limit(3)                              df.groupBy("g").count()
df.groupBy("g").agg(avg("x"), max("x"))  df.agg(sum("x"))
a.join(b, "k")                           a.join(b, Seq("k"), "left")
a.join(b, a("k") === b("k"), "inner")    a.crossJoin(b)
```

---

## 4. Practical master templates

**Template 1 — RDD → DataFrame → SQL**

```scala
val rdd = sc.parallelize(Seq(("Alice", 85), ("Bob", 72)))
val df = rdd.toDF("name", "marks")
df.createOrReplaceTempView("t")
spark.sql("SELECT * FROM t WHERE marks > 80").show()
```

**Template 2 — File → schema → DataFrame**

```scala
val lines = sc.textFile("C:/sparkdata/students.txt")
val rowRDD = lines.map(_.split(",")).map(p => Row(p(0).toInt, p(1), p(2), p(3).toInt, p(4).toInt))
val df = spark.createDataFrame(rowRDD, schema)
```

**Template 3 — Filter, sort, top-N**

```scala
spark.sql("SELECT name, marks FROM students WHERE marks > 70 ORDER BY marks DESC LIMIT 3").show()
```

**Template 4 — Wildcard search**

```scala
spark.sql("SELECT name FROM students WHERE name LIKE 'A%'").show()
```

**Template 5 — Group report**

```scala
spark.sql("SELECT department, COUNT(*) AS n, AVG(marks) AS avg_marks FROM students GROUP BY department ORDER BY department").show()
```

**Template 6 — HAVING**

```scala
spark.sql("SELECT department, AVG(marks) AS a FROM students GROUP BY department HAVING AVG(marks) > 70").show()
```

**Template 7 — Inner join**

```scala
spark.sql("SELECT s.name, d.dept_head FROM students s JOIN departments d ON s.department = d.department").show()
```

**Template 8 — Left join + anti pattern**

```scala
spark.sql("SELECT s.name FROM students s LEFT ANTI JOIN departments d ON s.department = d.department").show()
```

**Template 9 — Join + group**

```scala
spark.sql("""
  SELECT d.dept_head, COUNT(*) AS n, AVG(s.marks) AS avg_marks
  FROM students s JOIN departments d ON s.department = d.department
  GROUP BY d.dept_head
""").show()
```

**Template 10 — Subquery above average**

```scala
spark.sql("SELECT name, marks FROM students WHERE marks > (SELECT AVG(marks) FROM students)").show()
```

---

## 5. Common errors — one table

| Error / symptom | Cause | Fix |
| --- | --- | --- |
| `value toDF is not a member of RDD` | Missing implicits | `import spark.implicits._` |
| `The number of columns doesn't match` | Wrong count of names in `toDF` | Match the field count |
| Numbers behave like text | Forgot `.toInt` / no `inferSchema` | Convert types or set schema |
| `TABLE_OR_VIEW_NOT_FOUND` | View not created, misspelled, or other session | `listTables()`, check the name |
| `TEMP_TABLE_OR_VIEW_ALREADY_EXISTS` | `createTempView` twice | `createOrReplaceTempView` |
| `PARSE_SYNTAX_ERROR` | SQL typo | Re-read the SQL |
| `UNRESOLVED_COLUMN` | Misspelled column | `df.columns` |
| `MISSING_AGGREGATION` | Non-grouped column in SELECT | Add to GROUP BY or aggregate |
| Aggregate in `WHERE` | Wrong clause | `HAVING` or a subquery |
| `= NULL` returns nothing | NULL comparison | `IS NULL` |
| `LIKE '%a%'` misses "Alice" | LIKE is case-sensitive | `LOWER()` / `ILIKE` |
| `AMBIGUOUS_REFERENCE` | Same column in both tables | `alias.column` |
| LEFT JOIN count wrong | `COUNT(*)` counts NULL-padded rows | `COUNT(right.col)` |
| Output folder exists | Default write mode | `.mode("overwrite")` |
| Nothing displayed | Forgot the action | Add `.show()` |
| Windows path fails | Backslashes | `C:/sparkdata/...` |

---

## 6. Exactly 30 viva questions with answers

#### Q1. What is Spark SQL?

Spark SQL is Spark's module for processing **structured data**. It lets us query data using SQL or the DataFrame API, and uses the Catalyst optimizer for efficient execution.

#### Q2. What are the important features of Spark SQL?

- Works with structured and semi-structured data (CSV, JSON, Parquet, Hive tables).
- Supports both SQL queries and the DataFrame API.
- Automatic query optimization through Catalyst.
- Integrates with the rest of Spark (RDDs, MLlib, Streaming).
- Scales across a cluster with in-memory processing.

#### Q3. What is SparkSession, and how is it different from SparkContext?

`SparkSession` (`spark`) is the unified entry point for DataFrames and SQL. `SparkContext` (`sc`) is the older entry point for RDDs. A SparkSession contains and gives access to the SparkContext.

#### Q4. What is a DataFrame, and how does it differ from an RDD?

A DataFrame is a distributed collection of data organised into **named columns with a schema**. An RDD is a distributed collection of objects without a schema. Because a DataFrame has structure, Spark can optimize it and we can run SQL on it.

#### Q5. What is the difference between a DataFrame and a Dataset?

A Dataset is a distributed collection of **strongly typed** objects. A DataFrame is a Dataset of `Row` objects (`Dataset[Row]`) with named columns but no compile-time type per record.

#### Q6. What is the Catalyst optimizer?

Catalyst is Spark SQL's query optimizer. It analyses a query, rewrites it into a more efficient logical plan, and chooses a physical plan. It is neither the SQL parser nor the execution engine.

#### Q7. What are the methods to convert an RDD to a DataFrame?

1. `rdd.toDF()` / `rdd.toDF("col1", ...)`
2. `spark.createDataFrame(rdd)`
3. Case-class RDD with `toDF()` (reflection-based)
4. `Row` RDD + `StructType` with `spark.createDataFrame(rowRDD, schema)` (programmatic schema)

#### Q8. Why do we need `import spark.implicits._`, and what is the difference between `toDF()` and `createDataFrame()`?

The import adds `toDF()` to RDDs and Seqs of tuples and case classes. `toDF()` is called on the data and is the shortest form; `createDataFrame()` is called on the SparkSession and also supports an explicit `Row` + schema form.

#### Q9. Explain `StructType`, `StructField` and `Row`.

`StructType` is the full schema, a list of `StructField`s. Each `StructField` gives a column's name, data type and nullability. A `Row` is one record of values. Together they let us build a DataFrame with an explicit schema.

#### Q10. How do you convert a DataFrame back to an RDD?

Use the `.rdd` property, which returns `RDD[Row]`. Values are read with `r.getString(1)`, `r.getInt(0)` or `r.getAs[Int]("marks")`.

#### Q11. What is a temporary view and why do we need one?

A temporary view is a session-scoped name registered for a DataFrame so that SQL queries can refer to it. SQL cannot see Scala variables, and the view stores no copy of the data.

#### Q12. What is the difference between `createTempView`, `createOrReplaceTempView` and a global temporary view?

`createTempView` fails if the name exists; `createOrReplaceTempView` replaces it. A normal view is visible only in its own session. A global temporary view (`createOrReplaceGlobalTempView`) is shared across sessions of the application and is queried as `global_temp.name`.

#### Q13. What does `spark.sql()` return? Is it lazy?

It returns a **DataFrame**. It is lazy: the query executes only when an action such as `show()`, `count()` or `collect()` runs.

#### Q14. What is the difference between WHERE and HAVING?

WHERE filters individual rows **before** grouping and cannot use aggregate functions. HAVING filters groups **after** aggregation and can use aggregates. Example: `WHERE marks > 50` vs `HAVING AVG(marks) > 70`.

#### Q15. What is the logical execution order of a SQL query?

FROM (and JOIN) → WHERE → GROUP BY → HAVING → SELECT → ORDER BY → LIMIT.

#### Q16. Explain ORDER BY. What is its default order, and where do NULLs go?

`ORDER BY` sorts the result. The default is ascending (`ASC`); `DESC` reverses it, and multiple columns can be given, each with its own direction. In Spark SQL, NULLs come first in ascending order and last in descending order unless `NULLS FIRST/LAST` is specified.

#### Q17. How do you get the top 3 students by marks?

`SELECT name, marks FROM students ORDER BY marks DESC LIMIT 3`

#### Q18. What does DISTINCT do?

It removes duplicate rows from the result. With several columns, the combination of values must be unique.

#### Q19. What are SQL wildcards? Explain `%` and `_` with examples.

Wildcards are special characters used with `LIKE`. `%` matches any number of characters: `'A%'` = starts with A, `'%e'` = ends with e, `'%li%'` = contains li. `_` matches exactly one character: `'_ob'` matches Bob, `'D____'` matches 5-letter names starting with D.

#### Q20. Is `LIKE` case-sensitive in Spark SQL? How do you ignore case?

Yes, it is case-sensitive. For example `LIKE '%a%'` does not match "Alice". Use `LOWER(name) LIKE '%a%'` or `name ILIKE '%a%'` (Spark 3.3 and later).

#### Q21. How do you handle NULL values in SQL?

Test with `IS NULL` / `IS NOT NULL` (never `= NULL`, which matches nothing). Replace NULLs using `COALESCE(col, default)`. Comparisons such as `marks > 80` exclude NULL rows.

#### Q22. List the aggregate functions. How do they treat NULLs, and what is the difference between `COUNT(*)` and `COUNT(column)`?

`COUNT`, `SUM`, `AVG`, `MIN`, `MAX`. They ignore NULLs, except `COUNT(*)`, which counts all rows. `COUNT(column)` counts only non-null values, and `COUNT(DISTINCT column)` counts unique non-null values.

#### Q23. Explain GROUP BY and its main rule.

`GROUP BY` divides rows into groups with the same values in the grouping columns, and aggregates are computed per group. Every column in SELECT must be either in the GROUP BY clause or inside an aggregate function; otherwise Spark raises `MISSING_AGGREGATION`.

#### Q24. What are the types of joins in Spark SQL?

INNER, LEFT (OUTER), RIGHT (OUTER), FULL OUTER, LEFT SEMI, LEFT ANTI and CROSS joins.

#### Q25. Explain INNER JOIN and LEFT JOIN with an example.

INNER JOIN keeps only rows that match in both tables. LEFT JOIN keeps **all** rows from the left table and fills right-side columns with NULL when there is no match. With students and departments, Frank (department MECH, not in the departments table) disappears in an INNER JOIN but appears with `dept_head = NULL` in a LEFT JOIN.

#### Q26. What are LEFT SEMI and LEFT ANTI joins?

LEFT SEMI returns left rows that **have** a match in the right table (only left columns). LEFT ANTI returns left rows that have **no** match, which is useful for finding missing records.

#### Q27. What is a CROSS JOIN?

It pairs every row of one table with every row of the other, giving m × n rows (for example 6 × 4 = 24). It needs no join condition and should be used only for small tables.

#### Q28. What is the ambiguous column error in joins and how do we fix it?

It occurs when a column name exists in both tables and is referenced without qualification. Fix it by using table aliases, `s.department`, or in the DataFrame API by joining with `Seq("department")` or by selecting through `df("col")` or aliases.

#### Q29. What is a broadcast join?

A join strategy where the small table is copied to every executor so the large table does not need to be shuffled. It can be requested with the hint `/*+ BROADCAST(d) */` and Spark often chooses it automatically for small tables.

#### Q30. How do Spark SQL, the DataFrame API and RDD operations compare? When do you choose which?

SQL and the DataFrame API compile to the same optimized Catalyst plan, so performance is equivalent; choose SQL for complex joins and grouping and the DataFrame API for programmatic steps. RDD operations such as `reduceByKey` give low-level control but are not optimized by Catalyst. For example `reduceByKey(_ + _)` on (department, marks) equals `SELECT department, SUM(marks) ... GROUP BY department`. For structured data, DataFrames/SQL are preferred.

---

## 7. Last 10-minute cheat sheet

```scala
// SETUP
import spark.implicits._
import org.apache.spark.sql.functions._

// RDD → DF
val df = rdd.toDF("a","b")
val df = spark.createDataFrame(rowRDD, schema)

// VIEW + SQL
df.createOrReplaceTempView("t")
spark.sql("SELECT * FROM t").show()

// FILTER / SORT
spark.sql("SELECT * FROM t WHERE x > 5 AND y = 'A' ORDER BY x DESC LIMIT 3").show()

// WILDCARDS
spark.sql("SELECT * FROM t WHERE name LIKE 'A%'").show()    // starts with
spark.sql("SELECT * FROM t WHERE name LIKE '%e'").show()    // ends with
spark.sql("SELECT * FROM t WHERE name LIKE '%li%'").show()  // contains
spark.sql("SELECT * FROM t WHERE name LIKE '_ob'").show()   // one char + ob

// GROUP BY / HAVING
spark.sql("SELECT g, COUNT(*), AVG(x) FROM t GROUP BY g HAVING AVG(x) > 70 ORDER BY g").show()

// JOINS
spark.sql("SELECT a.n, b.m FROM a JOIN b ON a.k = b.k").show()            // inner
spark.sql("SELECT a.n, b.m FROM a LEFT JOIN b ON a.k = b.k").show()       // left
spark.sql("SELECT a.n FROM a LEFT ANTI JOIN b ON a.k = b.k").show()       // missing

// DATAFRAME API
df.filter(col("x") > 5).orderBy(desc("x")).show()
df.groupBy("g").agg(count("*"), avg("x")).show()
a.join(b, Seq("k"), "left").show()
```

**Ten things to remember**

1. `spark` is the SparkSession; `sc` is the SparkContext.
2. DataFrame = rows + named columns + schema.
3. `import spark.implicits._` unlocks `toDF()`.
4. Four RDD → DataFrame methods: `toDF`, `createDataFrame`, case class, `Row` + `StructType`.
5. SQL needs a **view**: `createOrReplaceTempView`.
6. `spark.sql()` returns a DataFrame and is lazy.
7. `%` = any characters, `_` = exactly one; LIKE is **case-sensitive**.
8. WHERE filters rows; HAVING filters groups.
9. Aggregates ignore NULLs (except `COUNT(*)`); use `IS NULL`, never `= NULL`.
10. LEFT JOIN keeps all left rows; ANTI JOIN finds the missing ones; qualify columns with aliases.

---

## 8. Unit IV status

```text
Phase 1  Spark SQL fundamentals                ✅
Phase 2  DataFrames practical operations       ✅
Phase 3  RDD → DataFrame conversion            ✅
Phase 4  Temp views and running SQL            ✅
Phase 5  Filtering, sorting, wildcards         ✅
Phase 6  GROUP BY and aggregate functions      ✅
Phase 7  Joins                                 ✅
Phase 8  Complete practical integration        ✅
Phase 9  Final revision + 30 viva              ✅
```

## ✅ UNIT IV — COMPLETE

> **Next → Unit V: Spark Streaming with Apache Kafka** (Kafka fundamentals, cluster architecture, installation, integration with Spark, producer/consumer model, Kafka pipeline, ingestion patterns).
