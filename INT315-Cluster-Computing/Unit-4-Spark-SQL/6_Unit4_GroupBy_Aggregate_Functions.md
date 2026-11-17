# Unit IV — Phase 6: GROUP BY and Aggregate Functions

> **Continuing from Phase 5.** So far, every query returned individual rows. Now we **summarise** data: totals, averages, counts per group. This covers the syllabus topics **groupby** and **aggregate functions**.
>
> All queries and outputs were run in a real `spark-shell` (Spark 3.5.1).

---

## Phase 6 — Roadmap

| # | Topic |
| --- | --- |
| 1 | What is aggregation? |
| 2 | Setup |
| 3 | The five aggregate functions |
| 4 | Aggregates on the whole table |
| 5 | GROUP BY — the core idea |
| 6 | GROUP BY with multiple aggregates |
| 7 | WHERE with GROUP BY |
| 8 | HAVING — filtering groups |
| 9 | WHERE vs HAVING |
| 10 | GROUP BY multiple columns |
| 11 | ORDER BY with aggregates / top group |
| 12 | `COUNT(*)` vs `COUNT(column)` vs `COUNT(DISTINCT)` |
| 13 | ROUND, and NULLs in aggregates |
| 14 | A second dataset: sales |
| 15 | Subquery with an aggregate |
| 16 | DataFrame API: `groupBy` and `agg` |
| 17 | Common errors |
| 18 | Practical scenarios |
| 19 | Viva questions |
| 20 | Master revision |

---

### 1. What is aggregation?

**Aggregation** means combining many rows into **one summary value**.

```text
marks:  85, 72, 91, 65, 88
                |
                | SUM
                ↓
              401          (one value from five rows)
```

Examples of aggregation questions:

- How many students are there?
- What is the **average** marks?
- What is the **highest** score in each department?

**GROUP BY** adds a twist: do the aggregation **separately for each group**.

```text
All students                     GROUP BY department
 ┌───────────────┐               ┌───────┐  ┌───────┐  ┌──────┐
 │ 5 rows mixed  │   ───────▶    │  CSE  │  │  ECE  │  │  IT  │
 └───────────────┘               │ 2 rows│  │ 2 rows│  │ 1 row│
                                 └───────┘  └───────┘  └──────┘
                                   ↓          ↓          ↓
                                one result per group
```

---

### 2. Setup

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

studentsDF.createOrReplaceTempView("students")
```

---

### 3. The five aggregate functions

| Function | What it computes | Result type |
| --- | --- | --- |
| `COUNT()` | Number of rows | Long |
| `SUM()` | Total of values | Long (for ints) / Double |
| `AVG()` | Average (mean) | Double |
| `MIN()` | Smallest value | Same as column |
| `MAX()` | Largest value | Same as column |

> **Key rule:** an aggregate function takes **many rows** and returns **one value**.

---

### 4. Aggregates on the whole table

Without `GROUP BY`, the whole table is treated as one group.

**COUNT**

```scala
spark.sql("SELECT COUNT(*) AS total_students FROM students").show()
```

```text
+--------------+
|total_students|
+--------------+
|             5|
+--------------+
```

**SUM**

```scala
spark.sql("SELECT SUM(marks) AS total_marks FROM students").show()
```

```text
+-----------+
|total_marks|
+-----------+
|        401|
+-----------+
```

**AVG**

```scala
spark.sql("SELECT AVG(marks) AS avg_marks FROM students").show()
```

```text
+---------+
|avg_marks|
+---------+
|     80.2|
+---------+
```

**MIN and MAX**

```scala
spark.sql("SELECT MIN(marks) AS lowest, MAX(marks) AS highest FROM students").show()
```

```text
+------+-------+
|lowest|highest|
+------+-------+
|    65|     91|
+------+-------+
```

**All five together**

```scala
spark.sql("""
  SELECT COUNT(*) AS n, SUM(marks) AS total, AVG(marks) AS average,
         MIN(marks) AS lowest, MAX(marks) AS highest
  FROM students
""").show()
```

```text
+---+-----+-------+------+-------+
|  n|total|average|lowest|highest|
+---+-----+-------+------+-------+
|  5|  401|   80.2|    65|     91|
+---+-----+-------+------+-------+
```

---

### 5. GROUP BY — the core idea

**Question:** How many students are there in each department?

```scala
spark.sql("""
  SELECT department, COUNT(*) AS student_count
  FROM students
  GROUP BY department
  ORDER BY department
""").show()
```

**Output**

```text
+----------+-------------+
|department|student_count|
+----------+-------------+
|       CSE|            2|
|       ECE|            2|
|        IT|            1|
+----------+-------------+
```

**How Spark processes it**

```text
Step 1: Split rows into groups by department
  CSE → Alice(85), Charlie(91)
  ECE → Bob(72), Emma(88)
  IT  → David(65)

Step 2: Apply COUNT(*) to each group
  CSE → 2
  ECE → 2
  IT  → 1

Step 3: One output row per group
```

**Average marks per department**

```scala
spark.sql("""
  SELECT department, AVG(marks) AS avg_marks
  FROM students
  GROUP BY department
  ORDER BY department
""").show()
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

Check: CSE = (85 + 91) / 2 = 88.0 ✔

> **Golden rule:** Every column in `SELECT` must either be **in the GROUP BY** or be **inside an aggregate function**.

---

### 6. GROUP BY with multiple aggregates

```scala
spark.sql("""
  SELECT department,
         COUNT(*)   AS n,
         SUM(marks) AS total,
         AVG(marks) AS avg_marks,
         MIN(marks) AS lowest,
         MAX(marks) AS highest
  FROM students
  GROUP BY department
  ORDER BY department
""").show()
```

**Output**

```text
+----------+---+-----+---------+------+-------+
|department|  n|total|avg_marks|lowest|highest|
+----------+---+-----+---------+------+-------+
|       CSE|  2|  176|     88.0|    85|     91|
|       ECE|  2|  160|     80.0|    72|     88|
|        IT|  1|   65|     65.0|    65|     65|
+----------+---+-----+---------+------+-------+
```

This is a typical "department report" query.

---

### 7. WHERE with GROUP BY

`WHERE` filters **rows before grouping**.

**Question:** Count students per department, but only those with marks above 70.

```scala
spark.sql("""
  SELECT department, COUNT(*) AS n
  FROM students
  WHERE marks > 70
  GROUP BY department
  ORDER BY department
""").show()
```

```text
+----------+---+
|department|  n|
+----------+---+
|       CSE|  2|
|       ECE|  2|
+----------+---+
```

IT disappears because David (65) is removed **before** grouping, so the IT group is empty.

---

### 8. HAVING — filtering groups

`HAVING` filters **groups after aggregation**.

**Question:** Show only departments with average marks above 70.

```scala
spark.sql("""
  SELECT department, AVG(marks) AS avg_marks
  FROM students
  GROUP BY department
  HAVING AVG(marks) > 70
  ORDER BY department
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

**Question:** Departments with at least 2 students.

```scala
spark.sql("""
  SELECT department, COUNT(*) AS n
  FROM students
  GROUP BY department
  HAVING COUNT(*) >= 2
  ORDER BY department
""").show()
```

```text
+----------+---+
|department|  n|
+----------+---+
|       CSE|  2|
|       ECE|  2|
+----------+---+
```

---

### 9. WHERE vs HAVING 🔥

This is a **favourite viva question**.

| `WHERE` | `HAVING` |
| --- | --- |
| Filters **individual rows** | Filters **groups** |
| Runs **before** GROUP BY | Runs **after** GROUP BY |
| Cannot use aggregate functions | **Can** use aggregate functions |
| `WHERE marks > 70` | `HAVING AVG(marks) > 70` |

**Execution order**

```text
FROM  →  WHERE  →  GROUP BY  →  HAVING  →  SELECT  →  ORDER BY  →  LIMIT
        (rows)                 (groups)
```

**Using both in one query**

```scala
spark.sql("""
  SELECT department, AVG(marks) AS avg_marks
  FROM students
  WHERE age >= 22
  GROUP BY department
  HAVING AVG(marks) > 70
""").show()
```

**Output**

```text
+----------+---------+
|department|avg_marks|
+----------+---------+
|       ECE|     80.0|
+----------+---------+
```

```text
1. WHERE   → keep only students aged 22 or more
2. GROUP BY → group the remaining rows by department
3. HAVING  → keep only departments whose average is above 70
```

> **Memory line:** *WHERE filters rows; HAVING filters groups.*

---

### 10. GROUP BY multiple columns

Groups are formed from **unique combinations**.

```scala
spark.sql("""
  SELECT department, age, COUNT(*) AS n
  FROM students
  GROUP BY department, age
  ORDER BY department, age
""").show()
```

```text
+----------+---+---+
|department|age|  n|
+----------+---+---+
|       CSE| 21|  2|
|       ECE| 22|  2|
|        IT| 23|  1|
+----------+---+---+
```

Each distinct (department, age) pair is its own group.

---

### 11. ORDER BY with aggregates / the top group

Sort by an aggregate using its alias:

```scala
spark.sql("""
  SELECT department, ROUND(AVG(marks), 1) AS avg_marks
  FROM students
  GROUP BY department
  ORDER BY avg_marks DESC
""").show()
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

**Which department has the highest top score?**

```scala
spark.sql("""
  SELECT department, MAX(marks) AS top
  FROM students
  GROUP BY department
  ORDER BY top DESC
  LIMIT 1
""").show()
```

```text
+----------+---+
|department|top|
+----------+---+
|       CSE| 91|
+----------+---+
```

**Pattern:** `GROUP BY` → aggregate → `ORDER BY alias DESC` → `LIMIT 1` gives "the group with the max/min".

---

### 12. `COUNT(*)` vs `COUNT(column)` vs `COUNT(DISTINCT column)`

| Form | Counts |
| --- | --- |
| `COUNT(*)` | **All rows**, including those with NULLs |
| `COUNT(column)` | Rows where that column is **not NULL** |
| `COUNT(DISTINCT column)` | **Unique** non-null values |

**Distinct count**

```scala
spark.sql("SELECT COUNT(DISTINCT department) AS dept_count FROM students").show()
```

```text
+----------+
|dept_count|
+----------+
|         3|
+----------+
```

**NULL difference** (using the `students_n` data from Phase 5: one student has NULL marks)

```scala
val studentsN = Seq(
  (101, "Alice", "CSE", 21, Some(85)),
  (102, "Bob", "ECE", 22, Some(72)),
  (103, "Charlie", "CSE", 21, Some(91)),
  (106, "Farah", "IT", 22, Option.empty[Int])
).toDF("student_id", "name", "department", "age", "marks")
studentsN.createOrReplaceTempView("students_n")

spark.sql("""
  SELECT COUNT(*) AS all_rows,
         COUNT(marks) AS marks_rows,
         AVG(marks) AS avg_marks
  FROM students_n
""").show()
```

```text
+--------+----------+-----------------+
|all_rows|marks_rows|        avg_marks|
+--------+----------+-----------------+
|       4|         3|82.66666666666667|
+--------+----------+-----------------+
```

**Key lesson**

- `COUNT(*)` = 4 (Farah's row is counted).
- `COUNT(marks)` = 3 (Farah's NULL is skipped).
- `AVG(marks)` = (85 + 72 + 91) / **3** = 82.67. Farah is **ignored**, not treated as zero.

> **All aggregate functions except `COUNT(*)` ignore NULLs.**

---

### 13. ROUND and formatting results

`AVG` often gives long decimals. Use `ROUND(value, digits)`.

```scala
spark.sql("SELECT ROUND(AVG(marks), 2) AS avg_marks FROM students").show()
```

```text
+---------+
|avg_marks|
+---------+
|     80.2|
+---------+
```

---

### 14. A second dataset: sales

Grouping is more interesting with transactional data. Create a sales table:

```scala
val salesDF = Seq(
  (1, "Laptop", "Delhi",  2, 55000),
  (2, "Phone",  "Mumbai", 5, 20000),
  (3, "Laptop", "Mumbai", 1, 55000),
  (4, "Tablet", "Delhi",  3, 15000),
  (5, "Phone",  "Delhi",  2, 20000),
  (6, "Tablet", "Mumbai", 4, 15000)
).toDF("order_id", "product", "city", "quantity", "price")

salesDF.createOrReplaceTempView("sales")
```

**Units and revenue per product**

```scala
spark.sql("""
  SELECT product,
         SUM(quantity)         AS units,
         SUM(quantity * price) AS revenue
  FROM sales
  GROUP BY product
  ORDER BY revenue DESC
""").show()
```

```text
+-------+-----+-------+
|product|units|revenue|
+-------+-----+-------+
| Laptop|    3| 165000|
|  Phone|    7| 140000|
| Tablet|    7| 105000|
+-------+-----+-------+
```

We can put an **expression** (`quantity * price`) inside the aggregate. Laptop revenue: 2 × 55000 + 1 × 55000 = 165000 ✔

**Revenue per city**

```scala
spark.sql("""
  SELECT city, SUM(quantity * price) AS revenue
  FROM sales
  GROUP BY city
  ORDER BY city
""").show()
```

```text
+------+-------+
|  city|revenue|
+------+-------+
| Delhi| 195000|
|Mumbai| 215000|
+------+-------+
```

**Units by city and product**

```scala
spark.sql("""
  SELECT city, product, SUM(quantity) AS units
  FROM sales
  GROUP BY city, product
  ORDER BY city, product
""").show()
```

```text
+------+-------+-----+
|  city|product|units|
+------+-------+-----+
| Delhi| Laptop|    2|
| Delhi|  Phone|    2|
| Delhi| Tablet|    3|
|Mumbai| Laptop|    1|
|Mumbai|  Phone|    5|
|Mumbai| Tablet|    4|
+------+-------+-----+
```

**Products earning more than ₹1,20,000 (HAVING)**

```scala
spark.sql("""
  SELECT product, SUM(quantity * price) AS revenue
  FROM sales
  GROUP BY product
  HAVING SUM(quantity * price) > 120000
  ORDER BY revenue DESC
""").show()
```

```text
+-------+-------+
|product|revenue|
+-------+-------+
| Laptop| 165000|
|  Phone| 140000|
+-------+-------+
```

**Grand total revenue**

```scala
spark.sql("SELECT SUM(quantity * price) AS total_revenue FROM sales").show()
```

```text
+-------------+
|total_revenue|
+-------------+
|       410000|
+-------------+
```

---

### 15. Subquery with an aggregate

A **subquery** is a query inside another query. It lets us compare each row with an aggregate value.

**Question:** Students scoring above the class average.

```scala
spark.sql("""
  SELECT name, marks
  FROM students
  WHERE marks > (SELECT AVG(marks) FROM students)
  ORDER BY marks DESC
""").show()
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

```text
Inner query:  SELECT AVG(marks) FROM students   →  80.2
Outer query:  WHERE marks > 80.2
```

> We cannot write `WHERE marks > AVG(marks)`, because aggregates are not allowed directly in `WHERE`. A subquery solves this.

---

### 16. DataFrame API: `groupBy` and `agg`

| SQL | DataFrame API |
| --- | --- |
| `GROUP BY department` + `COUNT(*)` | `groupBy("department").count()` |
| `AVG(marks)` per group | `groupBy("department").avg("marks")` |
| Several aggregates | `groupBy("department").agg(count("*"), avg("marks"), max("marks"))` |
| Whole-table aggregate | `df.agg(max("marks"), min("marks"))` |
| `HAVING` | `.filter(...)` **after** the aggregation |

**Count per department**

```scala
studentsDF.groupBy("department").count().orderBy("department").show()
```

```text
+----------+-----+
|department|count|
+----------+-----+
|       CSE|    2|
|       ECE|    2|
|        IT|    1|
+----------+-----+
```

**Average per department**

```scala
studentsDF.groupBy("department").avg("marks").orderBy("department").show()
```

```text
+----------+----------+
|department|avg(marks)|
+----------+----------+
|       CSE|      88.0|
|       ECE|      80.0|
|        IT|      65.0|
+----------+----------+
```

**Several aggregates with aliases**

```scala
studentsDF.groupBy("department")
  .agg(
    count("*").alias("n"),
    avg("marks").alias("avg_marks"),
    max("marks").alias("highest")
  )
  .orderBy("department")
  .show()
```

```text
+----------+---+---------+-------+
|department|  n|avg_marks|highest|
+----------+---+---------+-------+
|       CSE|  2|     88.0|     91|
|       ECE|  2|     80.0|     88|
|        IT|  1|     65.0|     65|
+----------+---+---------+-------+
```

**Whole-table aggregates**

```scala
studentsDF.agg(max("marks"), min("marks")).show()
```

```text
+----------+----------+
|max(marks)|min(marks)|
+----------+----------+
|        91|        65|
+----------+----------+
```

**HAVING equivalent: filter after aggregation**

```scala
studentsDF.groupBy("department")
  .agg(round(avg("marks"), 1).alias("avg_marks"))
  .filter(col("avg_marks") > 70)
  .orderBy("department")
  .show()
```

```text
+----------+---------+
|department|avg_marks|
+----------+---------+
|       CSE|     88.0|
|       ECE|     80.0|
+----------+---------+
```

**Quick statistics: `describe()`**

```scala
studentsDF.describe("marks").show()
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

**Collecting values per group: `collect_list`**

```scala
spark.sql("""
  SELECT department, collect_list(name) AS names
  FROM students
  GROUP BY department
  ORDER BY department
""").show(false)
```

```text
+----------+----------------+
|department|names           |
+----------+----------------+
|CSE       |[Alice, Charlie]|
|ECE       |[Bob, Emma]     |
|IT        |[David]         |
+----------+----------------+
```

---

### 17. Common errors

#### Error 1 — Selecting a column that is neither grouped nor aggregated

```scala
spark.sql("SELECT department, name FROM students GROUP BY department").show()
```

```text
[MISSING_AGGREGATION] The non-aggregating expression "name" is based on columns which are not
participating in the GROUP BY clause.
```

**Why?** Each department has several names, and Spark cannot choose one.

**Fix:** Add `name` to the GROUP BY, aggregate it (`collect_list(name)`, `MAX(name)`), or reconsider the question.

#### Error 2 — Aggregate in WHERE

```sql
WHERE AVG(marks) > 70        -- ❌
```

**Fix:** use `HAVING AVG(marks) > 70`, or a subquery.

#### Error 3 — Mixing up `COUNT(*)` and `COUNT(column)` when NULLs exist

Remember Section 12.

#### Error 4 — Expecting `AVG` to treat NULL as zero

It ignores NULLs. Use `AVG(COALESCE(marks, 0))` if zero is intended.

#### Error 5 — Forgetting `ORDER BY` after GROUP BY

Group output order is **not guaranteed**. Add `ORDER BY` whenever order matters.

#### Error 6 — Using the alias inside HAVING without repeating the function

Safe form: `HAVING AVG(marks) > 70` (repeat the expression).

---

### 18. Practical scenarios — Phase 6 practice

1. **Total number of students**
   ```scala
   spark.sql("SELECT COUNT(*) FROM students").show()
   ```
2. **Highest and lowest marks**
   ```scala
   spark.sql("SELECT MAX(marks), MIN(marks) FROM students").show()
   ```
3. **Class average**
   ```scala
   spark.sql("SELECT AVG(marks) AS avg_marks FROM students").show()
   ```
4. **Students per department**
   ```scala
   spark.sql("SELECT department, COUNT(*) AS n FROM students GROUP BY department").show()
   ```
5. **Average marks per department**
   ```scala
   spark.sql("SELECT department, AVG(marks) AS avg_marks FROM students GROUP BY department").show()
   ```
6. **Departments with an average above 70**
   ```scala
   spark.sql("SELECT department, AVG(marks) AS avg_marks FROM students GROUP BY department HAVING AVG(marks) > 70").show()
   ```
7. **Department with the highest topper**
   ```scala
   spark.sql("SELECT department, MAX(marks) AS top FROM students GROUP BY department ORDER BY top DESC LIMIT 1").show()
   ```
8. **Number of distinct departments**
   ```scala
   spark.sql("SELECT COUNT(DISTINCT department) FROM students").show()
   ```
9. **Students above the class average** (subquery, see Section 15)
10. **Revenue per product** (sales table, see Section 14)
11. **Revenue per city, highest first**
    ```scala
    spark.sql("SELECT city, SUM(quantity*price) AS revenue FROM sales GROUP BY city ORDER BY revenue DESC").show()
    ```
12. **Department report using the DataFrame API**
    ```scala
    studentsDF.groupBy("department").agg(count("*"), avg("marks"), max("marks")).show()
    ```

---

### 19. Phase 6 — viva questions and answers

#### Q1. What is an aggregate function?

A function that takes many rows and returns a single summary value.

#### Q2. Name the common aggregate functions.

`COUNT`, `SUM`, `AVG`, `MIN`, `MAX`.

#### Q3. What does GROUP BY do?

It divides rows into groups of equal values in the specified columns, so aggregates are computed per group.

#### Q4. Which columns can appear in SELECT with GROUP BY?

Only the grouped columns and aggregate expressions.

#### Q5. What is HAVING?

A clause that filters groups after aggregation.

#### Q6. Difference between WHERE and HAVING?

WHERE filters rows before grouping and cannot use aggregates; HAVING filters groups after grouping and can use aggregates.

#### Q7. What is the logical order of execution of a SQL query?

FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY → LIMIT.

#### Q8. Difference between `COUNT(*)` and `COUNT(column)`?

`COUNT(*)` counts all rows; `COUNT(column)` counts only rows where the column is not NULL.

#### Q9. What does `COUNT(DISTINCT column)` do?

It counts unique non-null values.

#### Q10. How do aggregate functions treat NULL?

They ignore NULL values (except `COUNT(*)`).

#### Q11. What is the return type of `AVG`?

Double.

#### Q12. How do you find the top department by average marks?

`GROUP BY department ORDER BY AVG(marks) DESC LIMIT 1`.

#### Q13. Can we use an aggregate function in WHERE?

No. Use HAVING or a subquery.

#### Q14. What is the DataFrame equivalent of GROUP BY with COUNT?

`df.groupBy("col").count()`.

#### Q15. How do we apply several aggregates with `groupBy`?

Using `agg()`: `df.groupBy("col").agg(count("*"), avg("x"), max("x"))`.

#### Q16. What does `describe()` do?

It shows count, mean, standard deviation, min and max for the given columns.

#### Q17. Why do we get `MISSING_AGGREGATION`?

Because SELECT contains a column that is neither in GROUP BY nor inside an aggregate function.

#### Q18. How do we write HAVING in the DataFrame API?

Apply `filter()` after the `groupBy().agg()`.

#### Q19. What is a subquery?

A query nested inside another query, often used to compare rows with an aggregate value.

#### Q20. Can we GROUP BY more than one column?

Yes. Each unique combination of the grouped columns forms a group.

---

### 20. Phase 6 — final master code revision

```scala
import spark.implicits._
import org.apache.spark.sql.functions._

val studentsDF = Seq(
  (101, "Alice", "CSE", 21, 85), (102, "Bob", "ECE", 22, 72),
  (103, "Charlie", "CSE", 21, 91), (104, "David", "IT", 23, 65),
  (105, "Emma", "ECE", 22, 88)
).toDF("student_id", "name", "department", "age", "marks")
studentsDF.createOrReplaceTempView("students")

// Whole-table aggregates
spark.sql("SELECT COUNT(*) AS n, SUM(marks) AS total, AVG(marks) AS average, MIN(marks) AS lowest, MAX(marks) AS highest FROM students").show()

// GROUP BY
spark.sql("SELECT department, COUNT(*) AS n FROM students GROUP BY department ORDER BY department").show()
spark.sql("SELECT department, AVG(marks) AS avg_marks FROM students GROUP BY department ORDER BY department").show()

// WHERE + GROUP BY
spark.sql("SELECT department, COUNT(*) AS n FROM students WHERE marks > 70 GROUP BY department").show()

// HAVING
spark.sql("SELECT department, AVG(marks) AS avg_marks FROM students GROUP BY department HAVING AVG(marks) > 70").show()

// Multiple group columns
spark.sql("SELECT department, age, COUNT(*) AS n FROM students GROUP BY department, age").show()

// Top group
spark.sql("SELECT department, MAX(marks) AS top FROM students GROUP BY department ORDER BY top DESC LIMIT 1").show()

// Subquery
spark.sql("SELECT name, marks FROM students WHERE marks > (SELECT AVG(marks) FROM students)").show()

// DataFrame API
studentsDF.groupBy("department").count().show()
studentsDF.groupBy("department").agg(count("*").alias("n"), avg("marks").alias("avg_marks")).show()
studentsDF.agg(max("marks"), min("marks")).show()
```

**Aggregate quick-recall card**

| Need | SQL | DataFrame |
| --- | --- | --- |
| Row count | `COUNT(*)` | `count()` |
| Total | `SUM(x)` | `sum("x")` |
| Average | `AVG(x)` | `avg("x")` |
| Smallest / largest | `MIN(x)` / `MAX(x)` | `min("x")` / `max("x")` |
| Per group | `GROUP BY g` | `groupBy("g")` |
| Filter groups | `HAVING ...` | `.filter(...)` after `agg` |

**Phase 6 completed**

---

> **Next → Phase 7: Joins in Spark SQL** (INNER, LEFT, RIGHT, FULL OUTER, CROSS, SEMI, ANTI). File: `7_Unit4_Joins.md`
