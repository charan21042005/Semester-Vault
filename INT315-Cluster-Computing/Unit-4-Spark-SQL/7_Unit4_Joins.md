# Unit IV — Phase 7: Joins in Spark SQL

> **Continuing from Phase 6.** Real data lives in **several tables**. A **join** combines rows from two (or more) tables using a related column. This covers the syllabus topic **joins**.
>
> All queries and outputs below were run in a real `spark-shell` (Spark 3.5.1). Where the row order is not guaranteed, we used `ORDER BY` so your output matches.

---

## Phase 7 — Roadmap

| # | Topic |
| --- | --- |
| 1 | Why do we need joins? |
| 2 | Setup: two tables with deliberate mismatches |
| 3 | The join idea and the join condition |
| 4 | INNER JOIN |
| 5 | LEFT (OUTER) JOIN |
| 6 | RIGHT (OUTER) JOIN |
| 7 | FULL (OUTER) JOIN |
| 8 | LEFT SEMI JOIN |
| 9 | LEFT ANTI JOIN |
| 10 | CROSS JOIN |
| 11 | Join types — master table |
| 12 | Table aliases and the ambiguous column error |
| 13 | Join + WHERE, GROUP BY, ORDER BY |
| 14 | LEFT JOIN + aggregation (the COUNT trap) |
| 15 | Self join |
| 16 | `USING` and the broadcast hint |
| 17 | Joining three tables |
| 18 | DataFrame API joins |
| 19 | Common errors |
| 20 | Practical scenarios |
| 21 | Viva questions |
| 22 | Master revision |

---

### 1. Why do we need joins?

Imagine two tables:

```text
STUDENTS                         DEPARTMENTS
name     department              department   dept_head
Alice    CSE                     CSE          Dr. Sharma
Bob      ECE                     ECE          Dr. Verma
```

The student table knows the **department code**, but the department head's name is stored elsewhere. To get "Alice's department head" we must **combine** the two tables on the common column `department`.

```text
students.department  ═══  departments.department
```

That common column is the **join key**. The expression that matches rows is the **join condition**.

> **Definition:** A join combines rows from two tables based on a related column between them.

---

### 2. Setup: two tables with deliberate mismatches

To understand outer joins we need rows that **do not match**. So we create data like this:

```scala
import spark.implicits._
import org.apache.spark.sql.functions._

val studentsDF = Seq(
  (101, "Alice",   "CSE",  21, 85),
  (102, "Bob",     "ECE",  22, 72),
  (103, "Charlie", "CSE",  21, 91),
  (104, "David",   "IT",   23, 65),
  (105, "Emma",    "ECE",  22, 88),
  (106, "Frank",   "MECH", 22, 59)
).toDF("student_id", "name", "department", "age", "marks")

val deptDF = Seq(
  ("CSE", "Dr. Sharma", "Block A"),
  ("ECE", "Dr. Verma",  "Block B"),
  ("IT",  "Dr. Singh",  "Block C"),
  ("EEE", "Dr. Kaur",   "Block D")
).toDF("department", "dept_head", "building")

studentsDF.createOrReplaceTempView("students")
deptDF.createOrReplaceTempView("departments")
```

**The deliberate mismatches**

```text
students has  MECH  (Frank)      → NOT in departments
departments has  EEE             → NO student in it
```

```text
STUDENTS (6 rows)                 DEPARTMENTS (4 rows)
101 Alice   CSE                   CSE  Dr. Sharma  Block A
102 Bob     ECE                   ECE  Dr. Verma   Block B
103 Charlie CSE                   IT   Dr. Singh   Block C
104 David   IT                    EEE  Dr. Kaur    Block D
105 Emma    ECE
106 Frank   MECH   ← no match
                                  EEE ← no match
```

---

### 3. The join idea and the join condition

**General syntax**

```sql
SELECT columns
FROM   left_table  alias1
JOIN   right_table alias2
ON     alias1.key = alias2.key
```

- `FROM ... a` gives the **left** table.
- `JOIN ... b` gives the **right** table.
- `ON` states **how rows match**.

The join type decides what happens to rows that **find no match**.

---

### 4. INNER JOIN

**Keeps only rows that match in both tables.**

```scala
spark.sql("""
  SELECT s.name, s.department, d.dept_head
  FROM students s
  INNER JOIN departments d
    ON s.department = d.department
  ORDER BY s.student_id
""").show()
```

**Output**

```text
+-------+----------+----------+
|   name|department| dept_head|
+-------+----------+----------+
|  Alice|       CSE|Dr. Sharma|
|    Bob|       ECE| Dr. Verma|
|Charlie|       CSE|Dr. Sharma|
|  David|        IT| Dr. Singh|
|   Emma|       ECE| Dr. Verma|
+-------+----------+----------+
```

- **Frank (MECH)** is gone: there is no MECH department row.
- **EEE** is gone: there is no EEE student.

`JOIN` alone means `INNER JOIN`:

```scala
spark.sql("""
  SELECT s.name, s.department, d.dept_head
  FROM students s JOIN departments d ON s.department = d.department
  ORDER BY s.student_id
""").show()
```

Same result.

```text
INNER JOIN = only the overlap

   students   ┌──────┐   departments
              │ both │
              └──────┘
```

---

### 5. LEFT (OUTER) JOIN

**Keeps ALL rows from the left table.** If there is no match on the right, the right-side columns become **NULL**.

```scala
spark.sql("""
  SELECT s.name, s.department, d.dept_head
  FROM students s
  LEFT JOIN departments d
    ON s.department = d.department
  ORDER BY s.student_id
""").show()
```

**Output**

```text
+-------+----------+----------+
|   name|department| dept_head|
+-------+----------+----------+
|  Alice|       CSE|Dr. Sharma|
|    Bob|       ECE| Dr. Verma|
|Charlie|       CSE|Dr. Sharma|
|  David|        IT| Dr. Singh|
|   Emma|       ECE| Dr. Verma|
|  Frank|      MECH|      NULL|
+-------+----------+----------+
```

**Frank is now present**, with `dept_head = NULL`.

```text
LEFT JOIN = everything from the left + matches from the right

   students ┌─────────┐ departments
            │ ░░░░░░░ │
            │ ░░ both │
            └─────────┘
```

> `LEFT JOIN` and `LEFT OUTER JOIN` mean the same.

**Replace the NULLs with a friendly default**

```scala
spark.sql("""
  SELECT s.name, COALESCE(d.dept_head, 'No Head Assigned') AS dept_head
  FROM students s
  LEFT JOIN departments d ON s.department = d.department
  ORDER BY s.student_id
""").show()
```

```text
+-------+----------------+
|   name|       dept_head|
+-------+----------------+
|  Alice|      Dr. Sharma|
|    Bob|       Dr. Verma|
|Charlie|      Dr. Sharma|
|  David|       Dr. Singh|
|   Emma|       Dr. Verma|
|  Frank|No Head Assigned|
+-------+----------------+
```

---

### 6. RIGHT (OUTER) JOIN

**Keeps ALL rows from the right table.** Unmatched left-side columns become NULL.

```scala
spark.sql("""
  SELECT s.name, d.department, d.dept_head
  FROM students s
  RIGHT JOIN departments d
    ON s.department = d.department
  ORDER BY d.department, s.student_id
""").show()
```

**Output**

```text
+-------+----------+----------+
|   name|department| dept_head|
+-------+----------+----------+
|  Alice|       CSE|Dr. Sharma|
|Charlie|       CSE|Dr. Sharma|
|    Bob|       ECE| Dr. Verma|
|   Emma|       ECE| Dr. Verma|
|   NULL|       EEE|  Dr. Kaur|
|  David|        IT| Dr. Singh|
+-------+----------+----------+
```

- **EEE** appears with `name = NULL` (no student).
- **Frank** is gone (his department MECH is not in the right table).

> A RIGHT JOIN is just a LEFT JOIN with the tables swapped. Most people prefer writing LEFT JOINs.

---

### 7. FULL (OUTER) JOIN

**Keeps ALL rows from BOTH tables.** Missing sides become NULL.

```scala
spark.sql("""
  SELECT s.name, s.department AS s_dept, d.department AS d_dept, d.dept_head
  FROM students s
  FULL OUTER JOIN departments d
    ON s.department = d.department
  ORDER BY s.student_id, d.department
""").show()
```

**Output**

```text
+-------+------+------+----------+
|   name|s_dept|d_dept| dept_head|
+-------+------+------+----------+
|   NULL|  NULL|   EEE|  Dr. Kaur|
|  Alice|   CSE|   CSE|Dr. Sharma|
|    Bob|   ECE|   ECE| Dr. Verma|
|Charlie|   CSE|   CSE|Dr. Sharma|
|  David|    IT|    IT| Dr. Singh|
|   Emma|   ECE|   ECE| Dr. Verma|
|  Frank|  MECH|  NULL|      NULL|
+-------+------+------+----------+
```

- Frank (only in students) → right side NULL.
- EEE (only in departments) → left side NULL.
- The EEE row is at the top because NULL sorts first in ascending order.

We aliased both `department` columns (`s_dept`, `d_dept`) so we can see each side.

```text
FULL OUTER JOIN = everything from both

   students ┌─────────┐ departments
            │ ░░░░░░░░│░░░░░░░░░░░░
            │ ░ both ░│░░░░░░░░░░░░
            └─────────┘
```

---

### 8. LEFT SEMI JOIN

**Returns only the left table's rows that HAVE a match in the right table.** No columns from the right table are returned.

```scala
spark.sql("""
  SELECT s.name, s.department
  FROM students s
  LEFT SEMI JOIN departments d
    ON s.department = d.department
  ORDER BY s.student_id
""").show()
```

**Output**

```text
+-------+----------+
|   name|department|
+-------+----------+
|  Alice|       CSE|
|    Bob|       ECE|
|Charlie|       CSE|
|  David|        IT|
|   Emma|       ECE|
+-------+----------+
```

It works like "does this student's department exist?", a filter, not a data-combining join.

---

### 9. LEFT ANTI JOIN

**Returns only the left table's rows that have NO match in the right table.**

**Students whose department is not listed**

```scala
spark.sql("""
  SELECT s.name, s.department
  FROM students s
  LEFT ANTI JOIN departments d
    ON s.department = d.department
""").show()
```

```text
+-----+----------+
| name|department|
+-----+----------+
|Frank|      MECH|
+-----+----------+
```

**Departments that have no students** (swap the tables)

```scala
spark.sql("""
  SELECT d.department
  FROM departments d
  LEFT ANTI JOIN students s
    ON s.department = d.department
""").show()
```

```text
+----------+
|department|
+----------+
|       EEE|
+----------+
```

> **Use ANTI JOIN for "find records that are missing in the other table".**

---

### 10. CROSS JOIN

**Pairs every row of the left table with every row of the right table** (a Cartesian product). No join condition.

```scala
spark.sql("SELECT COUNT(*) AS rows FROM students CROSS JOIN departments").show()
```

```text
+----+
|rows|
+----+
|  24|
+----+
```

6 students × 4 departments = **24 rows**.

**A small view: one student crossed with all departments**

```scala
spark.sql("""
  SELECT s.name, d.department
  FROM students s CROSS JOIN departments d
  WHERE s.student_id = 101
  ORDER BY d.department
""").show()
```

```text
+-----+----------+
| name|department|
+-----+----------+
|Alice|       CSE|
|Alice|       ECE|
|Alice|       EEE|
|Alice|        IT|
+-----+----------+
```

> ⚠️ Cross joins grow quickly: 1 million × 1 million rows = 1 trillion rows. Use them only on small tables, such as generating all combinations.

---

### 11. Join types — master table

| Join type | Keeps | Unmatched rows become |
| --- | --- | --- |
| `INNER JOIN` | Only matching rows | Dropped |
| `LEFT JOIN` | All left rows + matches | Right columns NULL |
| `RIGHT JOIN` | All right rows + matches | Left columns NULL |
| `FULL OUTER JOIN` | All rows from both | Missing side NULL |
| `LEFT SEMI JOIN` | Left rows that have a match | Right columns not returned |
| `LEFT ANTI JOIN` | Left rows that have **no** match | Right columns not returned |
| `CROSS JOIN` | Every combination | No condition |

**Result sizes in our example**

| Join | Rows | Reason |
| --- | --- | --- |
| INNER | 5 | Frank and EEE dropped |
| LEFT | 6 | All students kept |
| RIGHT | 6 | 5 student-department matches + 1 EEE row with NULL name |
| FULL | 7 | 5 matched + Frank + EEE |
| SEMI | 5 | Students with a department match |
| ANTI | 1 | Only Frank |
| CROSS | 24 | 6 × 4 |

**The Venn way to remember**

```text
INNER :   A ∩ B
LEFT  :   A  + (A ∩ B)
RIGHT :   B  + (A ∩ B)
FULL  :   A ∪ B
SEMI  :   A rows that are in (A ∩ B)
ANTI  :   A − B
```

---

### 12. Table aliases and the ambiguous column error

**Aliases** (`s`, `d`) shorten the code and are **essential** when both tables have a column with the same name.

```scala
spark.sql("""
  SELECT department, name
  FROM students s JOIN departments d ON s.department = d.department
""").show()
```

**Error**

```text
org.apache.spark.sql.AnalysisException: [AMBIGUOUS_REFERENCE]
Reference `department` is ambiguous, could be: [`d`.`department`, `s`.`department`].
```

Both tables have `department`, so Spark cannot decide which one we mean.

**Fix: qualify the column with the alias**

```scala
spark.sql("""
  SELECT s.department, s.name
  FROM students s JOIN departments d ON s.department = d.department
""")
```

> **Rule:** In a join, qualify every column that exists in both tables as `alias.column`. It is good practice to qualify all of them.

---

### 13. Join + WHERE, GROUP BY, ORDER BY

Joins combine with everything we learned.

**Join + WHERE**

```scala
spark.sql("""
  SELECT s.name, d.dept_head
  FROM students s JOIN departments d ON s.department = d.department
  WHERE s.marks > 80
  ORDER BY s.name
""").show()
```

```text
+-------+----------+
|   name| dept_head|
+-------+----------+
|  Alice|Dr. Sharma|
|Charlie|Dr. Sharma|
|   Emma| Dr. Verma|
+-------+----------+
```

**Join + GROUP BY: department report**

```scala
spark.sql("""
  SELECT d.dept_head, d.building,
         COUNT(*) AS students,
         ROUND(AVG(s.marks), 1) AS avg_marks
  FROM students s
  JOIN departments d ON s.department = d.department
  GROUP BY d.dept_head, d.building
  ORDER BY d.dept_head
""").show()
```

```text
+----------+--------+--------+---------+
| dept_head|building|students|avg_marks|
+----------+--------+--------+---------+
|Dr. Sharma| Block A|       2|     88.0|
| Dr. Singh| Block C|       1|     65.0|
| Dr. Verma| Block B|       2|     80.0|
+----------+--------+--------+---------+
```

**Order of execution with a join**

```text
FROM + JOIN  →  WHERE  →  GROUP BY  →  HAVING  →  SELECT  →  ORDER BY  →  LIMIT
```

---

### 14. LEFT JOIN + aggregation (the COUNT trap) ⚠️

**Question:** How many students are in each department, including departments with zero students?

Start from `departments` and LEFT JOIN students.

**Wrong: `COUNT(*)`**

```scala
spark.sql("""
  SELECT d.department, COUNT(*) AS students
  FROM departments d LEFT JOIN students s ON d.department = s.department
  GROUP BY d.department
  ORDER BY d.department
""").show()
```

```text
+----------+--------+
|department|students|
+----------+--------+
|       CSE|       2|
|       ECE|       2|
|       EEE|       1|     ← WRONG: EEE has no students
|        IT|       1|
+----------+--------+
```

`COUNT(*)` counts the NULL-padded row for EEE.

**Correct: count a column from the right table**

```scala
spark.sql("""
  SELECT d.department, COUNT(s.student_id) AS students
  FROM departments d LEFT JOIN students s ON d.department = s.department
  GROUP BY d.department
  ORDER BY d.department
""").show()
```

```text
+----------+--------+
|department|students|
+----------+--------+
|       CSE|       2|
|       ECE|       2|
|       EEE|       0|
|        IT|       1|
+----------+--------+
```

`COUNT(column)` skips NULLs, so EEE correctly shows 0.

> **Rule:** After an outer join, count a **right-table column**, not `COUNT(*)`.

---

### 15. Self join

A **self join** joins a table with itself, using two different aliases. It is useful for comparing rows within the same table.

**Question:** List pairs of students from the same department.

```scala
spark.sql("""
  SELECT a.name AS student1, b.name AS student2, a.department
  FROM students a
  JOIN students b
    ON a.department = b.department
   AND a.student_id < b.student_id
  ORDER BY a.student_id
""").show()
```

```text
+--------+--------+----------+
|student1|student2|department|
+--------+--------+----------+
|   Alice| Charlie|       CSE|
|     Bob|    Emma|       ECE|
+--------+--------+----------+
```

The extra condition `a.student_id < b.student_id` stops a student pairing with themselves and avoids duplicate mirrored pairs (Alice–Charlie and Charlie–Alice).

---

### 16. `USING` and the broadcast hint

**`USING (column)`**: shortcut when both tables have the same key name.

```scala
spark.sql("""
  SELECT s.name, d.dept_head
  FROM students s JOIN departments d USING (department)
  ORDER BY s.student_id
""").show()
```

```text
+-------+----------+
|   name| dept_head|
+-------+----------+
|  Alice|Dr. Sharma|
|    Bob| Dr. Verma|
|Charlie|Dr. Sharma|
|  David| Dr. Singh|
|   Emma| Dr. Verma|
+-------+----------+
```

**Broadcast hint (performance idea)**

When one table is **small**, Spark can send a copy of it to every worker so the large table does not need to be shuffled. This is a **broadcast join**.

```scala
spark.sql("""
  SELECT /*+ BROADCAST(d) */ s.name, d.building
  FROM students s JOIN departments d ON s.department = d.department
  ORDER BY s.student_id
""").show()
```

```text
+-------+--------+
|   name|building|
+-------+--------+
|  Alice| Block A|
|    Bob| Block B|
|Charlie| Block A|
|  David| Block C|
|   Emma| Block B|
+-------+--------+
```

The result is identical; only the execution strategy differs. Spark often chooses a broadcast join automatically for small tables.

> **Viva line:** A broadcast join ships the small table to all executors, avoiding a shuffle of the big table.

---

### 17. Joining three tables

Add a many-to-many structure: students ↔ courses through an enrollment table.

```scala
val courseDF = Seq(("C1", "Big Data"), ("C2", "Databases"), ("C3", "Networks"))
  .toDF("course_id", "course_name")

val enrollDF = Seq((101, "C1"), (101, "C2"), (102, "C3"), (103, "C1"), (105, "C2"))
  .toDF("student_id", "course_id")

courseDF.createOrReplaceTempView("courses")
enrollDF.createOrReplaceTempView("enrollments")
```

```text
enrollments links the other two:
students.student_id ── enrollments.student_id
                       enrollments.course_id ── courses.course_id
```

**Student names with their courses**

```scala
spark.sql("""
  SELECT s.name, c.course_name
  FROM students s
  JOIN enrollments e ON s.student_id = e.student_id
  JOIN courses c     ON e.course_id  = c.course_id
  ORDER BY s.student_id, c.course_name
""").show()
```

```text
+-------+-----------+
|   name|course_name|
+-------+-----------+
|  Alice|   Big Data|
|  Alice|  Databases|
|    Bob|   Networks|
|Charlie|   Big Data|
|   Emma|  Databases|
+-------+-----------+
```

**Students per course**

```scala
spark.sql("""
  SELECT c.course_name, COUNT(*) AS enrolled
  FROM enrollments e JOIN courses c ON e.course_id = c.course_id
  GROUP BY c.course_name
  ORDER BY c.course_name
""").show()
```

```text
+-----------+--------+
|course_name|enrolled|
+-----------+--------+
|   Big Data|       2|
|  Databases|       2|
|   Networks|       1|
+-----------+--------+
```

**Students who are not enrolled in any course** (LEFT JOIN + `IS NULL` pattern)

```scala
spark.sql("""
  SELECT s.name
  FROM students s
  LEFT JOIN enrollments e ON s.student_id = e.student_id
  WHERE e.student_id IS NULL
  ORDER BY s.name
""").show()
```

```text
+-----+
| name|
+-----+
|David|
|Frank|
+-----+
```

> This "LEFT JOIN … WHERE right.key IS NULL" pattern does the same job as a LEFT ANTI JOIN.

---

### 18. DataFrame API joins

**Syntax**

```scala
leftDF.join(rightDF, joinCondition, joinType)
```

**18.1 Join on a common column name: the simplest form**

```scala
studentsDF.join(deptDF, "department").orderBy("student_id").show()
```

```text
+----------+----------+-------+---+-----+----------+--------+
|department|student_id|   name|age|marks| dept_head|building|
+----------+----------+-------+---+-----+----------+--------+
|       CSE|       101|  Alice| 21|   85|Dr. Sharma| Block A|
|       ECE|       102|    Bob| 22|   72| Dr. Verma| Block B|
|       CSE|       103|Charlie| 21|   91|Dr. Sharma| Block A|
|        IT|       104|  David| 23|   65| Dr. Singh| Block C|
|       ECE|       105|   Emma| 22|   88| Dr. Verma| Block B|
+----------+----------+-------+---+-----+----------+--------+
```

The shared key `department` appears **once**, and the default join type is inner.

**18.2 With a join type**

```scala
studentsDF.join(deptDF, Seq("department"), "left").orderBy("student_id").show()
```

```text
+----------+----------+-------+---+-----+----------+--------+
|department|student_id|   name|age|marks| dept_head|building|
+----------+----------+-------+---+-----+----------+--------+
|       CSE|       101|  Alice| 21|   85|Dr. Sharma| Block A|
|       ECE|       102|    Bob| 22|   72| Dr. Verma| Block B|
|       CSE|       103|Charlie| 21|   91|Dr. Sharma| Block A|
|        IT|       104|  David| 23|   65| Dr. Singh| Block C|
|       ECE|       105|   Emma| 22|   88| Dr. Verma| Block B|
|      MECH|       106|  Frank| 22|   59|      NULL|    NULL|
+----------+----------+-------+---+-----+----------+--------+
```

**Join type strings**

| String | Join |
| --- | --- |
| `"inner"` (default) | INNER |
| `"left"` / `"left_outer"` | LEFT |
| `"right"` / `"right_outer"` | RIGHT |
| `"full"` / `"outer"` / `"full_outer"` | FULL OUTER |
| `"left_semi"` | LEFT SEMI |
| `"left_anti"` | LEFT ANTI |

**Semi and anti**

```scala
studentsDF.join(deptDF, Seq("department"), "left_anti").show()
```

```text
+----------+----------+-----+---+-----+
|department|student_id| name|age|marks|
+----------+----------+-----+---+-----+
|      MECH|       106|Frank| 22|   59|
+----------+----------+-----+---+-----+
```

**18.3 Cross join**

```scala
studentsDF.crossJoin(deptDF).count()
```

```text
res34: Long = 24
```

**18.4 Join condition with `===` (when key names differ)**

Rename the key in one table to simulate different names:

```scala
val e = deptDF.withColumnRenamed("department", "dept_code")

studentsDF.join(e, studentsDF("department") === e("dept_code"))
  .select("name", "dept_code", "dept_head")
  .orderBy("name")
  .show()
```

```text
+-------+---------+----------+
|   name|dept_code| dept_head|
+-------+---------+----------+
|  Alice|      CSE|Dr. Sharma|
|    Bob|      ECE| Dr. Verma|
|Charlie|      CSE|Dr. Sharma|
|  David|       IT| Dr. Singh|
|   Emma|      ECE| Dr. Verma|
+-------+---------+----------+
```

**18.5 Same key name with `===` creates duplicate columns**

```scala
studentsDF.join(deptDF, studentsDF("department") === deptDF("department"), "inner")
  .orderBy("student_id").show()
```

```text
+----------+-------+----------+---+-----+----------+----------+--------+
|student_id|   name|department|age|marks|department| dept_head|building|
+----------+-------+----------+---+-----+----------+----------+--------+
|       101|  Alice|       CSE| 21|   85|       CSE|Dr. Sharma| Block A|
|       102|    Bob|       ECE| 22|   72|       ECE| Dr. Verma| Block B|
|       103|Charlie|       CSE| 21|   91|       CSE|Dr. Sharma| Block A|
|       104|  David|        IT| 23|   65|        IT| Dr. Singh| Block C|
|       105|   Emma|       ECE| 22|   88|       ECE| Dr. Verma| Block B|
+----------+-------+----------+---+-----+----------+----------+--------+
```

`department` appears **twice**. Selecting it by name then fails with the ambiguous error. Fixes:

```scala
// Fix 1: pick columns through the DataFrame that owns them
studentsDF.join(deptDF, studentsDF("department") === deptDF("department"))
  .select(studentsDF("name"), deptDF("dept_head"))
  .orderBy("name").show()

// Fix 2: DataFrame aliases
studentsDF.as("s").join(deptDF.as("d"), col("s.department") === col("d.department"))
  .select(col("s.name"), col("d.dept_head"))
  .orderBy("name").show()

// Fix 3: join using Seq("department") so only one key column is kept
```

**Output (Fix 1 or 2)**

```text
+-------+----------+
|   name| dept_head|
+-------+----------+
|  Alice|Dr. Sharma|
|    Bob| Dr. Verma|
|Charlie|Dr. Sharma|
|  David| Dr. Singh|
|   Emma| Dr. Verma|
+-------+----------+
```

**SQL vs DataFrame join**

| SQL | DataFrame |
| --- | --- |
| `FROM a JOIN b ON a.k = b.k` | `a.join(b, a("k") === b("k"))` |
| `... USING (k)` | `a.join(b, "k")` or `a.join(b, Seq("k"), "left")` |
| `LEFT JOIN` | `"left"` |
| `CROSS JOIN` | `a.crossJoin(b)` |

---

### 19. Common errors

| Mistake | Symptom | Fix |
| --- | --- | --- |
| Forgetting the `ON` condition | Wrong or huge result | Always write the join condition (except CROSS JOIN) |
| Ambiguous column name | `AMBIGUOUS_REFERENCE` | Qualify with `alias.column` |
| Using `COUNT(*)` after a LEFT JOIN | Counts 1 for unmatched groups | `COUNT(right_table.column)` |
| Expecting NULL keys to match | NULL never equals NULL in a join | Handle NULLs before joining |
| Using `=` in the DataFrame join | Wrong comparison | Use `===` for column equality |
| Same key name + `===` | Duplicate `department` column | Use `Seq("department")` or aliases |
| Wrong table order for LEFT JOIN | Missing rows you expected | The "kept" table must be on the **left** |
| Cross join on large tables | Out of memory / very slow | Avoid; filter first |
| Wrong join type | Missing/extra rows | Re-read the question: *"all X, even without Y"* → LEFT JOIN from X |

**Choosing a join: the question method**

| Question says | Use |
| --- | --- |
| "students with their department head" | INNER JOIN |
| "all students, even if department missing" | LEFT JOIN (students on left) |
| "all departments, even with no students" | LEFT JOIN (departments on left) or RIGHT JOIN |
| "everything from both" | FULL OUTER JOIN |
| "students not in any department" / "never enrolled" | LEFT ANTI JOIN |
| "students who have a matching department" (no extra columns) | LEFT SEMI JOIN |
| "all combinations" | CROSS JOIN |

---

### 20. Practical scenarios — Phase 7 practice

1. **Student names with department heads (inner join)**
   ```scala
   spark.sql("SELECT s.name, d.dept_head FROM students s JOIN departments d ON s.department = d.department").show()
   ```
2. **All students, with NULL for missing departments**
   ```scala
   spark.sql("SELECT s.name, d.dept_head FROM students s LEFT JOIN departments d ON s.department = d.department").show()
   ```
3. **All departments, including those without students**
   ```scala
   spark.sql("SELECT d.department, s.name FROM departments d LEFT JOIN students s ON d.department = s.department").show()
   ```
4. **Full outer join**
   ```scala
   spark.sql("SELECT s.name, d.dept_head FROM students s FULL OUTER JOIN departments d ON s.department = d.department").show()
   ```
5. **Students whose department is not in the departments table**
   ```scala
   spark.sql("SELECT s.name FROM students s LEFT ANTI JOIN departments d ON s.department = d.department").show()
   ```
6. **Students with marks above 80 and their department head**
7. **Average marks per department head** (join + group by, Section 13)
8. **Number of students per department, including zero** (Section 14)
9. **Pairs of students from the same department** (self join, Section 15)
10. **Student–course listing** (three tables, Section 17)
11. **Students not enrolled in any course** (Section 17)
12. **Inner join using the DataFrame API**
    ```scala
    studentsDF.join(deptDF, "department").show()
    ```

---

### 21. Phase 7 — viva questions and answers

#### Q1. What is a join?

A join combines rows from two tables based on a related column.

#### Q2. What is the join key / join condition?

The join key is the common column; the join condition (the `ON` clause) states how rows match.

#### Q3. What is an INNER JOIN?

It returns only rows that have matching values in both tables.

#### Q4. What is a LEFT JOIN?

It returns all rows from the left table, with matching right-table values or NULL when no match exists.

#### Q5. What is a RIGHT JOIN?

It returns all rows from the right table, with matching left-table values or NULL.

#### Q6. What is a FULL OUTER JOIN?

It returns all rows from both tables; unmatched sides are filled with NULL.

#### Q7. What is a CROSS JOIN?

It pairs every row of the first table with every row of the second (Cartesian product). Rows = m × n.

#### Q8. What is a LEFT SEMI JOIN?

It returns left-table rows that have a match in the right table, without any right-table columns.

#### Q9. What is a LEFT ANTI JOIN?

It returns left-table rows that have **no** match in the right table.

#### Q10. Which join gives rows only from one table, but filtered by the other?

LEFT SEMI and LEFT ANTI joins.

#### Q11. What is a self join?

A join of a table with itself using two aliases.

#### Q12. Why are table aliases used?

They shorten code and resolve ambiguity between columns with the same name.

#### Q13. What is the "ambiguous reference" error?

It occurs when a column name exists in both joined tables and is not qualified with an alias.

#### Q14. How do we find rows in A that are missing in B?

LEFT ANTI JOIN, or LEFT JOIN with `WHERE b.key IS NULL`.

#### Q15. Why is `COUNT(*)` wrong after a LEFT JOIN for "count of matches"?

Unmatched rows still produce one NULL-padded row, which `COUNT(*)` counts. Use `COUNT(right.column)`.

#### Q16. What is a broadcast join?

A strategy where the small table is copied to every executor so the big table need not be shuffled.

#### Q17. How do we write an inner join in the DataFrame API?

`left.join(right, left("k") === right("k"))`, or `left.join(right, "k")` for a shared column name.

#### Q18. What are the DataFrame join type strings?

`inner`, `left`, `right`, `full`, `left_semi`, `left_anti` (and their `_outer` variants).

#### Q19. How do we join three tables?

Chain joins: `FROM a JOIN b ON ... JOIN c ON ...`.

#### Q20. Do NULL keys match in a join?

No. A NULL never equals another NULL in a join condition.

---

### 22. Phase 7 — final master code revision

```scala
import spark.implicits._
import org.apache.spark.sql.functions._

val studentsDF = Seq(
  (101, "Alice", "CSE", 21, 85), (102, "Bob", "ECE", 22, 72),
  (103, "Charlie", "CSE", 21, 91), (104, "David", "IT", 23, 65),
  (105, "Emma", "ECE", 22, 88), (106, "Frank", "MECH", 22, 59)
).toDF("student_id", "name", "department", "age", "marks")

val deptDF = Seq(
  ("CSE", "Dr. Sharma", "Block A"), ("ECE", "Dr. Verma", "Block B"),
  ("IT", "Dr. Singh", "Block C"), ("EEE", "Dr. Kaur", "Block D")
).toDF("department", "dept_head", "building")

studentsDF.createOrReplaceTempView("students")
deptDF.createOrReplaceTempView("departments")

// INNER
spark.sql("SELECT s.name, d.dept_head FROM students s JOIN departments d ON s.department = d.department").show()
// LEFT
spark.sql("SELECT s.name, d.dept_head FROM students s LEFT JOIN departments d ON s.department = d.department").show()
// RIGHT
spark.sql("SELECT s.name, d.department FROM students s RIGHT JOIN departments d ON s.department = d.department").show()
// FULL OUTER
spark.sql("SELECT s.name, d.department FROM students s FULL OUTER JOIN departments d ON s.department = d.department").show()
// SEMI / ANTI
spark.sql("SELECT s.name FROM students s LEFT SEMI JOIN departments d ON s.department = d.department").show()
spark.sql("SELECT s.name FROM students s LEFT ANTI JOIN departments d ON s.department = d.department").show()
// CROSS
spark.sql("SELECT COUNT(*) FROM students CROSS JOIN departments").show()
// JOIN + GROUP BY
spark.sql("""
  SELECT d.dept_head, COUNT(*) AS students, ROUND(AVG(s.marks),1) AS avg_marks
  FROM students s JOIN departments d ON s.department = d.department
  GROUP BY d.dept_head ORDER BY d.dept_head
""").show()

// DataFrame API
studentsDF.join(deptDF, "department").show()
studentsDF.join(deptDF, Seq("department"), "left").show()
studentsDF.join(deptDF, Seq("department"), "left_anti").show()
studentsDF.crossJoin(deptDF).count()
```

**Join quick-recall card**

| Join | Keeps |
| --- | --- |
| INNER | matching rows only |
| LEFT | all left + matches |
| RIGHT | all right + matches |
| FULL | all rows of both |
| SEMI | left rows with match |
| ANTI | left rows without match |
| CROSS | all combinations |

**Phase 7 completed**

---

> **Next → Phase 8: Complete Practical Integration** (end-to-end RDD → DataFrame → SQL → join → group → report). File: `8_Unit4_Complete_Practical_Integration.md`
