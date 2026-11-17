# Unit IV — Phase 5: SQL Filtering, Sorting and Wildcards

> **Continuing from Phase 4.** We can now run SQL on a view. In this phase we learn to **choose rows** (WHERE, DISTINCT, LIMIT), **order them** (ORDER BY) and **search text** with the syllabus topic **SQL wildcards** (`LIKE`, `%`, `_`).
>
> Every query and output below was run in a real `spark-shell` (Spark 3.5.1).

---

## Phase 5 — Roadmap

| # | Topic |
| --- | --- |
| 1 | Setup |
| 2 | WHERE and comparison operators |
| 3 | AND, OR, NOT |
| 4 | IN and NOT IN |
| 5 | BETWEEN |
| 6 | DISTINCT |
| 7 | ORDER BY (ASC / DESC) |
| 8 | Sorting by multiple columns |
| 9 | LIMIT and top-N queries |
| 10 | Wildcards — the `LIKE` operator |
| 11 | `%` wildcard |
| 12 | `_` wildcard |
| 13 | Combining wildcards and conditions |
| 14 | Case sensitivity of LIKE |
| 15 | RLIKE (regular expressions) — bonus |
| 16 | NULL handling |
| 17 | CASE WHEN |
| 18 | DataFrame API equivalents |
| 19 | SQL clause order |
| 20 | Common errors |
| 21 | Practical scenarios |
| 22 | Viva questions |
| 23 | Master revision |

---

### 1. Setup

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

**Reference table used throughout**

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

---

### 2. WHERE and comparison operators

`WHERE` keeps only the rows for which the condition is **true**.

```scala
spark.sql("SELECT name, marks FROM students WHERE marks > 80").show()
```

**Comparison operators**

| Operator | Meaning | Example |
| --- | --- | --- |
| `=` | Equal to | `department = 'CSE'` |
| `<>` or `!=` | Not equal | `age <> 21` |
| `>` | Greater than | `marks > 80` |
| `<` | Less than | `marks < 70` |
| `>=` | Greater or equal | `marks >= 85` |
| `<=` | Less or equal | `age <= 22` |

> **SQL uses a single `=`** for equality, not `==` (and not `===` which is the DataFrame API).

**Example: not equal**

```scala
spark.sql("SELECT name, age FROM students WHERE age <> 21").show()
```

**Output**

```text
+-----+---+
| name|age|
+-----+---+
|  Bob| 22|
|David| 23|
| Emma| 22|
+-----+---+
```

---

### 3. AND, OR, NOT

| Operator | Meaning |
| --- | --- |
| `AND` | **Both** conditions must be true |
| `OR` | **At least one** condition must be true |
| `NOT` | Reverses a condition |

**AND**

```scala
spark.sql("SELECT * FROM students WHERE department = 'CSE' AND marks > 85").show()
```

**Output**

```text
+----------+-------+----------+---+-----+
|student_id|   name|department|age|marks|
+----------+-------+----------+---+-----+
|       103|Charlie|       CSE| 21|   91|
+----------+-------+----------+---+-----+
```

**OR**

```scala
spark.sql("SELECT * FROM students WHERE department = 'IT' OR marks > 90").show()
```

**Output**

```text
+----------+-------+----------+---+-----+
|student_id|   name|department|age|marks|
+----------+-------+----------+---+-----+
|       103|Charlie|       CSE| 21|   91|
|       104|  David|        IT| 23|   65|
+----------+-------+----------+---+-----+
```

**NOT**

```scala
spark.sql("SELECT * FROM students WHERE NOT department = 'CSE'").show()
```

**Output**

```text
+----------+-----+----------+---+-----+
|student_id| name|department|age|marks|
+----------+-----+----------+---+-----+
|       102|  Bob|       ECE| 22|   72|
|       104|David|        IT| 23|   65|
|       105| Emma|       ECE| 22|   88|
+----------+-----+----------+---+-----+
```

> **Precedence:** `AND` is evaluated before `OR`. When mixing both, **use brackets**: `WHERE (a OR b) AND c`.

---

### 4. IN and NOT IN

`IN` checks against a list, and is a cleaner replacement for many `OR` conditions.

```scala
spark.sql("SELECT name, department FROM students WHERE department IN ('CSE', 'IT')").show()
```

**Output**

```text
+-------+----------+
|   name|department|
+-------+----------+
|  Alice|       CSE|
|Charlie|       CSE|
|  David|        IT|
+-------+----------+
```

```scala
spark.sql("SELECT name, department FROM students WHERE department NOT IN ('CSE', 'IT')").show()
```

**Output**

```text
+----+----------+
|name|department|
+----+----------+
| Bob|       ECE|
|Emma|       ECE|
+----+----------+
```

---

### 5. BETWEEN

`BETWEEN a AND b` includes **both ends** (inclusive).

```scala
spark.sql("SELECT name, marks FROM students WHERE marks BETWEEN 70 AND 88").show()
```

**Output**

```text
+-----+-----+
| name|marks|
+-----+-----+
|Alice|   85|
|  Bob|   72|
| Emma|   88|
+-----+-----+
```

`Emma` with exactly 88 is included, which shows that the boundaries are inclusive.

```text
marks BETWEEN 70 AND 88   ≡   marks >= 70 AND marks <= 88
```

---

### 6. DISTINCT

`DISTINCT` removes duplicate rows from the result.

```scala
spark.sql("SELECT DISTINCT department FROM students ORDER BY department").show()
```

**Output**

```text
+----------+
|department|
+----------+
|       CSE|
|       ECE|
|        IT|
+----------+
```

**On multiple columns: the combination must be unique**

```scala
spark.sql("SELECT DISTINCT department, age FROM students ORDER BY department, age").show()
```

**Output**

```text
+----------+---+
|department|age|
+----------+---+
|       CSE| 21|
|       ECE| 22|
|        IT| 23|
+----------+---+
```

**Count of distinct values** (we use this again in Phase 6)

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

---

### 7. ORDER BY (ASC / DESC)

`ORDER BY` sorts the result.

| Keyword | Order |
| --- | --- |
| `ASC` | Ascending (small → large). **This is the default.** |
| `DESC` | Descending (large → small) |

**Ascending (default)**

```scala
spark.sql("SELECT name, marks FROM students ORDER BY marks").show()
```

**Output**

```text
+-------+-----+
|   name|marks|
+-------+-----+
|  David|   65|
|    Bob|   72|
|  Alice|   85|
|   Emma|   88|
|Charlie|   91|
+-------+-----+
```

**Descending**

```scala
spark.sql("SELECT name, marks FROM students ORDER BY marks DESC").show()
```

**Output**

```text
+-------+-----+
|   name|marks|
+-------+-----+
|Charlie|   91|
|   Emma|   88|
|  Alice|   85|
|    Bob|   72|
|  David|   65|
+-------+-----+
```

**Sort by alias or position**

```scala
spark.sql("SELECT name, marks AS score FROM students ORDER BY score DESC LIMIT 1").show()
```

```text
+-------+-----+
|   name|score|
+-------+-----+
|Charlie|   91|
+-------+-----+
```

```scala
spark.sql("SELECT name, marks FROM students ORDER BY 2 DESC").show()
```

`ORDER BY 2` means "sort by the **2nd** column in the SELECT list". It works, but naming the column is clearer.

> **Important:** Without `ORDER BY`, SQL gives **no guaranteed row order**. Never rely on the order of unsorted results.

---

### 8. Sorting by multiple columns

Spark sorts by the first column; ties are broken by the second, and so on.

```scala
spark.sql("""
  SELECT name, department, marks
  FROM students
  ORDER BY department ASC, marks DESC
""").show()
```

**Output**

```text
+-------+----------+-----+
|   name|department|marks|
+-------+----------+-----+
|Charlie|       CSE|   91|
|  Alice|       CSE|   85|
|   Emma|       ECE|   88|
|    Bob|       ECE|   72|
|  David|        IT|   65|
+-------+----------+-----+
```

```text
Step 1: group by department alphabetically  → CSE, ECE, IT
Step 2: inside each department, highest marks first
```

Each column has its **own** direction.

---

### 9. LIMIT and top-N queries

`LIMIT n` returns at most `n` rows. Combined with `ORDER BY` it gives **top-N** queries.

**Top 3 students**

```scala
spark.sql("SELECT name, marks FROM students ORDER BY marks DESC LIMIT 3").show()
```

**Output**

```text
+-------+-----+
|   name|marks|
+-------+-----+
|Charlie|   91|
|   Emma|   88|
|  Alice|   85|
+-------+-----+
```

**Topper only**

```scala
spark.sql("SELECT name, marks FROM students ORDER BY marks DESC LIMIT 1").show()
```

**Bottom 2 students**

```scala
spark.sql("SELECT name, marks FROM students ORDER BY marks ASC LIMIT 2").show()
```

> **Pattern:** top N = `ORDER BY column DESC LIMIT N`. bottom N = `ORDER BY column ASC LIMIT N`.

---

### 10. Wildcards — the `LIKE` operator

Sometimes we do not know the exact value; we know only **part** of it. For example, "all names starting with A".

`LIKE` matches text against a **pattern**. Inside a pattern, two special characters are **wildcards**:

| Wildcard | Meaning |
| --- | --- |
| `%` | **Any number of characters** (zero, one or many) |
| `_` | **Exactly one** character |

**Syntax**

```sql
SELECT columns FROM table WHERE column LIKE 'pattern'
```

---

### 11. The `%` wildcard

**Names starting with A:** `'A%'`

```scala
spark.sql("SELECT name FROM students WHERE name LIKE 'A%'").show()
```

```text
+-----+
| name|
+-----+
|Alice|
+-----+
```

**Names ending with e:** `'%e'`

```scala
spark.sql("SELECT name FROM students WHERE name LIKE '%e'").show()
```

```text
+-------+
|   name|
+-------+
|  Alice|
|Charlie|
+-------+
```

**Names containing "li" anywhere:** `'%li%'`

```scala
spark.sql("SELECT name FROM students WHERE name LIKE '%li%'").show()
```

```text
+-------+
|   name|
+-------+
|  Alice|
|Charlie|
+-------+
```

**Names NOT starting with A:** `NOT LIKE`

```scala
spark.sql("SELECT name FROM students WHERE name NOT LIKE 'A%'").show()
```

```text
+-------+
|   name|
+-------+
|    Bob|
|Charlie|
|  David|
|   Emma|
+-------+
```

**Placement cheat sheet**

| Pattern | Meaning |
| --- | --- |
| `'A%'` | Starts with A |
| `'%e'` | Ends with e |
| `'%li%'` | Contains "li" |
| `'A%e'` | Starts with A **and** ends with e (Alice) |

---

### 12. The `_` wildcard

`_` stands for **exactly one** character.

**Three-letter names that end in "ob":** `'_ob'`

```scala
spark.sql("SELECT name FROM students WHERE name LIKE '_ob'").show()
```

```text
+----+
|name|
+----+
| Bob|
+----+
```

**Names where the second letter is "m":** `'_m%'`

```scala
spark.sql("SELECT name FROM students WHERE name LIKE '_m%'").show()
```

```text
+----+
|name|
+----+
|Emma|
+----+
```

**Five-letter names starting with D:** `'D____'` (D followed by four underscores)

```scala
spark.sql("SELECT name FROM students WHERE name LIKE 'D____'").show()
```

```text
+-----+
| name|
+-----+
|David|
+-----+
```

**`%` vs `_` visualized**

```text
'_ob'      → [any 1 char] o b           → Bob
'D____'    → D [4 chars]                → David
'A%'       → A [any length, even 0]     → Alice
```

| `%` | `_` |
| --- | --- |
| Any number of characters (0 or more) | Exactly one character |
| `'A%'` matches `A`, `Al`, `Alice` | `'A_'` matches only 2-letter words starting with A |

---

### 13. Combining wildcards and conditions

```scala
spark.sql("SELECT name FROM students WHERE name LIKE 'A%' OR name LIKE 'E%'").show()
```

```text
+-----+
| name|
+-----+
|Alice|
| Emma|
+-----+
```

```scala
spark.sql("SELECT name, department FROM students WHERE department LIKE 'C%' AND name LIKE '%e'").show()
```

```text
+-------+----------+
|   name|department|
+-------+----------+
|  Alice|       CSE|
|Charlie|       CSE|
+-------+----------+
```

> One `LIKE` takes **one pattern**. For several patterns, join multiple `LIKE` conditions with `OR`/`AND`.

---

### 14. Case sensitivity of LIKE ⚠️

**In Spark SQL, `LIKE` is case-sensitive.**

```scala
spark.sql("SELECT name FROM students WHERE name LIKE '%a%'").show()
```

```text
+-------+
|   name|
+-------+
|Charlie|
|  David|
|   Emma|
+-------+
```

`Alice` is **missing**. It contains `A` (capital) but no lowercase `a`. The pattern `'%a%'` looks for lowercase only.

**Fix 1 — convert to lowercase first**

```scala
spark.sql("SELECT name FROM students WHERE LOWER(name) LIKE '%a%'").show()
```

```text
+-------+
|   name|
+-------+
|  Alice|
|Charlie|
|  David|
|   Emma|
+-------+
```

**Fix 2 — `ILIKE` (case-insensitive LIKE; Spark 3.3 and later)**

```scala
spark.sql("SELECT name FROM students WHERE name ILIKE '%A%'").show()
```

Same output as Fix 1.

> **Viva trap:** "Is LIKE case-sensitive in Spark SQL?" → **Yes.** Use `LOWER()`/`UPPER()` or `ILIKE`.

---

### 15. RLIKE (regular expressions) — bonus

`RLIKE` matches a **regular expression** instead of a `%`/`_` pattern.

```scala
spark.sql("SELECT name FROM students WHERE name RLIKE '^[A-C]'").show()
```

```text
+-------+
|   name|
+-------+
|  Alice|
|    Bob|
|Charlie|
+-------+
```

`^[A-C]` means "starts with A, B or C". Know that it exists, but `LIKE` with `%` and `_` is what the syllabus expects.

---

### 16. NULL handling

`NULL` means **unknown / missing**. It needs special handling.

Create data with a missing mark:

```scala
val studentsN = Seq(
  (101, "Alice", "CSE", 21, Some(85)),
  (102, "Bob", "ECE", 22, Some(72)),
  (103, "Charlie", "CSE", 21, Some(91)),
  (106, "Farah", "IT", 22, Option.empty[Int])
).toDF("student_id", "name", "department", "age", "marks")

studentsN.createOrReplaceTempView("students_n")
```

`Some(85)` means a value exists; `Option.empty[Int]` means no value, so it becomes NULL.

**Find missing values: `IS NULL`**

```scala
spark.sql("SELECT * FROM students_n WHERE marks IS NULL").show()
```

```text
+----------+-----+----------+---+-----+
|student_id| name|department|age|marks|
+----------+-----+----------+---+-----+
|       106|Farah|        IT| 22| NULL|
+----------+-----+----------+---+-----+
```

**Find existing values: `IS NOT NULL`**

```scala
spark.sql("SELECT * FROM students_n WHERE marks IS NOT NULL").show()
```

```text
+----------+-------+----------+---+-----+
|student_id|   name|department|age|marks|
+----------+-------+----------+---+-----+
|       101|  Alice|       CSE| 21|   85|
|       102|    Bob|       ECE| 22|   72|
|       103|Charlie|       CSE| 21|   91|
+----------+-------+----------+---+-----+
```

**⚠️ `= NULL` does not work**

```scala
spark.sql("SELECT * FROM students_n WHERE marks = NULL").show()
```

```text
+----------+----+----------+---+-----+
|student_id|name|department|age|marks|
+----------+----+----------+---+-----+
+----------+----+----------+---+-----+
```

**Empty!** Comparing anything to NULL gives *unknown*, never true. Always use `IS NULL`.

**Comparisons skip NULL rows**

```scala
spark.sql("SELECT * FROM students_n WHERE marks > 80").show()
```

```text
+----------+-------+----------+---+-----+
|student_id|   name|department|age|marks|
+----------+-------+----------+---+-----+
|       101|  Alice|       CSE| 21|   85|
|       103|Charlie|       CSE| 21|   91|
+----------+-------+----------+---+-----+
```

Farah is missing, because `NULL > 80` is neither true nor false.

**Replace NULL with a default: `COALESCE`**

```scala
spark.sql("SELECT name, COALESCE(marks, 0) AS marks FROM students_n").show()
```

```text
+-------+-----+
|   name|marks|
+-------+-----+
|  Alice|   85|
|    Bob|   72|
|Charlie|   91|
|  Farah|    0|
+-------+-----+
```

**NULLs and sorting**

```scala
spark.sql("SELECT name, marks FROM students_n ORDER BY marks").show()
```

```text
+-------+-----+
|   name|marks|
+-------+-----+
|  Farah| NULL|
|    Bob|   72|
|  Alice|   85|
|Charlie|   91|
+-------+-----+
```

In Spark SQL, ascending order puts NULLs **first**; descending puts them **last**.

```scala
spark.sql("SELECT name, marks FROM students_n ORDER BY marks DESC").show()
```

```text
+-------+-----+
|   name|marks|
+-------+-----+
|Charlie|   91|
|  Alice|   85|
|    Bob|   72|
|  Farah| NULL|
+-------+-----+
```

You can control this explicitly with `NULLS FIRST` / `NULLS LAST`:

```scala
spark.sql("SELECT name, marks FROM students_n ORDER BY marks DESC NULLS LAST").show()
```

---

### 17. CASE WHEN

`CASE WHEN` is SQL's if-else. It creates a new column based on conditions. (The DataFrame equivalent is `when().otherwise()` from Phase 2.)

```scala
spark.sql("""
  SELECT name, marks,
         CASE WHEN marks >= 90 THEN 'A'
              WHEN marks >= 80 THEN 'B'
              ELSE 'C'
         END AS grade
  FROM students
""").show()
```

**Output**

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

**How it is evaluated:** top to bottom; the first condition that is true wins.

---

### 18. DataFrame API equivalents

| SQL | DataFrame API |
| --- | --- |
| `WHERE marks > 80` | `.filter(col("marks") > 80)` |
| `name LIKE 'A%'` | `col("name").like("A%")` or `.startsWith("A")` |
| `name LIKE '%e'` | `col("name").endsWith("e")` |
| `name LIKE '%li%'` | `col("name").contains("li")` |
| `marks BETWEEN 70 AND 88` | `col("marks").between(70, 88)` |
| `department IN ('CSE','IT')` | `col("department").isin("CSE", "IT")` |
| `ORDER BY marks DESC` | `.orderBy(desc("marks"))` or `.orderBy(col("marks").desc)` |
| `ORDER BY a ASC, b DESC` | `.orderBy(col("a").asc, col("b").desc)` |
| `LIMIT 3` | `.limit(3)` |
| `marks IS NULL` | `col("marks").isNull` |

**Examples**

```scala
studentsDF.filter(col("name").like("A%")).show()
studentsDF.filter(col("name").endsWith("e")).show()
studentsDF.filter(col("name").contains("li")).show()
studentsDF.filter(col("marks").between(70, 88)).select("name", "marks").show()
studentsDF.filter(col("department").isin("CSE", "IT")).select("name", "department").show()
studentsDF.select("name", "marks").orderBy(desc("marks")).limit(3).show()
```

**Output of the last line**

```text
+-------+-----+
|   name|marks|
+-------+-----+
|Charlie|   91|
|   Emma|   88|
|  Alice|   85|
+-------+-----+
```

---

### 19. SQL clause order

SQL clauses must be written in this order:

```text
SELECT   columns
FROM     table
WHERE    row conditions
GROUP BY grouping columns        (Phase 6)
HAVING   group conditions        (Phase 6)
ORDER BY sorting
LIMIT    row limit
```

**A complete query using what we learned**

```scala
spark.sql("""
  SELECT DISTINCT department
  FROM students
  WHERE marks > 70
  ORDER BY department DESC
  LIMIT 2
""").show()
```

```text
+----------+
|department|
+----------+
|       ECE|
|       CSE|
+----------+
```

Students with marks over 70 are Alice (CSE), Bob (ECE), Charlie (CSE) and Emma (ECE). Distinct departments are CSE and ECE. Sorted descending gives ECE, CSE.

> **Memory trick:** *"Some Fine Wines Give Happy Of Life"* → **S**ELECT, **F**ROM, **W**HERE, **G**ROUP BY, **H**AVING, **O**RDER BY, **L**IMIT.

---

### 20. Common errors

| Mistake | Wrong | Correct |
| --- | --- | --- |
| Using `==` in SQL | `WHERE marks == 85` | `WHERE marks = 85` |
| Comparing with NULL | `WHERE marks = NULL` | `WHERE marks IS NULL` |
| Double quotes for text | `department = "CSE"` | `department = 'CSE'` |
| Expecting `LIKE` to ignore case | `LIKE '%a%'` finds "Alice" | Use `LOWER(name)` or `ILIKE` |
| `%` instead of `_` | `'D%'` for "5-letter names starting with D" | `'D____'` |
| Wrong clause order | `ORDER BY ... WHERE ...` | `WHERE` first, then `ORDER BY` |
| Sorting on an unselected alias in wrong place | Using alias in `WHERE` | Aliases work in `ORDER BY`, not in `WHERE` |
| Missing brackets with AND/OR | `a OR b AND c` | `(a OR b) AND c` |

---

### 21. Practical scenarios — Phase 5 practice

Use `students` and `students_n` from the setup.

1. **Students with marks above 80**
   ```scala
   spark.sql("SELECT name, marks FROM students WHERE marks > 80").show()
   ```
2. **CSE students scoring above 85**
   ```scala
   spark.sql("SELECT * FROM students WHERE department = 'CSE' AND marks > 85").show()
   ```
3. **Students in CSE or IT**
   ```scala
   spark.sql("SELECT name, department FROM students WHERE department IN ('CSE','IT')").show()
   ```
4. **Marks between 70 and 90**
   ```scala
   spark.sql("SELECT name, marks FROM students WHERE marks BETWEEN 70 AND 90").show()
   ```
5. **Unique departments**
   ```scala
   spark.sql("SELECT DISTINCT department FROM students").show()
   ```
6. **All students sorted by marks, highest first**
   ```scala
   spark.sql("SELECT name, marks FROM students ORDER BY marks DESC").show()
   ```
7. **Top 3 students**
   ```scala
   spark.sql("SELECT name, marks FROM students ORDER BY marks DESC LIMIT 3").show()
   ```
8. **Names starting with E**
   ```scala
   spark.sql("SELECT name FROM students WHERE name LIKE 'E%'").show()
   ```
9. **Names with exactly three letters**
   ```scala
   spark.sql("SELECT name FROM students WHERE name LIKE '___'").show()
   ```
   Result: `Bob`.
10. **Names containing "a" in any case**
    ```scala
    spark.sql("SELECT name FROM students WHERE LOWER(name) LIKE '%a%'").show()
    ```
11. **Students without marks**
    ```scala
    spark.sql("SELECT * FROM students_n WHERE marks IS NULL").show()
    ```
12. **Grade column**
    ```scala
    spark.sql("SELECT name, marks, CASE WHEN marks >= 90 THEN 'A' WHEN marks >= 80 THEN 'B' ELSE 'C' END AS grade FROM students").show()
    ```

---

### 22. Phase 5 — viva questions and answers

#### Q1. What does the WHERE clause do?

It filters rows, keeping only those for which the condition is true.

#### Q2. What is the difference between `AND` and `OR`?

`AND` needs both conditions true; `OR` needs at least one true.

#### Q3. What does `IN` do?

It checks whether a value matches any value in a list.

#### Q4. Is `BETWEEN` inclusive?

Yes, both boundary values are included.

#### Q5. What does `DISTINCT` do?

It removes duplicate rows from the result.

#### Q6. What is the default order of `ORDER BY`?

Ascending (`ASC`).

#### Q7. How do you sort in descending order?

Add `DESC` after the column: `ORDER BY marks DESC`.

#### Q8. How do you find the top 3 students?

`ORDER BY marks DESC LIMIT 3`.

#### Q9. What is a wildcard?

A special character in a `LIKE` pattern that stands for unknown characters.

#### Q10. What are the two wildcards used with LIKE?

`%` (any number of characters) and `_` (exactly one character).

#### Q11. Write a query for names that start with A.

`SELECT name FROM students WHERE name LIKE 'A%'`

#### Q12. What does `'_ob'` match?

Any three-letter text whose last two letters are "ob", such as "Bob".

#### Q13. Is LIKE case-sensitive in Spark SQL?

Yes. Use `LOWER()`, `UPPER()` or `ILIKE` for case-insensitive matching.

#### Q14. How do you check for NULL values?

Using `IS NULL` and `IS NOT NULL`. `= NULL` never works.

#### Q15. Where do NULLs appear in ascending and descending ORDER BY?

Ascending puts NULLs first; descending puts them last (by default in Spark SQL).

#### Q16. What does `COALESCE(marks, 0)` do?

It returns the first non-null value, so NULL marks become 0.

#### Q17. What is the correct order of SQL clauses?

SELECT, FROM, WHERE, GROUP BY, HAVING, ORDER BY, LIMIT.

#### Q18. What is the DataFrame equivalent of `LIKE 'A%'`?

`col("name").like("A%")` or `col("name").startsWith("A")`.

#### Q19. What does `CASE WHEN` do?

It evaluates conditions in order and returns the value of the first true branch, like if-else.

#### Q20. Why can't we use `=` to compare with NULL?

Because any comparison with NULL results in unknown, not true.

---

### 23. Phase 5 — final master code revision

```scala
import spark.implicits._
import org.apache.spark.sql.functions._

val studentsDF = Seq(
  (101, "Alice", "CSE", 21, 85), (102, "Bob", "ECE", 22, 72),
  (103, "Charlie", "CSE", 21, 91), (104, "David", "IT", 23, 65),
  (105, "Emma", "ECE", 22, 88)
).toDF("student_id", "name", "department", "age", "marks")
studentsDF.createOrReplaceTempView("students")

// WHERE
spark.sql("SELECT * FROM students WHERE marks > 80").show()
spark.sql("SELECT * FROM students WHERE department = 'CSE' AND marks > 85").show()
spark.sql("SELECT * FROM students WHERE department IN ('CSE','IT')").show()
spark.sql("SELECT * FROM students WHERE marks BETWEEN 70 AND 88").show()

// DISTINCT
spark.sql("SELECT DISTINCT department FROM students").show()

// ORDER BY and LIMIT
spark.sql("SELECT name, marks FROM students ORDER BY marks DESC").show()
spark.sql("SELECT name, department, marks FROM students ORDER BY department ASC, marks DESC").show()
spark.sql("SELECT name, marks FROM students ORDER BY marks DESC LIMIT 3").show()

// Wildcards
spark.sql("SELECT name FROM students WHERE name LIKE 'A%'").show()      // starts with A
spark.sql("SELECT name FROM students WHERE name LIKE '%e'").show()      // ends with e
spark.sql("SELECT name FROM students WHERE name LIKE '%li%'").show()    // contains li
spark.sql("SELECT name FROM students WHERE name LIKE '_ob'").show()     // Bob
spark.sql("SELECT name FROM students WHERE name LIKE 'D____'").show()   // David
spark.sql("SELECT name FROM students WHERE LOWER(name) LIKE '%a%'").show()

// NULLs and CASE
spark.sql("SELECT * FROM students_n WHERE marks IS NULL").show()
spark.sql("SELECT name, CASE WHEN marks >= 90 THEN 'A' WHEN marks >= 80 THEN 'B' ELSE 'C' END AS grade FROM students").show()
```

**Wildcard memory card**

| Pattern | Meaning | Matches |
| --- | --- | --- |
| `'A%'` | Starts with A | Alice |
| `'%e'` | Ends with e | Alice, Charlie |
| `'%li%'` | Contains li | Alice, Charlie |
| `'_ob'` | 3 letters, ends "ob" | Bob |
| `'D____'` | D + 4 letters | David |
| `'_m%'` | 2nd letter m | Emma |

**Phase 5 completed**

---

> **Next → Phase 6: GROUP BY and Aggregate Functions** (COUNT, SUM, AVG, MIN, MAX, HAVING). File: `6_Unit4_GroupBy_Aggregate_Functions.md`
