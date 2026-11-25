# INT315 — Cluster Computing: The Complete Practical Guide and Viva Bank

> **One file, whole syllabus.** This guide takes you from **installing Spark** to **machine learning**, as complete, runnable **practicals** (Aim → Theory → Steps → Code → Expected output → Explanation → Common errors), followed by a **viva question bank for every unit** with full, interview-grade answers.
>
> Written for **two audiences at once**: the **university examiner** (precise definitions, clean steps, correct output) and the **industry interviewer** (why it works, trade-offs, what goes wrong in production).

---

## 0. How to use this guide

| If you have… | Do this |
| --- | --- |
| **A lab exam tomorrow** | Read Section 0.4, then run Practicals 1 → 9 once each (2 to 3 hours total) |
| **A viva/interview** | Read Part B (the question bank). Each answer is written to be **spoken**, in a logical order |
| **Time to learn properly** | Do the practicals in order, then use the viva bank to test yourself |

### 0.1 Your environment

| Component | Version | Used for |
| --- | --- | --- |
| Operating system | Windows (CMD / PowerShell) | All commands |
| Java | 1.8.0_501 | Runs Spark, Kafka, ZooKeeper |
| Spark | 3.5.9 | Core, SQL, GraphX, Streaming, MLlib |
| Scala | 2.13.8 | Unit II to V practicals (`spark-shell`) |
| Python | 3.10.11 | PySpark (Unit VI), Kafka scripts |
| Kafka | 3.4.0 | Unit V |

### 0.2 One-time folder and data setup

Create `C:\sparklabs` and put these files in it.

**`C:\sparklabs\words.txt`**

```text
spark is fast
spark is in memory
hadoop uses mapreduce
spark beats mapreduce for iterative jobs
```

**`C:\sparklabs\students.txt`**

```text
101,Alice,CSE,21,85
102,Bob,ECE,22,72
103,Charlie,CSE,21,91
104,David,IT,23,65
105,Emma,ECE,22,88
```

**`C:\sparklabs\make_data.py`** (200 reproducible students for the ML practical)

```python
import random

def make_students(n=200, seed=42):
    random.seed(seed)
    rows = []
    for i in range(n):
        hours = round(random.uniform(1, 10), 1)
        attendance = round(random.uniform(50, 100), 1)
        prev = round(random.uniform(35, 90), 1)
        noise = random.gauss(0, 3)
        final = 4 * hours + 0.25 * attendance + 0.35 * prev + 5 + noise
        final = round(min(100.0, max(0.0, final)), 1)
        passed = 1 if final >= 68 else 0
        rows.append((i + 1, hours, attendance, prev, final, passed))
    return rows
```

> **Windows path rule:** in Scala and Spark use forward slashes (`C:/sparklabs/words.txt`). Backslashes are escape characters and cause errors.

### 0.3 How the outputs were verified (honest note)

- **Scala, RDD, GraphX, feature-extraction, SQL, Structured Streaming and PySpark ML** outputs were produced by running the code in a real Spark 3.5 installation. The numbers are genuine.
- **Kafka commands and their console outputs** follow the official Kafka 3.4.0 syntax; a broker could not be run in my environment, so those outputs are labelled **⚠️ Typical output** (ports, IDs, timestamps differ).
- **Scala 2.13 vs 2.12:** my test machine used Scala 2.12. Where your 2.13.8 behaves differently (for example `Stream` is replaced by `LazyList`, or a collection prints as `ArraySeq` instead of `WrappedArray`) the text says so.

### 0.4 The practical map

| # | Unit | Practical | Official lab item |
| --- | --- | --- | --- |
| **P1** | I | Install and verify Spark, run a job, Spark vs Hadoop | |
| **P2** | II | Scala programming fundamentals | |
| **P3** | III | RDDs: operations, lazy evaluation, persistence, shared variables | |
| **P4** | III | **GraphX** | ✅ *Demonstrate the use of GraphX* |
| **P5** | III | Feature extraction and transformation | |
| **P6** | IV | **Spark SQL** | ✅ *Demonstrate the use of Spark SQL* |
| **P7** | V | **Kafka producer and consumer** | ✅ *Demonstrate the use of producer and consumer in Kafka* |
| **P8** | V | **Spark Streaming** (DStreams, Structured Streaming, Kafka) | ✅ *Demonstrate the use of Spark Streaming* |
| **P9** | VI | **MLlib** with PySpark | ✅ *Demonstrate the use of MLlib* |
| **P10** | All | Integrated capstone pipeline | |

### 0.5 The lab-exam routine (use this every time)

```text
 1. START      environment check (java -version, spark-shell banner)             1 min
 2. STATE      write the AIM in one line                                          30 s
 3. BUILD      type the code in small blocks; run each; read the output           bulk of time
 4. EXPLAIN    say what each block did, in plain words ("this is a transformation, lazy…")
 5. EXTEND     be ready to change one thing (filter, another column, another metric)
 6. CLOSE      mention the unit's key viva points (listed under each practical)
```

> **What examiners reward:** correct output, clean steps, and the ability to **explain** the code. What interviewers reward: knowing **why** (lazy evaluation, shuffles, checkpointing, leakage) and **what fails in production**.

---

# PART A — PRACTICALS

---

## P1 — Installing, verifying and running Spark; Spark vs Hadoop (Unit I)

### Aim

To install Apache Spark as a standalone user, verify the installation, run Spark interactively and as a submitted application, and compare Spark with Hadoop MapReduce.

### Theory

**Limitations of MapReduce (why Spark was created)**

| Limitation | What it means |
| --- | --- |
| **Disk I/O between every stage** | Each MapReduce job writes its output to HDFS, and the next job reads it back: very slow for multi-step work |
| **Only two operations** (map and reduce) | Complex pipelines become many chained jobs |
| **Poor for iterative algorithms** | Machine learning and graph algorithms repeat the same data many times, re-reading from disk every iteration |
| **High latency** | Minutes, not seconds: unsuitable for interactive or real-time use |
| **No interactive shell** | Developers wait for each job |
| **Verbose code** | A word count needs dozens of lines of Java |

**How Spark answers**

```text
 MapReduce:  HDFS → Map → disk → Reduce → HDFS → Map → disk → Reduce → HDFS
 Spark:      HDFS → [ transformation → transformation → transformation ]  (kept in MEMORY) → result
```

| Spark feature | Benefit |
| --- | --- |
| **In-memory computation** | Intermediate data stays in RAM; up to 10 to 100× faster for iterative work |
| **DAG execution engine** | The whole job is planned as a graph and optimized (stages) |
| **Lazy evaluation** | Work happens only when an action needs a result |
| **Rich API** | `map`, `filter`, `join`, `groupBy`… on RDDs, DataFrames, SQL |
| **Unified stack** | SQL, Streaming, MLlib, GraphX in one engine |
| **Multi-language** | Scala, Java, Python, R, SQL |
| **Fault tolerance** | Lost data recomputed from **lineage**, not replicated everywhere |
| **Runs anywhere** | Local, Standalone, YARN, Kubernetes; reads HDFS, S3, Kafka, JDBC |

**Batch vs real-time analytics**

| | Batch | Real-time (stream) |
| --- | --- | --- |
| Data | Finite, already collected | Continuous, unbounded |
| Latency | Minutes to hours | Milliseconds to seconds |
| Example | Nightly sales report | Live fraud alert |
| Spark tool | RDD / DataFrame / SQL | Spark Streaming / Structured Streaming |

**Spark vs Hadoop ecosystem**

| | **Hadoop (MapReduce)** | **Spark** |
| --- | --- | --- |
| Processing | Batch, disk-based | Batch + streaming + interactive + ML + graph, in-memory |
| Storage | HDFS (its own) | **None of its own**; uses HDFS, S3, local, Kafka… |
| Speed | Slower (disk each step) | Much faster (memory) |
| Ease | Verbose Java | Concise APIs, interactive shell |
| Real-time | No | Yes |
| Cluster manager | YARN | Standalone, YARN, Kubernetes |
| **Relationship** | | Spark **complements** Hadoop: it commonly runs on **YARN** and reads **HDFS** |

> **Important nuance for interviews:** Spark is *not* "replacing Hadoop". It replaces **MapReduce** as the processing engine. HDFS and YARN are still widely used underneath.

### Requirements

JDK 8, the Spark 3.5.9 binary package (the "pre-built for Hadoop 3" Scala 2.13 build), and for Windows the Hadoop helper `winutils.exe`.

### Steps

**Step 1: install Java and set `JAVA_HOME`**

```powershell
java -version
echo %JAVA_HOME%
```

```text
java version "1.8.0_501"
Java(TM) SE Runtime Environment (build 1.8.0_501-b08)
```

**Step 2: install Spark as a standalone user (no admin cluster needed)**

1. Download the Spark 3.5.9 pre-built package from `spark.apache.org/downloads`.
2. Extract it to a **short path with no spaces**, for example `C:\spark`.
3. Set environment variables: `SPARK_HOME=C:\spark`, and add `C:\spark\bin` to `PATH`.
4. **Windows only:** place `winutils.exe` in `C:\hadoop\bin` and set `HADOOP_HOME=C:\hadoop` (Spark needs it for some local-file operations).

**Step 3: verify with the Scala shell**

```powershell
spark-shell
```

The banner shows the versions:

```text
Spark context Web UI available at http://localhost:4040
Spark context available as 'sc' (master = local[*], app id = local-...).
Spark session available as 'spark'.
Using Scala version 2.13.8 (Java HotSpot(TM) 64-Bit Server VM, Java 1.8.0_501)
```

Type a first computation:

```scala
sc.parallelize(1 to 100).sum()
```

```text
res0: Double = 5050.0
```

Open **`http://localhost:4040`** in a browser: this is the **Spark UI** (jobs, stages, storage, executors).

**Step 4: verify the Python shell**

```powershell
pyspark
```

```python
>>> spark.version
'3.5.9'
>>> spark.range(5).count()
5
```

**Step 5: run a Spark application with `spark-submit`**

Save as `C:\sparklabs\wc_submit.py`:

```python
from pyspark.sql import SparkSession

spark = SparkSession.builder.appName("WordCount").master("local[2]").getOrCreate()
spark.sparkContext.setLogLevel("ERROR")
sc = spark.sparkContext

counts = (sc.textFile("C:/sparklabs/words.txt")
            .flatMap(lambda line: line.split(" "))
            .map(lambda word: (word, 1))
            .reduceByKey(lambda a, b: a + b))

for word, count in sorted(counts.collect(), key=lambda x: (-x[1], x[0]))[:4]:
    print(word, count)

print("partitions:", counts.getNumPartitions())
spark.stop()
```

```powershell
spark-submit --master local[2] C:\sparklabs\wc_submit.py
```

**Output (verified)**

```text
spark 3
is 2
mapreduce 2
beats 1
partitions: 2
```

| `spark-submit` part | Meaning |
| --- | --- |
| `--master local[2]` | Run locally with 2 threads |
| `--master spark://HOST:7077` | Run on a Standalone cluster |
| `--master yarn` | Run on Hadoop YARN |
| `--deploy-mode client / cluster` | Where the driver runs |
| `--executor-memory 2g`, `--num-executors`, `--conf` | Resources and settings |

**Step 6: Spark Standalone cluster (concept practical; ⚠️ not run in my environment)**

```powershell
REM Terminal 1: start the master (UI at http://localhost:8080)
spark-class org.apache.spark.deploy.master.Master

REM Terminal 2: start a worker and attach it to the master
spark-class org.apache.spark.deploy.worker.Worker spark://YOUR-PC:7077

REM Terminal 3: run the shell on the cluster
spark-shell --master spark://YOUR-PC:7077
```

```text
 Driver (your program) ──▶ Master (resource manager) ──▶ Workers (run Executors) ──▶ Tasks
```

**Spark modes**

| Mode | Use |
| --- | --- |
| `local[N]` | Development on one machine |
| **Standalone** | Spark's own simple cluster manager |
| **YARN** | Share a Hadoop cluster |
| Kubernetes | Containers in the cloud |

### Word count: MapReduce vs Spark (the classic comparison)

| MapReduce | Spark (Scala) |
| --- | --- |
| ~60 lines of Java (Mapper class, Reducer class, Driver) | `sc.textFile(path).flatMap(_.split(" ")).map((_, 1)).reduceByKey(_ + _)` |
| Each stage writes to disk | Stays in memory |
| Separate job to chain more steps | Just add another transformation |

### Common errors

| Error | Cause | Fix |
| --- | --- | --- |
| `JAVA_HOME is not set` | Missing variable | Set to the JDK folder |
| `'spark-shell' is not recognized` | PATH | Add `C:\spark\bin` |
| `winutils.exe` / `HADOOP_HOME` errors | Windows helper missing | Install `winutils.exe`, set `HADOOP_HOME` |
| Spark UI not opening | Port busy or shell closed | Check `4040` (or `4041`); keep the shell open |
| Spaces in the install path | Class-path issues | Use `C:\spark` |

### Viva pointers for P1

*Limitations of MapReduce; features of Spark; Spark vs Hadoop; batch vs real-time; standalone installation; Spark UI; modes of running Spark.*

---

## P2 — Scala programming fundamentals (Unit II)

### Aim

To learn the Scala features needed for Spark: variables and types, type inference, operators and methods, functions, collections (immutable vs mutable), lists, maps and lazy streams.

### Theory

Scala is a **statically typed, object-oriented and functional** language on the JVM. Spark is written in Scala, so Scala gives the most natural Spark API.

| Feature | Meaning |
| --- | --- |
| **Statically typed with inference** | Types are checked at compile time but usually **inferred**, so code stays short |
| **Everything is an object** | Even `5` and `'a'` have methods |
| **Functional** | Functions are values; `map`, `filter`, `reduce` operate on collections |
| **Immutable by default** | `val` cannot be reassigned; immutable collections are safer in parallel code |
| **Pattern matching, case classes** | Concise data handling |
| **JVM interoperability** | Uses any Java library |

### Steps and code (run in `spark-shell`; every output below is real)

**1. Variables: `val` vs `var`**

```scala
val name = "Spark"        // immutable (cannot be reassigned)
var count = 10            // mutable
count = count + 5
println(s"$name has count $count")
```

```text
Spark has count 15
```

`name = "x"` would be a compile error (*reassignment to val*). The `s"..."` prefix enables **string interpolation**.

**2. Basic data types and literals**

```scala
val x = 42; val y = 3.14; val z = 'A'; val b = true; val big = 5000000000L; val f = 2.5f
println(x.getClass.getSimpleName + " " + y.getClass.getSimpleName + " " + z.getClass.getSimpleName + " " +
        b.getClass.getSimpleName + " " + big.getClass.getSimpleName + " " + f.getClass.getSimpleName)
```

```text
int double char boolean long float
```

| Type | Literal | Notes |
| --- | --- | --- |
| `Int` | `42` | 32-bit |
| `Long` | `5000000000L` | 64-bit, needs `L` |
| `Double` | `3.14` | default decimal |
| `Float` | `2.5f` | needs `f` |
| `Char` | `'A'` | single quotes |
| `String` | `"Spark"` | double quotes |
| `Boolean` | `true` | |

**3. Type inference** (the compiler works the type out)

```scala
val t: Int = 7                 // explicit
val inferred = List(1, 2, 3)   // inferred as List[Int]
```

**4. Operators are methods**

```scala
val a = 10 + 3; val q = 10 / 3; val r = 10 % 3; val d = 10 / 3.0
println(s"$a $q $r $d")
println(10 > 3 && 5 < 2 || true)
println("abc".toUpperCase + " " + "hello".length + " " + "Scala".substring(1, 4) + " " + "a,b,c".split(",").toList)
```

```text
13 3 1 3.3333333333333335
true
ABC 5 cal List(a, b, c)
```

`10 / 3` is **integer** division (3); `10 / 3.0` is a Double. In Scala, `a + b` is really `a.+(b)`: operators are method calls.

**5. Functions**

```scala
def square(n: Int): Int = n * n
def greet(name: String, greeting: String = "Hello"): String = s"$greeting, $name!"
val add = (a: Int, b: Int) => a + b              // anonymous function (lambda)
def applyTwice(f: Int => Int, v: Int): Int = f(f(v))   // higher-order function

println(square(5) + " " + greet("Mate") + " " + greet("Mate", "Hi"))
println(add(2, 3))
println(applyTwice(_ * 2, 5))
```

```text
25 Hello, Mate! Hi, Mate!
5
20
```

| Concept | Example |
| --- | --- |
| Named function with return type | `def square(n: Int): Int` |
| Default parameter | `greeting: String = "Hello"` |
| Anonymous function | `(a: Int, b: Int) => a + b` |
| Higher-order function | takes a function as an argument |
| Placeholder syntax | `_ * 2` means `x => x * 2` |

**6. List: an immutable linked list**

```scala
val nums = List(1, 2, 3, 4, 5, 6)
println(nums.map(_ * 2))
println(nums.filter(_ % 2 == 0))
println(nums.reduce(_ + _) + " " + nums.foldLeft(0)(_ + _))
println(nums.head + " " + nums.tail + " " + nums.take(3) + " " + nums.sum + " " + nums.max)
println(0 :: nums)                // prepend
println(nums ::: List(7, 8))      // concatenate
```

```text
List(2, 4, 6, 8, 10, 12)
List(2, 4, 6)
21 21
1 List(2, 3, 4, 5, 6) List(1, 2, 3) 21 6
List(0, 1, 2, 3, 4, 5, 6)
List(1, 2, 3, 4, 5, 6, 7, 8)
```

More list operations:

```scala
println(List(3, 1, 2).sorted + " " + List(1, 2, 2, 3).distinct + " " + List("a", "b").zip(List(1, 2)) + " " + List(1, 2, 3).mkString("-"))
println(List(List(1, 2), List(3)).flatten + " " + List("a b", "c").flatMap(_.split(" ")))
val (evens, odds) = nums.partition(_ % 2 == 0)
println(evens + " " + odds)
```

```text
List(1, 2, 3) List(1, 2, 3) List((a,1), (b,2)) 1-2-3
List(1, 2, 3) List(a, b, c)
List(2, 4, 6) List(1, 3, 5)
```

**7. Map: key-value pairs**

```scala
val m = Map("a" -> 1, "b" -> 2)
println(m + ("c" -> 3))                  // returns a NEW map
println(m.getOrElse("z", 0))
println(m.map { case (k, v) => (k, v * 10) })
```

```text
Map(a -> 1, b -> 2, c -> 3)
0
Map(a -> 10, b -> 20)
```

**8. Mutable vs immutable collections**

```scala
val imm = List(1, 2, 3)
val newList = imm :+ 4                   // creates a new list; imm is unchanged
println(imm + " " + newList)

import scala.collection.mutable
val buf = mutable.ListBuffer(1, 2, 3)
buf += 4; buf.remove(0)                  // changes the SAME object
println(buf)

val mm = mutable.Map("a" -> 1); mm("b") = 2; mm += ("c" -> 3)
println(mm.toList.sortBy(_._1))

val arr = Array(1, 2, 3); arr(0) = 99    // arrays are mutable
println(arr.toList)
```

```text
List(1, 2, 3) List(1, 2, 3, 4)
ListBuffer(2, 3, 4)
List((a,1), (b,2), (c,3))
List(99, 2, 3)
```

| | **Immutable** (`scala.collection.immutable`, default) | **Mutable** (`scala.collection.mutable`) |
| --- | --- | --- |
| Changeable in place? | **No**: operations return a new collection | **Yes** |
| Thread safety | Safe to share | Needs care |
| Examples | `List`, `Vector`, `Map`, `Set` | `ListBuffer`, `ArrayBuffer`, `mutable.Map`, `Array` |
| Default in Scala? | ✅ | Must be imported |
| Spark preference | ✅ (functional, parallel-safe) | Only locally |

**9. Streams: lazy sequences (Unit II "streams")**

```scala
val s = Stream.from(1).map(_ * 2)                 // an INFINITE stream, computed lazily
println(s.take(5).toList)

val fib: Stream[BigInt] = BigInt(0) #:: BigInt(1) #:: fib.zip(fib.tail).map { case (a, b) => a + b }
println(fib.take(10).toList)
```

```text
List(2, 4, 6, 8, 10)
List(0, 1, 1, 2, 3, 5, 8, 13, 21, 34)
```

A `Stream` computes elements **only when needed** (lazy), so an infinite sequence is possible. **Scala 2.13 note:** `Stream` is deprecated; use **`LazyList`** (same idea, same `#::` syntax): `val s = LazyList.from(1).map(_ * 2)`.

**10. Case classes, pattern matching and Option**

```scala
case class Student(name: String, marks: Int)
val sts = List(Student("Alice", 85), Student("Bob", 72), Student("Charlie", 91))

println(sts.filter(_.marks > 80).map(_.name))
println(sts.sortBy(-_.marks).map(_.name))
println(sts.groupBy(_.marks >= 80).map { case (k, v) => k -> v.map(_.name) })

val day = 3
val nm = day match { case 1 => "Mon"; case 2 => "Tue"; case 3 => "Wed"; case _ => "Other" }
println(nm)

val opt: Option[Int] = Some(5)
println(opt.map(_ + 1) + " " + None.getOrElse(0))
```

```text
List(Alice, Charlie)
List(Charlie, Alice, Bob)
Map(true -> List(Alice, Charlie), false -> List(Bob))
Wed
Some(6) 0
```

| Feature | Why it matters for Spark |
| --- | --- |
| **Case class** | Auto-generates `equals`, `toString`, `copy`; used as the schema for Datasets/DataFrames |
| **Pattern matching** | Used heavily in `map { case (k, v) => … }` on pair RDDs |
| **`Option`** | Safe handling of missing values (`Some` / `None`) instead of `null` |

**11. Ranges, for-comprehensions and recursion**

```scala
println((1 to 5).toList + " " + (1 until 5).toList + " " + (1 to 10 by 3).toList)
val res = for (i <- 1 to 3; j <- 1 to 2) yield i * j
println(res)
def fact(n: Int): Int = if (n <= 1) 1 else n * fact(n - 1)
println(fact(5))
```

```text
List(1, 2, 3, 4, 5) List(1, 2, 3, 4) List(1, 4, 7, 10)
Vector(1, 2, 2, 4, 3, 6)
120
```

### Common errors

| Error | Cause | Fix |
| --- | --- | --- |
| *reassignment to val* | Changed a `val` | Use `var`, or better, create a new value |
| `type mismatch; found Int, required String` | Wrong type | Convert (`.toString`) or fix the type |
| `List.updated` expecting a change in place | Immutable list | Capture the returned new list |
| `Stream` deprecation warning (2.13) | Old API | Use `LazyList` |
| Integer division surprise | `10 / 3` is `3` | Use `10 / 3.0` |

### Viva pointers for P2

*Features of Scala; val vs var; type inference; immutable vs mutable collections; higher-order functions; lazy streams; why Spark uses Scala.*

---

## P3 — RDDs: creation, transformations, actions, lazy evaluation, persistence, shared variables (Unit III)

### Aim

To create RDDs, apply transformations and actions, work with key-value (pair) RDDs and joins, observe **lazy evaluation and lineage**, use **persistence**, and use **shared variables** (broadcast variables and accumulators).

### Theory

> **An RDD (Resilient Distributed Dataset) is an immutable, partitioned collection of records spread across the cluster that can be processed in parallel and rebuilt after failure using its lineage.**

| Word | Meaning |
| --- | --- |
| **Resilient** | Fault tolerant: lost partitions are **recomputed** from lineage |
| **Distributed** | Split into **partitions** across machines |
| **Dataset** | A collection of records |

**Features of RDDs:** in-memory, lazy, immutable, partitioned, fault tolerant, typed (in Scala), persistable.

**Transformations vs actions**

| | **Transformation** | **Action** |
| --- | --- | --- |
| Result | A **new RDD** | A **value** to the driver (or written to storage) |
| Execution | **Lazy** (only recorded) | **Triggers** the job |
| Examples | `map`, `filter`, `flatMap`, `reduceByKey`, `join` | `collect`, `count`, `take`, `reduce`, `saveAsTextFile` |

**Narrow vs wide transformations**

| | Narrow | Wide |
| --- | --- | --- |
| Data movement | Each output partition depends on **one** input partition | Needs data from **many** partitions → **shuffle** |
| Examples | `map`, `filter`, `flatMap` | `reduceByKey`, `groupByKey`, `join`, `distinct` |
| Cost | Cheap | Expensive (network + disk) |

### Steps and code

Start `spark-shell` (the `sc` SparkContext exists already).

**1. Creating RDDs**

```scala
val nums = sc.parallelize(1 to 10, 3)           // from a collection, with 3 partitions
println(nums.getNumPartitions)
println(nums.collect().toList)

val lines = sc.textFile("C:/sparklabs/words.txt")   // from a file
println(lines.count())
println(lines.first())
println(lines.take(2).toList)
```

```text
3
List(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)
4
spark is fast
List(spark is fast, spark is in memory)
```

**2. Basic transformations**

```scala
println(nums.map(_ * 2).collect().toList)
println(nums.filter(_ % 2 == 0).collect().toList)

val words = lines.flatMap(_.split(" "))     // one line → many words
println(words.count())
println(words.distinct().count())
```

```text
List(2, 4, 6, 8, 10, 12, 14, 16, 18, 20)
List(2, 4, 6, 8, 10)
16
12
```

**`map` vs `flatMap`:** `map` gives **exactly one** output per input (so a line stays one element); `flatMap` can give **zero or many** and flattens the result (a line becomes its words).

**3. Word count: the classic**

```scala
val counts = words.map(w => (w, 1)).reduceByKey(_ + _)
println(counts.collect().sortBy(x => (-x._2, x._1)).toList)
```

```text
List((spark,3), (is,2), (mapreduce,2), (beats,1), (fast,1), (for,1), (hadoop,1), (in,1), (iterative,1), (jobs,1), (memory,1), (uses,1))
```

**4. Actions**

```scala
println(nums.reduce(_ + _) + " " + nums.sum() + " " + nums.max() + " " + nums.min() + " " + nums.mean())
println(nums.fold(0)(_ + _))
println(nums.aggregate((0, 0))((acc, v) => (acc._1 + v, acc._2 + 1), (a, b) => (a._1 + b._1, a._2 + b._2)))
println(nums.takeOrdered(3).toList + " " + nums.top(3).toList)
println(words.countByValue().toList.sortBy(x => (-x._2, x._1)).take(3))
```

```text
55 55.0 10 1 5.5
55
(55,10)
List(1, 2, 3) List(10, 9, 8)
List((spark,3), (is,2), (mapreduce,2))
```

`aggregate` returned `(sum, count) = (55, 10)`, so the average is 5.5: it computes **two** results in one pass.

**5. Set operations**

```scala
val a = sc.parallelize(List(1, 2, 3, 4)); val b = sc.parallelize(List(3, 4, 5, 6))
println(a.union(b).collect().toList.sorted)       // keeps duplicates
println(a.intersection(b).collect().toList.sorted)
println(a.subtract(b).collect().toList.sorted)
println(a.cartesian(sc.parallelize(List("x", "y"))).count())
```

```text
List(1, 2, 3, 3, 4, 4, 5, 6)
List(3, 4)
List(1, 2)
8
```

**6. Pair RDDs and joins**

```scala
val marks = sc.parallelize(List(("CSE", 85), ("ECE", 72), ("CSE", 91), ("IT", 65), ("ECE", 88)))

println(marks.reduceByKey(_ + _).collect().toList.sortBy(_._1))
println(marks.groupByKey().mapValues(_.toList.sorted).collect().toList.sortBy(_._1))
println(marks.mapValues(v => (v, 1)).reduceByKey((x, y) => (x._1 + y._1, x._2 + y._2))
            .mapValues(t => t._1.toDouble / t._2).collect().toList.sortBy(_._1))
println(marks.countByKey().toList.sortBy(_._1))
println(marks.lookup("CSE").toList)
```

```text
List((CSE,176), (ECE,160), (IT,65))
List((CSE,List(85, 91)), (ECE,List(72, 88)), (IT,List(65)))
List((CSE,88.0), (ECE,80.0), (IT,65.0))
List((CSE,2), (ECE,2), (IT,1))
List(85, 91)
```

```scala
val emp  = sc.parallelize(List((1, "Alice"), (2, "Bob"), (3, "Charlie")))
val dept = sc.parallelize(List((1, "CSE"), (2, "ECE"), (4, "IT")))

println(emp.join(dept).collect().toList.sortBy(_._1))
println(emp.leftOuterJoin(dept).collect().toList.sortBy(_._1))
println(emp.fullOuterJoin(dept).collect().toList.sortBy(_._1))
```

```text
List((1,(Alice,CSE)), (2,(Bob,ECE)))
List((1,(Alice,Some(CSE))), (2,(Bob,Some(ECE))), (3,(Charlie,None)))
List((1,(Some(Alice),Some(CSE))), (2,(Some(Bob),Some(ECE))), (3,(Charlie,None)), (4,(None,Some(IT))))
```

> **`reduceByKey` vs `groupByKey`:** `reduceByKey` **combines values on each partition first** (map-side combine) and then shuffles the small partial results. `groupByKey` shuffles **all** values. So `reduceByKey` is the better choice for aggregations: far less network traffic.

**7. Partitions**

```scala
println(nums.glom().collect().map(_.toList).toList)
println(nums.mapPartitionsWithIndex((i, it) => Iterator((i, it.size))).collect().toList)
println(sc.parallelize(List(1, 2, 3), 3).repartition(2).getNumPartitions + " " + nums.coalesce(1).getNumPartitions)
```

```text
List(List(1, 2, 3), List(4, 5, 6), List(7, 8, 9, 10))
List((0,3), (1,3), (2,4))
2 1
```

`repartition` can increase or decrease partitions (full shuffle); `coalesce` only **decreases** them (avoids a shuffle).

**8. Lazy evaluation and lineage**

```scala
val rdd = sc.textFile("C:/sparklabs/words.txt")
            .flatMap(_.split(" ")).map((_, 1)).reduceByKey(_ + _).filter(_._2 > 1)
// NOTHING has run yet: only a plan was recorded

println(rdd.toDebugString)
```

⚠️ Typical lineage (indices vary):

```text
(2) MapPartitionsRDD[5] at filter at <console>:23 []
 |  ShuffledRDD[4] at reduceByKey at <console>:23 []
 +-(2) MapPartitionsRDD[3] at map at <console>:23 []
    |  MapPartitionsRDD[2] at flatMap at <console>:23 []
    |  C:/sparklabs/words.txt MapPartitionsRDD[1] at textFile at <console>:23 []
    |  C:/sparklabs/words.txt HadoopRDD[0] at textFile at <console>:23 []
```

Read it **bottom to top**: read file → flatMap → map → *(shuffle)* → reduceByKey → filter. The `+-` marks a **stage boundary** (the shuffle). The lineage is the recipe Spark replays if a partition is lost: this is **fault tolerance**.

```scala
println(rdd.collect().toList)       // the ACTION: now the whole job runs
```

**9. Persistence (caching)**

```scala
import org.apache.spark.storage.StorageLevel

val c1 = nums.map(_ * 2).cache()          // cache = persist(MEMORY_ONLY)
c1.count()                                // first action computes AND stores it
println(c1.getStorageLevel)

val c2 = nums.map(_ + 1).persist(StorageLevel.MEMORY_AND_DISK)
c2.count()
println(c2.getStorageLevel.description)

c1.unpersist()
println(c1.getStorageLevel)
```

```text
StorageLevel(memory, deserialized, 1 replicas)
Disk Memory Deserialized 1x Replicated
StorageLevel(1 replicas)
```

| Storage level | Behaviour |
| --- | --- |
| `MEMORY_ONLY` (`cache()`) | Keep in RAM; recompute partitions that do not fit |
| `MEMORY_AND_DISK` | Spill partitions that do not fit to disk |
| `DISK_ONLY` | Store on disk only |
| `*_SER` | Store serialized (smaller, slower to read) |
| `*_2` | Replicate on 2 nodes |

**When to cache:** an RDD that is **reused several times** (iterative algorithms, interactive analysis). Caching a one-use RDD only wastes memory. Check the **Storage** tab of the Spark UI.

**10. Shared variables**

Spark normally sends a **copy** of every variable to each task. Two special shared variables solve two common needs:

| | **Broadcast variable** | **Accumulator** |
| --- | --- | --- |
| Direction | Driver → executors | Executors → driver |
| Access | **Read-only** on executors | **Write-only** (add) on executors; read on the driver |
| Purpose | Share a **large lookup table** once per executor | Counters and sums (for example count bad records) |

*Broadcast:*

```scala
val lookup = sc.broadcast(Map("CSE" -> "Computer Science", "ECE" -> "Electronics", "IT" -> "Information Tech"))
println(marks.map { case (d, m) => (lookup.value.getOrElse(d, "?"), m) }.collect().toList)
println(lookup.value.size)
```

```text
List((Computer Science,85), (Electronics,72), (Computer Science,91), (Information Tech,65), (Electronics,88))
3
```

*Accumulator:*

```scala
val bad = sc.longAccumulator("bad")
sc.parallelize(List("1", "2", "x", "4", "y")).foreach(s => if (!s.forall(_.isDigit)) bad.add(1))
println(bad.value)

val total = sc.longAccumulator("sum")
nums.foreach(n => total.add(n))
println(total.value)
```

```text
2
55
```

Two bad records were counted across all partitions. **Caution:** update accumulators only inside **actions** (like `foreach`). Inside transformations they can be updated more than once if a task is retried, giving wrong counts.

**11. Saving output**

```scala
nums.saveAsTextFile("C:/sparklabs/out_nums")
println(sc.textFile("C:/sparklabs/out_nums").count())
```

```text
10
```

The output is a **folder** with one `part-0000x` file per partition (plus `_SUCCESS`). The folder must **not already exist**, or Spark raises `FileAlreadyExistsException`.

### Common errors

| Error | Cause | Fix |
| --- | --- | --- |
| `FileAlreadyExistsException` on save | Output folder exists | Delete or change the path |
| `Input path does not exist` | Wrong path | Use `C:/sparklabs/...` with forward slashes |
| Nothing happens after `map` | **Lazy** evaluation | Call an action |
| `Task not serializable` | Closure uses a non-serializable object | Avoid capturing such objects; use broadcast |
| `collect()` out-of-memory | Pulled a huge RDD to the driver | Use `take(n)` or save to files |
| Slow job with `groupByKey` | Shuffles all values | Use `reduceByKey` |
| Accumulator over-counts | Updated inside a transformation | Update in actions only |

### Viva pointers for P3

*What is an RDD and why resilient; transformations vs actions; lazy evaluation; lineage and fault tolerance; narrow vs wide; `reduceByKey` vs `groupByKey`; `cache` vs `persist`; broadcast vs accumulator; partitions and shuffle.*

---

## P4 — GraphX: graph processing in Spark (Unit III) ✅ *official lab item*

### Aim

To build a property graph in Spark GraphX, explore it (vertices, edges, triplets, degrees), transform it, aggregate messages, and run graph algorithms: **PageRank, Connected Components, Triangle Count** and a **Pregel** shortest-path computation.

### Theory

> **GraphX is Spark's API for graphs and graph-parallel computation. It extends the RDD with a directed multigraph whose vertices and edges carry user-defined properties.**

```text
 Graph  =  VertexRDD[VD]   (id, vertex property)
        +  EdgeRDD[ED]     (srcId, dstId, edge property)
        +  EdgeTriplet     (source vertex + edge + destination vertex, joined)
```

| Term | Meaning |
| --- | --- |
| **Vertex** | A node, identified by a unique `VertexId` (a `Long`) with a property (name, age…) |
| **Edge** | A directed link `srcId → dstId` with a property (relationship type) |
| **Triplet** | An edge together with the properties of **both** its end vertices |
| **Property graph** | A graph in which vertices and edges carry data |
| **Multigraph** | Several edges may exist between the same two vertices |

**Why GraphX?** Social networks, recommendation, fraud rings, web ranking, road networks: all naturally graphs. GraphX gives **graph and table views of the same data**, fast in-memory iteration, and built-in algorithms. *Note:* GraphX has a **Scala/Java API only** (no PySpark API; Python users use the separate GraphFrames package).

### The graph used in this practical

```text
   (1) Alice ───friend───▶ (2) Bob ───follows───▶ (3) Charlie ───colleague───▶ (4) David
        ▲                                              │
        └──────────────────friend──────────────────────┘

   (5) Eve ───friend───▶ (6) Frank              (a separate component)
```

### Steps and code (all outputs verified)

**1. Build the graph**

```scala
import org.apache.spark.graphx._
import org.apache.spark.rdd.RDD

val users: RDD[(VertexId, (String, Int))] = sc.parallelize(Seq(
  (1L, ("Alice", 28)), (2L, ("Bob", 27)), (3L, ("Charlie", 65)),
  (4L, ("David", 42)), (5L, ("Eve", 35)), (6L, ("Frank", 50))))

val rels: RDD[Edge[String]] = sc.parallelize(Seq(
  Edge(1L, 2L, "friend"), Edge(2L, 3L, "follows"), Edge(3L, 1L, "friend"),
  Edge(3L, 4L, "colleague"), Edge(5L, 6L, "friend")))

val graph = Graph(users, rels)
println(graph.numVertices + " " + graph.numEdges)
```

```text
6 5
```

**2. Inspect: vertices, edges, triplets**

```scala
graph.vertices.collect().sortBy(_._1).foreach(println)
graph.edges.collect().sortBy(e => (e.srcId, e.dstId)).foreach(println)
graph.triplets.collect().sortBy(t => (t.srcId, t.dstId))
     .foreach(t => println(t.srcAttr._1 + " --" + t.attr + "--> " + t.dstAttr._1))
```

```text
(1,(Alice,28))
(2,(Bob,27))
(3,(Charlie,65))
(4,(David,42))
(5,(Eve,35))
(6,(Frank,50))

Edge(1,2,friend)
Edge(2,3,follows)
Edge(3,1,friend)
Edge(3,4,colleague)
Edge(5,6,friend)

Alice --friend--> Bob
Bob --follows--> Charlie
Charlie --friend--> Alice
Charlie --colleague--> David
Eve --friend--> Frank
```

**3. Degrees** (number of connections)

```scala
graph.inDegrees.collect().sortBy(_._1).foreach(println)
graph.outDegrees.collect().sortBy(_._1).foreach(println)
graph.degrees.collect().sortBy(_._1).foreach(println)
```

```text
in : (1,1) (2,1) (3,1) (4,1) (6,1)
out: (1,1) (2,1) (3,2) (5,1)
all: (1,2) (2,2) (3,3) (4,1) (5,1) (6,1)
```

Charlie (3) has the highest total degree: one incoming, two outgoing. Vertices with **no** incoming (or outgoing) edges are **missing** from `inDegrees` (or `outDegrees`): David has no out-edges, so he is absent from `outDegrees`.

**4. Structural operations**

```scala
// subgraph: keep vertices older than 30 (edges touching removed vertices disappear)
val older = graph.subgraph(vpred = (id, attr) => attr._2 > 30)
println(older.numVertices + " " + older.numEdges)

// keep only "friend" edges
val friends = graph.subgraph(epred = e => e.attr == "friend")
println(friends.edges.collect().sortBy(_.srcId).toList)

// reverse every edge direction
graph.reverse.edges.collect().sortBy(e => (e.srcId, e.dstId)).foreach(println)
```

```text
4 2
List(Edge(1,2,friend), Edge(3,1,friend), Edge(5,6,friend))
Edge(1,3,friend)
Edge(2,1,friend)
Edge(3,2,follows)
Edge(4,3,colleague)
Edge(6,5,friend)
```

Only Charlie, David, Eve and Frank are older than 30, and only two edges (Charlie→David, Eve→Frank) connect two such vertices.

**5. Property transformations** (the structure stays, the data changes)

```scala
graph.mapVertices((id, attr) => attr._1.toUpperCase).vertices.collect().sortBy(_._1).foreach(println)
graph.mapEdges(e => e.attr.length).edges.collect().sortBy(e => (e.srcId, e.dstId)).foreach(println)
```

```text
(1,ALICE) (2,BOB) (3,CHARLIE) (4,DAVID) (5,EVE) (6,FRANK)
Edge(1,2,6) Edge(2,3,7) Edge(3,1,6) Edge(3,4,9) Edge(5,6,6)
```

**6. Join operations**

```scala
// outerJoinVertices: attach each vertex's degree (0 if it has none)
val withDeg = graph.outerJoinVertices(graph.degrees) { (id, attr, deg) => (attr._1, deg.getOrElse(0)) }
withDeg.vertices.collect().sortBy(_._1).foreach(println)

// joinVertices: add 1 year to the ages of vertices 1 and 2 only
val inc = sc.parallelize(Seq((1L, 1), (2L, 1)))
graph.joinVertices(inc) { (id, attr, i) => (attr._1, attr._2 + i) }
     .vertices.filter(_._1 <= 2).collect().sortBy(_._1).foreach(println)
```

```text
(1,(Alice,2)) (2,(Bob,2)) (3,(Charlie,3)) (4,(David,1)) (5,(Eve,1)) (6,(Frank,1))
(1,(Alice,29))
(2,(Bob,28))
```

| `joinVertices` | `outerJoinVertices` |
| --- | --- |
| Updates only vertices that **match**; others keep their old value | Applies to **all** vertices; unmatched get `None` |
| Result type must equal the old type | Result type can **change** |

**7. `aggregateMessages`: the core graph primitive**

Each edge **sends a message** to a vertex; messages arriving at a vertex are **merged**.

```scala
// number of incoming edges per vertex
val inCount = graph.aggregateMessages[Int](ctx => ctx.sendToDst(1), _ + _)
inCount.collect().sortBy(_._1).foreach(println)

// the oldest person who points to each vertex
val oldest = graph.aggregateMessages[(String, Int)](
  ctx => ctx.sendToDst(ctx.srcAttr),
  (a, b) => if (a._2 > b._2) a else b)
oldest.collect().sortBy(_._1).foreach(println)
```

```text
(1,1) (2,1) (3,1) (4,1) (6,1)
(1,(Charlie,65)) (2,(Alice,28)) (3,(Bob,27)) (4,(Charlie,65)) (6,(Eve,35))
```

**8. Neighbourhood**

```scala
graph.collectNeighborIds(EdgeDirection.Out).collect().sortBy(_._1)
     .foreach { case (id, arr) => println(id + " -> " + arr.toList.sorted) }
```

```text
1 -> List(2)
2 -> List(3)
3 -> List(1, 4)
4 -> List()
5 -> List(6)
6 -> List()
```

**9. PageRank: who is most important?**

> **PageRank** scores a vertex by how many vertices point to it **and how important those vertices are** (Google's original web-ranking idea). In GraphX: `PR(v) = 0.15 + 0.85 × Σ PR(u) / outDegree(u)` over the vertices `u` that point to `v`.

```scala
val ranks = graph.pageRank(0.0001).vertices
ranks.join(users).map { case (id, (rank, (name, _))) => (name, rank) }
     .collect().sortBy(-_._2).foreach { case (n, r) => println(f"$n%-8s $r%.4f") }
```

```text
Charlie  1.4940
Bob      1.2841
David    1.0374
Alice    1.0374
Frank    0.7447
Eve      0.4025
```

**Reading it:** Charlie ranks highest (Bob points to him), then Bob (Alice points to him, and Charlie's rank flows back to Alice). **Eve** has no incoming edges, so she receives only the baseline `0.15 / 0.85` share and ranks lowest. `0.0001` is the **tolerance**: iteration stops when scores change by less than this.

**10. Connected Components and Triangle Count**

```scala
graph.connectedComponents().vertices.collect().sortBy(_._1).foreach(println)
graph.triangleCount().vertices.collect().sortBy(_._1).foreach(println)
```

```text
(1,1) (2,1) (3,1) (4,1) (5,5) (6,5)
(1,1) (2,1) (3,1) (4,0) (5,0) (6,0)
```

- **Connected components:** every vertex is labelled with the **smallest vertex id in its component**. Alice, Bob, Charlie, David form component `1`; Eve and Frank form component `5`. So there are **two** components.
- **Triangle count:** Alice, Bob and Charlie each belong to **1 triangle** (1→2→3→1, treating edges as undirected); David, Eve and Frank belong to none.

**11. Pregel: iterative message-passing (shortest hops from Alice)**

```scala
val start = graph.mapVertices((id, _) => if (id == 1L) 0.0 else Double.PositiveInfinity)

val sssp = start.pregel(Double.PositiveInfinity)(
  (id, dist, newDist) => math.min(dist, newDist),               // vertex program
  t => if (t.srcAttr + 1 < t.dstAttr) Iterator((t.dstId, t.srcAttr + 1)) else Iterator.empty,   // send message
  (a, b) => math.min(a, b))                                     // merge messages

sssp.vertices.collect().sortBy(_._1).foreach(println)
```

```text
(1,0.0) (2,1.0) (3,2.0) (4,3.0) (5,Infinity) (6,Infinity)
```

Alice → Bob is 1 hop, → Charlie 2, → David 3. Eve and Frank are **unreachable** (infinite), because they are in another component.

### Common errors

| Error | Cause | Fix |
| --- | --- | --- |
| Vertex ids not `Long` | `VertexId` must be `Long` | Write `1L` |
| Vertex missing from `inDegrees` | It has no incoming edges | Use `outerJoinVertices` with `getOrElse(0)` |
| Edge refers to a missing vertex | Missing vertex gets a default attribute | Supply the default in `Graph(users, rels, defaultVertex)` |
| `Graph` not found | Import missing | `import org.apache.spark.graphx._` |
| PageRank never ends | Tolerance too small | Use `pageRank(0.0001)` or `staticPageRank(numIter)` |
| Wanting GraphX in Python | No PySpark API | Use GraphFrames |

### Viva pointers for P4

*GraphX architecture (VertexRDD, EdgeRDD, triplet); property graph; operators (structural, property, join, aggregate); `aggregateMessages`; PageRank idea; connected components; Pregel; GraphX vs GraphFrames.*

---

## P5 — Feature extraction and transformation (Unit III)

### Aim

To convert raw data (text, categories, numbers) into numerical **feature vectors** that machine-learning algorithms can use, using Spark ML's feature tools.

### Theory

| Term | Meaning | Examples |
| --- | --- | --- |
| **Feature extraction** | Creating numeric features **from raw data** | Text → TF-IDF vector |
| **Feature transformation** | **Changing** existing features | Scaling, encoding, bucketing |
| **Feature selection** | **Choosing** the useful features | Chi-square selector |

Algorithms need numbers in **one Vector column**. These tools build it.

### Steps and code (Scala `spark.ml`; outputs verified)

```scala
import org.apache.spark.ml.feature._
import org.apache.spark.ml.linalg.Vectors
import spark.implicits._

val docs = Seq((0, "Spark is fast and Spark is in memory"),
               (1, "Hadoop uses MapReduce on disk"),
               (2, "Spark and Kafka process streams")).toDF("id", "text")
```

**1. Tokenizer: text → words**

```scala
val tok  = new RegexTokenizer().setInputCol("text").setOutputCol("words").setPattern("\\W+").setToLowercase(true)
val tokd = tok.transform(docs)
tokd.select("id", "words").show(false)
```

```text
+---+---------------------------------------------+
|id |words                                        |
+---+---------------------------------------------+
|0  |[spark, is, fast, and, spark, is, in, memory]|
|1  |[hadoop, uses, mapreduce, on, disk]          |
|2  |[spark, and, kafka, process, streams]        |
+---+---------------------------------------------+
```

**2. StopWordsRemover: drop common words** (is, and, in, on…)

```scala
val sw = new StopWordsRemover().setInputCol("words").setOutputCol("clean")
val cleaned = sw.transform(tokd)
cleaned.select("id", "clean").show(false)
```

```text
+---+--------------------------------+
|id |clean                           |
+---+--------------------------------+
|0  |[spark, fast, spark, memory]    |
|1  |[hadoop, uses, mapreduce, disk] |
|2  |[spark, kafka, process, streams]|
+---+--------------------------------+
```

**3. CountVectorizer: words → term-frequency vectors**

```scala
val cv = new CountVectorizer().setInputCol("clean").setOutputCol("tf").fit(cleaned)
println(cv.vocabulary.toList)
val tf = cv.transform(cleaned)
tf.select("id", "tf").show(false)
```

```text
List(spark, fast, process, uses, hadoop, mapreduce, memory, kafka, disk, streams)

|0  |(10,[0,1,6],[2.0,1.0,1.0])      |
|1  |(10,[3,4,5,8],[1.0,1.0,1.0,1.0])|
|2  |(10,[0,2,7,9],[1.0,1.0,1.0,1.0])|
```

Document 0: `(10,[0,1,6],[2.0,1.0,1.0])` is a **sparse vector** of length 10: position 0 (`spark`) = 2, position 1 (`fast`) = 1, position 6 (`memory`) = 1.

**4. TF-IDF: weight words by how special they are**

> **TF-IDF = Term Frequency × Inverse Document Frequency.** A word that appears often in *one* document but in *few* documents overall is distinctive and gets a high weight; a word in *every* document gets a low weight.

Spark's formula: `IDF(t) = ln( (m + 1) / (df(t) + 1) )`, where `m` = number of documents and `df(t)` = number of documents containing `t`.

```scala
val idfm = new IDF().setInputCol("tf").setOutputCol("tfidf").fit(tf)
idfm.transform(tf).select("id", "tfidf").show(false)
```

```text
|0  |(10,[0,1,6],[0.5753641449035617,0.6931471805599453,0.6931471805599453])|
|1  |(10,[3,4,5,8],[0.6931471805599453,0.6931471805599453,0.6931471805599453,0.6931471805599453])|
|2  |(10,[0,2,7,9],[0.28768207245178085,0.6931471805599453,0.6931471805599453,0.6931471805599453])|
```

**Hand check.** `m = 3`.

```text
 "spark":   df = 2 (docs 0 and 2)   IDF = ln(4/3) = 0.2877
            doc 0: TF = 2  →  2 × 0.2877 = 0.5754   ✔      doc 2: TF = 1 → 0.2877   ✔
 "fast":    df = 1                  IDF = ln(4/2) = 0.6931
            doc 0: TF = 1  →  0.6931   ✔
```

**"spark" is common (2 of 3 documents), so it is down-weighted; "fast" and "hadoop" are rare and get higher weights.**

**5. N-grams and HashingTF**

```scala
new NGram().setN(2).setInputCol("clean").setOutputCol("bigrams").transform(cleaned).select("id", "bigrams").show(false)
new HashingTF().setInputCol("clean").setOutputCol("hashed").setNumFeatures(16).transform(cleaned).select("id", "hashed").show(false)
```

```text
|0  |[spark fast, fast spark, spark memory]       |
|1  |[hadoop uses, uses mapreduce, mapreduce disk]|
|2  |[spark kafka, kafka process, process streams]|

|0  |(16,[6,13,15],[2.0,1.0,1.0])     |
|1  |(16,[1,6,7,10],[1.0,1.0,1.0,1.0])|
|2  |(16,[6,7,8,14],[1.0,1.0,1.0,1.0])|
```

| `CountVectorizer` | `HashingTF` |
| --- | --- |
| Builds an explicit **vocabulary** | Uses a **hash function** (no vocabulary kept) |
| Exact, reversible (index → word) | Fast and memory-light, but **hash collisions** possible (see column 6 shared by different words) |

**6. StringIndexer and OneHotEncoder: categories → numbers**

```scala
val df = Seq(("CSE", 21, 85.0), ("ECE", 22, 72.0), ("CSE", 23, 91.0), ("IT", 22, 65.0)).toDF("dept", "age", "marks")

val si  = new StringIndexer().setInputCol("dept").setOutputCol("dept_idx").fit(df)
val idx = si.transform(df); idx.show(); println(si.labels.toList)

val ohe = new OneHotEncoder().setInputCols(Array("dept_idx")).setOutputCols(Array("dept_vec")).fit(idx)
val enc = ohe.transform(idx); enc.show(false)
```

```text
+----+---+-----+--------+
|dept|age|marks|dept_idx|
+----+---+-----+--------+
| CSE| 21| 85.0|     0.0|
| ECE| 22| 72.0|     1.0|
| CSE| 23| 91.0|     0.0|
|  IT| 22| 65.0|     2.0|
+----+---+-----+--------+
List(CSE, ECE, IT)

|CSE |21 |85.0 |0.0     |(2,[0],[1.0])|
|ECE |22 |72.0 |1.0     |(2,[1],[1.0])|
|CSE |23 |91.0 |0.0     |(2,[0],[1.0])|
|IT  |22 |65.0 |2.0     |(2,[],[])    |
```

`StringIndexer` gives the **most frequent** value the index 0. `OneHotEncoder` turns an index into a 0/1 vector so that no false order (IT = 2 is "bigger") is implied; by default the **last category is dropped** (all zeros) to avoid a redundant column.

**7. VectorAssembler: build the final `features` column**

```scala
val va   = new VectorAssembler().setInputCols(Array("age", "marks", "dept_vec")).setOutputCol("features")
val feat = va.transform(enc)
feat.select("features").show(false)
```

```text
|[21.0,85.0,1.0,0.0]|
|[22.0,72.0,0.0,1.0]|
|[23.0,91.0,1.0,0.0]|
|[22.0,65.0,0.0,0.0]|
```

**8. Scaling: StandardScaler and MinMaxScaler**

```scala
val ss = new StandardScaler().setInputCol("features").setOutputCol("scaled").setWithMean(true).setWithStd(true).fit(feat)
ss.transform(feat).select("scaled").show(false)
val mm = new MinMaxScaler().setInputCol("features").setOutputCol("mm").fit(feat)
mm.transform(feat).select("mm").show(false)
```

```text
scaled (first rows): [-1.2247,0.5686,0.8660,-0.5]   [0.0,-0.5265,-0.8660,1.5] ...
mm:                  [0.0,0.7692,1.0,0.0]           [0.5,0.2692,0.0,1.0] ...
```

```text
 StandardScaler:  z  = (x − mean) / std            → mean 0, standard deviation 1
 MinMaxScaler:    x' = (x − min) / (max − min)     → values in [0, 1]
```

*Hand check (age column 21, 22, 23, 22; mean 22; sample std 0.8165):* z for age 21 = (21 − 22) / 0.8165 = **−1.2247** ✔.

**9. Bucketizer, Binarizer, Normalizer, PCA**

```scala
new Bucketizer().setInputCol("marks").setOutputCol("band").setSplits(Array(0.0, 70.0, 85.0, 100.1))
  .transform(df).select("marks", "band").show()
new Binarizer().setInputCol("marks").setOutputCol("pass").setThreshold(70.0)
  .transform(df).select("marks", "pass").show()
```

```text
+-----+----+        +-----+----+
|marks|band|        |marks|pass|
+-----+----+        +-----+----+
| 85.0| 2.0|        | 85.0| 1.0|
| 72.0| 1.0|        | 72.0| 1.0|
| 91.0| 2.0|        | 91.0| 1.0|
| 65.0| 0.0|        | 65.0| 0.0|
+-----+----+        +-----+----+
```

Buckets: `[0,70) → 0`, `[70,85) → 1`, `[85,100.1) → 2`.

```scala
val pca = new PCA().setInputCol("features").setOutputCol("pca").setK(2).fit(feat)
pca.transform(feat).select("pca").show(false)
println(pca.explainedVariance)
```

```text
|[-85.2362,20.0350] |
|[-72.2061,21.3371] |
|[-91.2570,21.9693] |
|[-65.2298,21.2829] |
[0.9937342498081377,0.004612054882588201]
```

**PCA** (Principal Component Analysis) reduces 4 features to 2. The explained variance shows the **first component keeps 99.4%** of the information (dominated by the large-valued `marks` column, which is why we normally **scale first**).

| Tool | Type | Purpose |
| --- | --- | --- |
| `Tokenizer`, `RegexTokenizer` | extraction | Text → words |
| `StopWordsRemover` | transformation | Remove common words |
| `CountVectorizer`, `HashingTF`, `IDF` | extraction | Words → TF / TF-IDF |
| `NGram` | extraction | Word sequences |
| `StringIndexer`, `OneHotEncoder` | transformation | Categories → numbers |
| `VectorAssembler` | transformation | Combine into one vector |
| `StandardScaler`, `MinMaxScaler`, `Normalizer` | transformation | Scale |
| `Bucketizer`, `Binarizer` | transformation | Discretize |
| `PCA` | dimensionality reduction | Fewer features |

### Common errors

| Error | Cause | Fix |
| --- | --- | --- |
| `Unseen label` | New category in `StringIndexer` | `setHandleInvalid("keep")` |
| Nulls in `VectorAssembler` | Missing values | `Imputer` / drop |
| Dominated by large features | No scaling before distance-based methods | Scale first |
| Huge vocabulary | No limit | `setVocabSize`, `setMinDF` |

### Viva pointers for P5

*Extraction vs transformation vs selection; TF-IDF formula and meaning; one-hot encoding; why scale; PCA; sparse vs dense vectors.*

---

## P6 — Spark SQL (Unit IV) ✅ *official lab item*

### Aim

To process structured data with Spark SQL: create DataFrames (including from RDDs), register views, and run SQL queries with filtering, sorting, wildcards, aggregation, grouping and joins.

### Theory

> **Spark SQL is Spark's module for structured data. It lets us query data with SQL *or* the DataFrame API, using the same optimizer.**

| Importance and features | Explanation |
| --- | --- |
| **Structured data** | Works with CSV, JSON, Parquet, Hive, JDBC |
| **Two interfaces** | SQL strings and DataFrame methods give the **same plan** |
| **Catalyst optimizer** | Rewrites queries (predicate pushdown, column pruning, join reordering) |
| **Tungsten engine** | Efficient memory and code generation |
| **Integration** | One program mixes SQL, RDDs, Streaming and ML |

| | RDD | **DataFrame** | Dataset |
| --- | --- | --- | --- |
| Schema | None | **Named, typed columns** | Typed objects |
| Optimizer | None | **Catalyst** | Catalyst |
| Languages | All | All | Scala/Java |
| Use | Low-level control | **Structured/SQL work (default)** | Compile-time type safety |

**Concepts:** `SparkSession` (the entry point, `spark`), `DataFrame` (rows with a schema), **temporary view** (a session-scoped name for a DataFrame so SQL can use it), `spark.sql()` (returns a DataFrame, lazy).

### Steps and code (Scala; all outputs verified)

**1. Methods to convert an RDD to a DataFrame**

```scala
import spark.implicits._
import org.apache.spark.sql.Row
import org.apache.spark.sql.types._

val rdd = sc.parallelize(Seq((101, "Alice", "CSE", 21, 85), (102, "Bob", "ECE", 22, 72), (103, "Charlie", "CSE", 21, 91),
                             (104, "David", "IT", 23, 65), (105, "Emma", "ECE", 22, 88)))

// Method 1: toDF with column names
val df1 = rdd.toDF("student_id", "name", "department", "age", "marks")

// Method 2: case class (schema by reflection)
case class Student(student_id: Int, name: String, department: String, age: Int, marks: Int)
val df2 = rdd.map(t => Student(t._1, t._2, t._3, t._4, t._5)).toDF()

// Method 3: Row + StructType (programmatic schema)
val schema = StructType(Array(
  StructField("student_id", IntegerType, false), StructField("name", StringType, true),
  StructField("department", StringType, true), StructField("age", IntegerType, true), StructField("marks", IntegerType, true)))
val df3 = spark.createDataFrame(rdd.map(t => Row(t._1, t._2, t._3, t._4, t._5)), schema)

df1.show()
df1.printSchema()
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
 |-- student_id: integer (nullable = false)
 |-- name: string (nullable = true)
 |-- department: string (nullable = true)
 |-- age: integer (nullable = false)
 |-- marks: integer (nullable = false)
```

**Reading from a file** (CSV with header and inferred schema):

```scala
val csvDF = spark.read.option("header", "true").option("inferSchema", "true").csv("C:/sparklabs/students.csv")
```

**2. Register a view and query with SQL**

```scala
df1.createOrReplaceTempView("students")
spark.sql("SELECT name, marks FROM students WHERE marks > 80").show()
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

**3. Filtering, `ORDER BY`, `LIMIT`, `DISTINCT`**

```scala
spark.sql("SELECT * FROM students WHERE department = 'CSE' AND marks > 85").show()
spark.sql("SELECT name, marks FROM students ORDER BY marks DESC LIMIT 3").show()
spark.sql("SELECT DISTINCT department FROM students ORDER BY department").show()
spark.sql("SELECT name, department FROM students WHERE department IN ('CSE','IT')").show()
spark.sql("SELECT name, marks FROM students WHERE marks BETWEEN 70 AND 88").show()
```

```text
|103|Charlie|CSE| 21| 91|                      (CSE and marks > 85)

|Charlie| 91|  |Emma| 88|  |Alice| 85|          (top 3)

|CSE| |ECE| |IT|                                (distinct departments)

|Alice|CSE| |Charlie|CSE| |David|IT|            (IN)

|Alice| 85| |Bob| 72| |Emma| 88|                (BETWEEN is inclusive)
```

**4. SQL wildcards with `LIKE`**

`%` = any number of characters; `_` = exactly one character.

```scala
spark.sql("SELECT name FROM students WHERE name LIKE 'A%'").show()        // starts with A
spark.sql("SELECT name FROM students WHERE name LIKE '%e'").show()        // ends with e
spark.sql("SELECT name FROM students WHERE name LIKE '_ob'").show()       // 3 letters ending "ob"
spark.sql("SELECT name FROM students WHERE name LIKE '%a%'").show()       // contains lowercase a
```

```text
'A%'  → Alice
'%e'  → Alice, Charlie
'_ob' → Bob
'%a%' → Charlie, David, Emma         ← Alice is MISSING: LIKE is case-sensitive
```

Use `LOWER(name) LIKE '%a%'` (or `ILIKE`) for a case-insensitive match.

**5. Aggregate functions**

```scala
spark.sql("SELECT COUNT(*) AS n, SUM(marks) AS total, AVG(marks) AS average, MIN(marks) AS lowest, MAX(marks) AS highest FROM students").show()
```

```text
+---+-----+-------+------+-------+
|  n|total|average|lowest|highest|
+---+-----+-------+------+-------+
|  5|  401|   80.2|    65|     91|
+---+-----+-------+------+-------+
```

**6. `GROUP BY` and `HAVING`**

```scala
spark.sql("""SELECT department, COUNT(*) AS n, ROUND(AVG(marks),1) AS avg_marks, MAX(marks) AS top
             FROM students GROUP BY department ORDER BY department""").show()

spark.sql("""SELECT department, AVG(marks) AS avg_marks FROM students
             GROUP BY department HAVING AVG(marks) > 70 ORDER BY department""").show()
```

```text
+----------+---+---------+---+
|department|  n|avg_marks|top|
+----------+---+---------+---+
|       CSE|  2|     88.0| 91|
|       ECE|  2|     80.0| 88|
|        IT|  1|     65.0| 65|
+----------+---+---------+---+

+----------+---------+
|department|avg_marks|
+----------+---------+
|       CSE|     88.0|
|       ECE|     80.0|
+----------+---------+
```

> **`WHERE` filters rows *before* grouping; `HAVING` filters groups *after* aggregation** (and may use aggregate functions). Logical order: `FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY → LIMIT`.

**7. Joins**

Create a second table; we add a student whose department is missing from it (MECH), and a department with no students (EEE):

```scala
val students = Seq((101,"Alice","CSE",85),(102,"Bob","ECE",72),(103,"Charlie","CSE",91),
                   (104,"David","IT",65),(105,"Emma","ECE",88),(106,"Frank","MECH",59))
                 .toDF("student_id", "name", "department", "marks")
val depts = Seq(("CSE","Dr. Sharma"),("ECE","Dr. Verma"),("IT","Dr. Singh"),("EEE","Dr. Kaur")).toDF("department", "dept_head")
students.createOrReplaceTempView("s"); depts.createOrReplaceTempView("d")

spark.sql("SELECT s.name, s.department, d.dept_head FROM s INNER JOIN d ON s.department = d.department ORDER BY s.student_id").show()
spark.sql("SELECT s.name, s.department, d.dept_head FROM s LEFT JOIN d ON s.department = d.department ORDER BY s.student_id").show()
spark.sql("SELECT s.name FROM s LEFT ANTI JOIN d ON s.department = d.department").show()
```

```text
INNER JOIN:                         LEFT JOIN:                          LEFT ANTI JOIN:
|  Alice|CSE|Dr. Sharma|             |  Alice|CSE |Dr. Sharma|          +-----+
|    Bob|ECE| Dr. Verma|             |    Bob|ECE | Dr. Verma|          | name|
|Charlie|CSE|Dr. Sharma|             |Charlie|CSE |Dr. Sharma|          +-----+
|  David| IT| Dr. Singh|             |  David| IT | Dr. Singh|          |Frank|
|   Emma|ECE| Dr. Verma|             |   Emma|ECE | Dr. Verma|          +-----+
(Frank dropped)                      |  Frank|MECH|      NULL|
                                     (Frank kept, dept_head NULL)
```

| Join | Keeps |
| --- | --- |
| `INNER` | Only matching rows |
| `LEFT` | All left rows (NULL when no match) |
| `RIGHT` | All right rows |
| `FULL OUTER` | All rows of both |
| `LEFT SEMI` | Left rows that **have** a match (left columns only) |
| `LEFT ANTI` | Left rows with **no** match |
| `CROSS` | Every combination (m × n rows) |

**Join + group (department report):**

```scala
spark.sql("""SELECT d.dept_head, COUNT(*) AS students, ROUND(AVG(s.marks),1) AS avg_marks
             FROM s JOIN d ON s.department = d.department GROUP BY d.dept_head ORDER BY d.dept_head""").show()
```

```text
+----------+--------+---------+
| dept_head|students|avg_marks|
+----------+--------+---------+
|Dr. Sharma|       2|     88.0|
| Dr. Singh|       1|     65.0|
| Dr. Verma|       2|     80.0|
+----------+--------+---------+
```

**8. The DataFrame API gives the same result**

```scala
import org.apache.spark.sql.functions._
df1.filter(col("marks") > 80).select("name", "marks").show()
df1.groupBy("department").agg(count("*").alias("n"), round(avg("marks"), 1).alias("avg_marks")).orderBy("department").show()
students.join(depts, Seq("department"), "left").select("name", "department", "dept_head").show()
```

### Common errors

| Error | Cause | Fix |
| --- | --- | --- |
| `TABLE_OR_VIEW_NOT_FOUND` | View not created / misspelled | `createOrReplaceTempView`; `spark.catalog.listTables()` |
| `AMBIGUOUS_REFERENCE` | Same column in both tables | Qualify as `s.department` |
| `MISSING_AGGREGATION` | Non-grouped column in `SELECT` | Add to `GROUP BY` or aggregate it |
| `LIKE '%a%'` misses "Alice" | LIKE is case-sensitive | `LOWER(...)` / `ILIKE` |
| `WHERE marks = NULL` returns nothing | NULL comparison | `IS NULL` |
| Nothing displayed | No action | Add `.show()` |

### Viva pointers for P6

*Features of Spark SQL; Catalyst; RDD vs DataFrame vs Dataset; RDD → DataFrame methods; temp vs global temp views; `WHERE` vs `HAVING`; `LIKE` wildcards; join types; broadcast joins.*

---

## P7 — Apache Kafka: installation, producer and consumer (Unit V) ✅ *official lab item*

### Aim

To install and start Apache Kafka, create a topic, and demonstrate the **producer and consumer messaging model** using the console tools and Python, including keys, partitions, consumer groups and offsets.

### Theory

> **Apache Kafka is a distributed, fault-tolerant, high-throughput event streaming platform that stores streams of records and lets many applications publish and read them in real time.**

**Why a messaging layer?** Connecting every producer directly to every consumer creates M × N fragile links, loses data when a consumer is down, and cannot replay history. A central log gives **decoupling, buffering, durability, replay and many independent readers**.

```text
 PRODUCERS ──publish──▶  KAFKA CLUSTER (topics → partitions, replicated)  ──read──▶ CONSUMERS
                              ▲
                         ZooKeeper (metadata, controller election)
```

| Concept | Meaning |
| --- | --- |
| **Record** | key + value + timestamp (+ headers), stored as bytes |
| **Topic** | A named stream of records |
| **Partition** | An ordered, append-only log; the unit of parallelism |
| **Offset** | A record's position number within a partition (starts at 0, **per partition**) |
| **Producer** | Writes records; same key → same partition (`hash(key) mod partitions`) |
| **Consumer** | **Pulls** records and remembers its offset |
| **Consumer group** | Consumers sharing work: each partition is read by **one** member of the group |
| **Broker** | One Kafka server; a **cluster** is several brokers |
| **Replication** | Each partition has a **leader** and **follower** copies; the new leader comes from the **ISR** (in-sync replicas) |

**Cluster architecture in one picture**

```text
   Broker 1            Broker 2            Broker 3
   P0 (leader)         P1 (leader)         P2 (leader)
   P1 (follower)       P2 (follower)       P0 (follower)
   leader serves reads/writes ; followers copy ; if a leader dies, an ISR follower is promoted
```

**Guarantees:** ordering only **within a partition**; records are **kept** until retention expires (default 7 days), so a new consumer can replay from `earliest`; delivery semantics are *at most once*, *at least once* or *exactly once* depending on acknowledgements and offset commits.

### Steps (⚠️ Kafka outputs are *typical*; your IDs and timestamps differ)

**1. Configure** (`C:\kafka\config\zookeeper.properties` and `server.properties`; use forward slashes)

```properties
# zookeeper.properties
dataDir=C:/kafka/data/zookeeper

# server.properties
broker.id=0
listeners=PLAINTEXT://localhost:9092
log.dirs=C:/kafka/data/kafka-logs
zookeeper.connect=localhost:2181
```

**2. Start ZooKeeper (Window 1), then Kafka (Window 2)**

```powershell
cd C:\kafka
bin\windows\zookeeper-server-start.bat config\zookeeper.properties
bin\windows\kafka-server-start.bat config\server.properties
```

```text
... binding to port 0.0.0.0/0.0.0.0:2181                       (ZooKeeper ready)
... [KafkaServer id=0] started                                 (Kafka ready)
```

**3. Verify**

```powershell
jps
kafka-topics.bat --version
```

```text
4412 QuorumPeerMain          ← ZooKeeper
5876 Kafka                   ← broker
3.4.0 (Commit:2e1947d240607d53)
```

**4. Create and inspect a topic (Window 3)**

```powershell
kafka-topics.bat --create --topic sales --bootstrap-server localhost:9092 --partitions 3 --replication-factor 1
kafka-topics.bat --list --bootstrap-server localhost:9092
kafka-topics.bat --describe --topic sales --bootstrap-server localhost:9092
```

```text
Created topic sales.
sales
Topic: sales  PartitionCount: 3  ReplicationFactor: 1
    Topic: sales  Partition: 0  Leader: 0  Replicas: 0  Isr: 0
    Topic: sales  Partition: 1  Leader: 0  Replicas: 0  Isr: 0
    Topic: sales  Partition: 2  Leader: 0  Replicas: 0  Isr: 0
```

(Replication factor must be 1 on a single broker.)

**5. Console producer and consumer**

```powershell
REM Window 3: producer: type lines, press Enter after each
kafka-console-producer.bat --topic sales --bootstrap-server localhost:9092

REM Window 4: consumer: read everything from the start
kafka-console-consumer.bat --topic sales --from-beginning --bootstrap-server localhost:9092
```

```text
(producer)  >Laptop,2,55000        (consumer)  Laptop,2,55000
            >Phone,5,20000                     Phone,5,20000
```

A line typed in the producer appears in the consumer within moments: **real-time delivery**.

**6. Keys, partitions and offsets**

```powershell
kafka-console-producer.bat --topic sales --bootstrap-server localhost:9092 --property parse.key=true --property key.separator=:
kafka-console-consumer.bat --topic sales --from-beginning --bootstrap-server localhost:9092 --property print.key=true --property print.partition=true --property print.offset=true
```

Type `Delhi:Laptop,2`, `Mumbai:Phone,5`, `Delhi:Tablet,3`:

```text
Partition:0  Offset:0  Delhi   Laptop,2
Partition:0  Offset:1  Delhi   Tablet,3        ← same key → same partition, order kept
Partition:1  Offset:0  Mumbai  Phone,5
```

**7. Consumer groups**

```powershell
REM Two windows with the SAME group → they SHARE the records
kafka-console-consumer.bat --topic sales --group workers --bootstrap-server localhost:9092
REM Different group → receives ALL records independently
kafka-console-consumer.bat --topic sales --group audit --bootstrap-server localhost:9092
```

```powershell
kafka-consumer-groups.bat --bootstrap-server localhost:9092 --describe --group workers
```

```text
GROUP    TOPIC  PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG
workers  sales  0          2               2               0
workers  sales  1          1               1               0
```

`LAG = LOG-END-OFFSET − CURRENT-OFFSET` = records not yet read, the main health metric of a consumer.

**8. Producer and consumer in Python** (`pip install kafka-python`)

`producer.py`

```python
from kafka import KafkaProducer

producer = KafkaProducer(bootstrap_servers="localhost:9092",
                         key_serializer=lambda k: k.encode("utf-8"),
                         value_serializer=lambda v: v.encode("utf-8"))

for city, value in [("Delhi", "Laptop,2"), ("Mumbai", "Phone,5"), ("Delhi", "Tablet,3")]:
    meta = producer.send("sales", key=city, value=value).get(timeout=10)   # wait for the acknowledgement
    print(f"sent {city}:{value} -> partition={meta.partition} offset={meta.offset}")

producer.flush()
producer.close()
```

`consumer.py`

```python
from kafka import KafkaConsumer

consumer = KafkaConsumer("sales", bootstrap_servers="localhost:9092", group_id="py-group",
                         auto_offset_reset="earliest",
                         key_deserializer=lambda k: k.decode("utf-8") if k else None,
                         value_deserializer=lambda v: v.decode("utf-8"),
                         consumer_timeout_ms=10000)

for msg in consumer:
    print(f"partition={msg.partition} offset={msg.offset} key={msg.key} value={msg.value}")
consumer.close()
```

```text
sent Delhi:Laptop,2 -> partition=0 offset=0
partition=0 offset=0 key=Delhi value=Laptop,2
```

Kafka stores **bytes**, so the producer **serializes** and the consumer **deserializes**. `auto_offset_reset="earliest"` starts from the beginning when the group has no saved offset. Run the consumer twice: the second run prints nothing, because group `py-group` has already **committed** its offsets.

**9. Important producer/consumer settings**

| Setting | Meaning |
| --- | --- |
| `acks=0 / 1 / all` | Wait for nothing / the leader / all in-sync replicas (**safest**) |
| `enable.idempotence=true` | No duplicates from retries |
| `linger.ms`, `batch.size`, `compression.type` | Throughput tuning |
| `group.id` | Consumer group name |
| `auto.offset.reset` | `earliest` or `latest` when no saved offset |
| `enable.auto.commit` | Commit offsets automatically |

**Pipeline and ingestion patterns (theory for the viva).** Source → **ingestion** → Kafka (transport) → Spark (processing) → storage/serving. Styles: **batch**, **micro-batch**, **real-time**. Patterns: file→Kafka loading, log aggregation (fan-in), CDC (database changes as events), **fan-out** (many readers of one topic), **filter-and-route**, **enrichment** (stream joined with a static table), **dead-letter queue** (bad records to a separate topic), **de-duplication**, **replay**.

### Common errors

| Error | Cause | Fix |
| --- | --- | --- |
| `Timed out waiting for connection to ZooKeeper` | ZooKeeper not running | Start ZooKeeper first |
| `Connection refused` / `NoBrokersAvailable` | Broker down | `jps`; check `localhost:9092` |
| *The input line is too long* | Long install path | Install in `C:\kafka` |
| `InconsistentClusterIdException` | Deleted only one data folder | Delete **both** data folders |
| Consumer shows nothing | No `--from-beginning`, or group already committed | Add the flag / new group |
| Replication factor error | RF > brokers | Use `--replication-factor 1` |
| Broker crash after deleting a topic (Windows) | File locking | **Do not delete topics on Windows**; reset the data folders |

### Viva pointers for P7

*Kafka vs a traditional queue; topic/partition/offset; ordering guarantee; consumer groups (queue vs pub-sub); replication, leader, ISR; ZooKeeper vs KRaft; acks; delivery semantics; retention and replay; ingestion patterns.*

---

## P8 — Spark Streaming: DStreams, Structured Streaming and Kafka (Unit V) ✅ *official lab item*

### Aim

To process live data with Spark: (A) the classic **DStream** word count, (B) **Structured Streaming** with the socket and file sources, output modes and event-time windows, and (C) a **Kafka → Spark** pipeline.

### Theory

Spark processes streams in **micro-batches**: the stream is cut into small time slices, and each slice runs as a small batch job (high throughput, easy recovery, latency from ~100 ms to seconds).

| | **DStreams (Spark Streaming)** | **Structured Streaming** |
| --- | --- | --- |
| Abstraction | Sequence of RDDs | **Unbounded table (streaming DataFrame)** |
| API | `map`, `reduceByKey` | `select`, `groupBy`, SQL |
| Optimizer | None | **Catalyst** |
| State | Manual (`updateStateByKey`) | Automatic |
| Time | Processing time | **Event time + watermarks** |
| Exactly-once | Hard | Built in (replayable source + checkpoint) |
| Status | **Legacy** | **Recommended** |

> **Important for the shell:** multi-line code with lines starting with `.` must be typed with **`:paste`** (finish with **Ctrl + D**). Typed line by line, only the first line runs.

**A text source for practice (Windows has no `nc`).** Save `C:\sparklabs\socket_server.py`:

```python
import socket
server = socket.socket(); server.bind(("localhost", 9999)); server.listen(1)
print("Waiting for Spark on port 9999 ...")
conn, addr = server.accept(); print("Spark connected")
while True:
    conn.sendall((input("type a line> ") + "\n").encode("utf-8"))
```

Start it **first**, then start the Spark query.

### Part A: DStream word count

```scala
import org.apache.spark.streaming.{StreamingContext, Seconds}

val ssc    = new StreamingContext(sc, Seconds(2))          // 2-second micro-batches
val lines  = ssc.socketTextStream("localhost", 9999)
val counts = lines.flatMap(_.split(" ")).map((_, 1)).reduceByKey(_ + _)
counts.print()

ssc.start()
ssc.awaitTerminationOrTimeout(30000)
ssc.stop(stopSparkContext = false)                          // keep the shell's sc alive
```

Type `hello spark hello kafka`, then `kafka streams spark` in the server window. **Output (verified):**

```text
-------------------------------------------
Time: ... ms
-------------------------------------------
(hello,2)
(kafka,1)
(spark,1)

-------------------------------------------
Time: ... ms          (the next message)
-------------------------------------------
(kafka,1)
(spark,1)
(streams,1)
```

Each batch counts **only its own data** (`kafka` shows 1, not 2): DStream operations are **stateless** by default. For running totals use `updateStateByKey` (with `ssc.checkpoint(path)`); for "the last N seconds" use `reduceByKeyAndWindow(_ + _, Seconds(6), Seconds(2))`.

### Part B: Structured Streaming

**B1. Word count from a socket, with running totals**

```scala
import org.apache.spark.sql.functions._
import org.apache.spark.sql.streaming.Trigger
import spark.implicits._

val lines  = spark.readStream.format("socket").option("host", "localhost").option("port", 9999).load()
val words  = lines.select(explode(split($"value", " ")).as("word"))
val counts = words.groupBy("word").count()

val q = counts.writeStream.format("console").outputMode("complete")
          .trigger(Trigger.ProcessingTime("3 seconds")).start()
// use :paste for the multi-line chain
```

```text
Batch: 1                         Batch: 2
+-----+-----+                    +-------+-----+
| word|count|                    |   word|count|
+-----+-----+                    +-------+-----+
|hello|    2|                    |  hello|    2|
|spark|    1|                    |streams|    1|
|kafka|    1|                    |  spark|    2|
+-----+-----+                    |  kafka|    2|
                                 +-------+-----+
```

**The counts accumulate automatically** (`kafka` 1 → 2): Structured Streaming keeps the aggregation state for us. Stop with `q.stop()`.

**B2. A streaming query is a query on an ever-growing table**

```text
 stream ─▶ [row][row][row]...  (input table grows)  ─▶ your DataFrame query ─▶ result table ─▶ sink
 readStream  →  transform (select / filter / groupBy / join)  →  writeStream  →  start()
```

**B3. Output modes (verified with a file source)**

Create the folder `C:\sparklabs\stream-in` and run:

```scala
import org.apache.spark.sql.types._
val schema = new StructType().add("order_id", IntegerType).add("product", StringType)
  .add("city", StringType).add("quantity", IntegerType).add("price", IntegerType)

val sales = spark.readStream.schema(schema).csv("C:/sparklabs/stream-in")      // file streams need a schema
val rev   = sales.groupBy("product").agg(sum(col("quantity") * col("price")).alias("revenue"))
val q = rev.writeStream.format("console").outputMode("complete").start()        // use :paste
```

Copy `f1.csv` (`1,Laptop,Delhi,2,55000` / `2,Phone,Mumbai,5,20000`), then `f2.csv` (`3,Laptop,Mumbai,1,55000` / `4,Tablet,Delhi,3,15000`) into the folder:

```text
complete mode:
Batch 0:  Laptop 110000 | Phone 100000
Batch 1:  Tablet 45000 | Laptop 165000 | Phone 100000          ← the WHOLE table each time

update mode (then add f3.csv: 5,Phone,Delhi,2,20000):
Batch 1:  Phone 140000                                          ← only the CHANGED row
```

| Mode | Writes | Works for |
| --- | --- | --- |
| `append` | Only **new rows** | Queries **without** aggregation (or aggregation with a watermark) |
| `update` | Only **changed rows** | Aggregations and non-aggregations |
| `complete` | The **entire** result table | **Aggregations** only |

An aggregation in **append** mode is rejected: *"Append output mode not supported when there are streaming aggregations … without watermark"*, because an aggregate keeps changing but append writes a row once.

**B4. Event-time windows and watermarks (verified)**

```scala
val ev = spark.readStream.schema(new StructType().add("event_time", StringType).add("city", StringType).add("amount", IntegerType))
             .csv("C:/sparklabs/stream-in2").withColumn("ts", to_timestamp($"event_time"))
val agg = ev.groupBy(window($"ts", "10 minutes"), $"city").agg(sum("amount").as("total"))
```

With the file `2026-11-18 10:01:00,Delhi,100` / `10:04:00,Delhi,50` / `10:12:00,Delhi,70` / `10:03:00,Mumbai,30`:

```text
|win_start          |win_end            |city  |total|
|2026-11-18 10:00:00|2026-11-18 10:10:00|Delhi |150  |
|2026-11-18 10:00:00|2026-11-18 10:10:00|Mumbai|30   |
|2026-11-18 10:10:00|2026-11-18 10:20:00|Delhi |70   |
```

**Event time** is when the event *happened* (a column); **processing time** is when Spark *received* it. A **watermark** (`withWatermark("ts", "10 minutes")`) says how late data may arrive, so Spark can drop old state and bound memory.

**B5. Checkpointing.** Add `.option("checkpointLocation", "C:/sparkdata/checkpoints/q1")`. The folder holds `offsets`, `commits`, `metadata`, `sources` and `state`, so a restarted query **continues without loss or duplication**. Use **one checkpoint folder per query**.

### Part C: Kafka → Spark pipeline (⚠️ connector lines are *typical*)

Start Spark with the connector for **your** versions (the `_2.13` must match Spark's Scala; `3.5.9` must match Spark):

```powershell
spark-shell --packages org.apache.spark:spark-sql-kafka-0-10_2.13:3.5.9 --conf spark.sql.shuffle.partitions=4
```

Run the generator (`sales_generator.py`, one CSV event per second `order_id,product,city,quantity,price` into topic `sales`), then (using `:paste`):

```scala
val raw = spark.readStream.format("kafka")
  .option("kafka.bootstrap.servers", "localhost:9092")
  .option("subscribe", "sales")
  .option("startingOffsets", "earliest")
  .load()

raw.printSchema()
```

```text
 |-- key: binary          |-- topic: string       |-- offset: long
 |-- value: binary        |-- partition: integer  |-- timestamp: timestamp  |-- timestampType: integer
```

**`key` and `value` are binary: always cast, then parse.**

```scala
val p = split($"value".cast("string"), ",")
val sales = raw.select(
  p.getItem(0).cast("int").as("order_id"), p.getItem(1).as("product"), p.getItem(2).as("city"),
  p.getItem(3).cast("int").as("quantity"), p.getItem(4).cast("int").as("price"))

val revenue = sales.groupBy("product").agg(sum($"quantity" * $"price").as("revenue"))

val q = revenue.writeStream.format("console").outputMode("complete")
          .trigger(Trigger.ProcessingTime("5 seconds"))
          .option("checkpointLocation", "C:/sparkdata/checkpoints/rev").start()
```

✅ The parse-and-aggregate logic was verified on a DataFrame with the exact Kafka schema: for the 7 valid sample events below the final table is `Laptop 330000 | Phone 140000 | Tablet 105000`.

```text
1,Laptop,Delhi,2,55000     4,Laptop,Mumbai,1,55000     7,Laptop,Pune,3,55000
2,Phone,Mumbai,5,20000     5,Phone,Delhi,2,20000       bad,record   ← becomes NULL fields (bad data)
3,Tablet,Pune,3,15000      6,Tablet,Delhi,4,15000
```

**Writing results back to Kafka** (a `value` column is required; `key` is optional):

```scala
val alerts = sales.withColumn("revenue", $"quantity" * $"price").filter($"revenue" >= 100000)
  .select($"city".as("key"), to_json(struct($"order_id", $"product", $"city", $"revenue")).as("value"))

alerts.writeStream.format("kafka")
  .option("kafka.bootstrap.servers", "localhost:9092").option("topic", "sales-alerts")
  .option("checkpointLocation", "C:/sparkdata/checkpoints/alerts").start()
```

**Dead-letter queue for bad records** (bad rows have NULL fields; send `raw` text of those rows to `sales-dlq`):

```scala
val bad = all.filter($"order_id".isNull || $"quantity".isNull).select($"raw".as("value"))   // 'all' keeps the raw text
```

**Key facts about the integration**

| Fact | Detail |
| --- | --- |
| One Spark task per Kafka partition | Partitions control read parallelism |
| `startingOffsets` | Used **only on the first run**; afterwards the **checkpoint** decides |
| Offsets are kept in Spark's **checkpoint**, not Kafka consumer groups | So `kafka-consumer-groups` does not show Spark's lag |
| Kafka source + checkpoint | **Exactly-once** processing |
| Kafka **sink** | **At-least-once** → de-duplicate downstream (`dropDuplicates("order_id")`) |
| `maxOffsetsPerTrigger` | Limits batch size (rate control) |
| After resetting Kafka data | **Delete the old checkpoints too** |

### Common errors

| Error | Cause | Fix |
| --- | --- | --- |
| `Connection refused` (socket) | Server not started | Start the Python server first |
| Only the first line of a chain runs | Typed line by line | `:paste` + Ctrl + D |
| `Failed to find data source: kafka` | Connector missing | Add `--packages ...spark-sql-kafka-0-10_2.13:3.5.9` |
| `NoSuchMethodError` at start | Scala/Spark version mismatch | Match `_2.13` and `3.5.9` |
| Hex bytes on screen | Forgot the cast | `CAST(value AS STRING)` |
| `Append output mode not supported` | Aggregation in append mode | `complete` / `update` |
| Offset/data-loss error on start | Old checkpoint after Kafka reset | Delete the checkpoint folder |
| `show()` on a stream fails | Streaming DataFrame | Use `writeStream` |

### Viva pointers for P8

*Micro-batching; DStream vs Structured Streaming; output modes; triggers; checkpointing; event time and watermarks; Kafka source/sink semantics; exactly-once; why cast `value`.*

---

## P9 — MLlib with PySpark: regression, classification, clustering and evaluation (Unit VI) ✅ *official lab item*

### Aim

To build, evaluate and compare machine-learning models with Spark MLlib in Python: correlation analysis with a **heat map**, **Linear Regression**, **Logistic Regression**, **Decision Tree**, **SVM**, **Naive Bayes**, **K-means**, and the evaluation metrics (**confusion matrix, R², RMSE, MAE**).

### Theory

**ML** learns a rule from data. *Supervised* learning has labels (**regression** predicts a number; **classification** a category); *unsupervised* learning has none (**clustering**).

**Spark ML grammar**

| Concept | Meaning |
| --- | --- |
| **Transformer** | `transform()` adds columns (`VectorAssembler`, any fitted model) |
| **Estimator** | `fit()` learns and returns a **Model** (`LinearRegression`, `StandardScaler`) |
| **Pipeline** | A chain of stages; `fit()` → `PipelineModel` |
| **Evaluator** | Computes a metric |

Features must be **one Vector column**; fit on **train**, evaluate on **test**.

### Steps and code (PySpark; every number below is a real output)

**1. Data and set-up**

```python
import sys
sys.path.append(r"C:\sparklabs")
from make_data import make_students
from pyspark.ml import Pipeline
from pyspark.ml.feature import VectorAssembler, StandardScaler
from pyspark.ml.regression import LinearRegression
from pyspark.ml.classification import LogisticRegression, DecisionTreeClassifier, LinearSVC, NaiveBayes
from pyspark.ml.clustering import KMeans
from pyspark.ml.stat import Correlation
from pyspark.ml.evaluation import (RegressionEvaluator, BinaryClassificationEvaluator,
                                   MulticlassClassificationEvaluator, ClusteringEvaluator)
from pyspark.sql import functions as F

df = spark.createDataFrame(make_students(), ["id", "hours", "attendance", "prev_marks", "final_marks", "passed"])
feats = ["hours", "attendance", "prev_marks"]
df.groupBy("passed").count().orderBy("passed").show()
```

```text
+------+-----+
|passed|count|
+------+-----+
|     0|  100|
|     1|  100|
+------+-----+
```

The data follows the hidden rule `final ≈ 4·hours + 0.25·attendance + 0.35·prev + 5 (+ noise)`, so we can check that models recover it.

**2. Correlation and the heat map**

```python
import pandas as pd, matplotlib.pyplot as plt, seaborn as sns
cols = ["hours", "attendance", "prev_marks", "final_marks"]
v = VectorAssembler(inputCols=cols, outputCol="v").transform(df)
corr = pd.DataFrame(Correlation.corr(v, "v", "pearson").head()[0].toArray(), index=cols, columns=cols)
print(corr.round(3))

sns.heatmap(corr, annot=True, fmt=".2f", cmap="coolwarm", vmin=-1, vmax=1)
plt.title("Correlation heat map"); plt.show()
```

```text
             hours  attendance  prev_marks  final_marks
hours        1.000       0.015       0.007        0.834
attendance   0.015       1.000       0.059        0.318
prev_marks   0.007       0.059       1.000        0.419
final_marks  0.834       0.318       0.419        1.000
```

`hours` is the strongest predictor (r = 0.834); the features are almost uncorrelated with each other (no multicollinearity).

**3. Linear Regression: predict `final_marks`** (a number)

```python
d = df.select(*feats, F.col("final_marks").alias("label"))
train, test = d.randomSplit([0.8, 0.2], seed=42)

pipe = Pipeline(stages=[VectorAssembler(inputCols=feats, outputCol="features"),
                        LinearRegression(featuresCol="features", labelCol="label")])
model = pipe.fit(train)
pred = model.transform(test)

lr = model.stages[-1]
print("coefficients:", lr.coefficients, " intercept:", round(lr.intercept, 3))
ev = RegressionEvaluator(labelCol="label")
print({m: round(ev.setMetricName(m).evaluate(pred), 4) for m in ("rmse", "mae", "r2")})
```

```text
coefficients: [4.0612,0.2479,0.34]   intercept: 5.867
{'rmse': 2.8413, 'mae': 2.2595, 'r2': 0.9291}
```

The learned coefficients are close to the hidden truth (4, 0.25, 0.35, 5). On unseen students predictions are off by about **2.84 marks** (RMSE), **2.26 marks** on average (MAE), and the model explains **92.9%** of the variance (R²).

*Maths:* least squares minimizes `SSE = Σ(y − ŷ)²`; `R² = 1 − SSE/SST`; `RMSE = √MSE`; `MAE = Σ|y − ŷ|/n`.

**4. Classification: predict `passed`** (compare four algorithms on the same split)

```python
c = df.select(*feats, F.col("passed").alias("label"))
ctrain, ctest = c.randomSplit([0.8, 0.2], seed=42)
va = VectorAssembler(inputCols=feats, outputCol="features")
ctr, cte = va.transform(ctrain), va.transform(ctest)

models = {"Logistic Regression": LogisticRegression(maxIter=100),
          "Decision Tree": DecisionTreeClassifier(maxDepth=4, seed=42),
          "Linear SVM": LinearSVC(maxIter=100, regParam=0.1),
          "Naive Bayes (Gaussian)": NaiveBayes(modelType="gaussian")}

acc = MulticlassClassificationEvaluator(metricName="accuracy")
auc = BinaryClassificationEvaluator(metricName="areaUnderROC")
for name, est in models.items():
    p = est.fit(ctr).transform(cte)
    print(name, "| accuracy", round(acc.evaluate(p), 4), "| AUC", round(auc.evaluate(p), 4))
```

| Model | Test accuracy | AUC |
| --- | --- | --- |
| Linear SVM (regParam 0.1) | **0.9429** | 0.988 |
| Logistic Regression | 0.9143 | 0.988 |
| Decision Tree (depth 4) | 0.8857 | 0.880 |
| Naive Bayes (Gaussian) | 0.8286 | 0.808 |
| *Baseline: always "fail"* | *0.7143* | *0.5* |

The linear models win because the data is generated by a linear formula. Always compare against the **baseline**; with 35 test rows, one student moves accuracy by about 3 points.

**5. Confusion matrix and metrics** (logistic regression)

```python
tr2, te2 = c.randomSplit([0.7, 0.3], seed=7)            # a 70:30 split: 133 train, 67 test rows
pred = LogisticRegression(maxIter=100).fit(va.transform(tr2)).transform(va.transform(te2))
pred.groupBy("label", "prediction").count().orderBy("label", "prediction").show()
```

```text
Confusion matrix on a 70:30 split (rows = actual, columns = predicted):
                  Pred 0   Pred 1
 Actual 0 (fail)    33        2        TN = 33, FP = 2
 Actual 1 (pass)     1       31        FN = 1,  TP = 31

 Accuracy  = (31 + 33)/67 = 0.9552      Precision = 31/33 = 0.9394
 Recall    = 31/32        = 0.9688      F1        = 0.9538      AUC = 0.9866
```

```python
sns.heatmap([[33, 2], [1, 31]], annot=True, fmt="d", cmap="Blues",
            xticklabels=["Pred 0", "Pred 1"], yticklabels=["Actual 0", "Actual 1"]); plt.show()
```

**6. K-means: segment customers** (no labels)

```python
cust = [(1,15,20),(2,18,25),(3,20,15),(4,22,22),(5,18,80),(6,20,85),(7,24,78),(8,22,82),
        (9,80,18),(10,85,22),(11,88,15),(12,90,20),(13,82,80),(14,86,85),(15,90,78),(16,88,82)]
cdf = spark.createDataFrame(cust, ["id", "income", "score"])

pipe = Pipeline(stages=[VectorAssembler(inputCols=["income", "score"], outputCol="features"),
                        StandardScaler(inputCol="features", outputCol="scaled", withMean=True, withStd=True),
                        KMeans(featuresCol="scaled", k=4, seed=42)])
m = pipe.fit(cdf); out = m.transform(cdf)
print(m.stages[-1].summary.clusterSizes,
      round(ClusteringEvaluator(featuresCol="scaled").evaluate(out), 4))
```

```text
[4, 4, 4, 4]  0.9882
```

Choosing K with the **elbow** (WSSSE 15476.6, 8085.4, **271.8**, 223.2, 180.8 for K = 2…6) and the **silhouette** (peaks at **0.988 for K = 4**). Scale first: without scaling, income in rupees dominates and the clusters come out as `[5, 6, 3, 2]` (wrong).

**7. Tune with cross-validation and save the model**

```python
from pyspark.ml.tuning import ParamGridBuilder, CrossValidator
lr = LogisticRegression(maxIter=100)
pipe = Pipeline(stages=[va, StandardScaler(inputCol="features", outputCol="sf"), lr.setFeaturesCol("sf")])
grid = ParamGridBuilder().addGrid(lr.regParam, [0.0, 0.01, 0.1]).addGrid(lr.elasticNetParam, [0.0, 0.5]).build()
cv = CrossValidator(estimator=pipe, estimatorParamMaps=grid,
                    evaluator=BinaryClassificationEvaluator(), numFolds=3, seed=42)
best = cv.fit(ctrain)                                   # 6 combinations × 3 folds = 18 fits
best.bestModel.write().overwrite().save("C:/sparklabs/models/passfail")
```

Best `regParam = 0.01` (CV AUC 0.9951); test AUC 0.984. Reload with `PipelineModel.load(path)` to score new data.

### Algorithm cheat card (maths in one line each)

| Algorithm | Core idea / formula |
| --- | --- |
| **Linear Regression** | `ŷ = β₀ + β₁x₁ + …`; minimize SSE; `β = (XᵀX)⁻¹Xᵀy` |
| **Logistic Regression** | `p = 1/(1 + e⁻ᶻ)`; log-loss; odds ratio `e^β` |
| **Decision Tree** | Split by highest information gain; entropy `−Σ p log₂ p`, Gini `1 − Σ p²` |
| **K-means** | Assign to nearest centroid → recompute means; minimize WCSS |
| **SVM** | Maximize margin `2/‖w‖`; hinge loss `max(0, 1 − y f)` |
| **Naive Bayes** | `P(C|X) ∝ P(C) ΠP(xᵢ|C)`; Laplace smoothing |

### Common errors

| Error | Cause | Fix |
| --- | --- | --- |
| `Column features must be of type VectorUDT` | No `VectorAssembler` | Assemble features |
| `label does not exist` | Target not named `label` | `.alias("label")` |
| Perfect accuracy | **Leakage** (`final_marks` used to predict `passed`) | Remove it |
| Train ≫ test accuracy | **Overfitting** | Regularize, lower `maxDepth`, CV |
| K-means clusters wrong | Unscaled features | `StandardScaler` |
| `Vector values MUST NOT be Negative` | Negative features with multinomial Naive Bayes | Gaussian NB / MinMax scale |
| Low AUC | Passed hard predictions | Use `rawPrediction` / `probability` |
| Python version mismatch | Two Pythons | `PYSPARK_PYTHON` = `PYSPARK_DRIVER_PYTHON` |

### Viva pointers for P9

*Supervised vs unsupervised; Transformer/Estimator/Pipeline; overfitting and bias-variance; each algorithm's idea and loss; confusion matrix, precision/recall/F1, ROC-AUC; RMSE/MAE/R²; correlation heat map; cross-validation; leakage; scaling.*

---

## P10 — Integrated capstone: a real-time retail analytics pipeline (all units)

### Aim

To combine the whole syllabus into one working design: **ingest** events with Kafka, **process** them in real time with Spark Structured Streaming, **store** clean data, analyse it with **Spark SQL**, and **score** it with a **MLlib** model.

### Architecture

```text
 sales_generator.py ──▶ [Kafka: retail-sales] ──▶ Spark Structured Streaming ──┬─▶ valid ─▶ revenue per product (console)
   (Unit V: producer)     (Unit V: topic,           (Unit V: readStream)       │         └▶ revenue ≥ 100000 ─▶ [Kafka: retail-alerts]
                           partitions)                                         │         └▶ clean data ─▶ CSV/Parquet files   (storage)
                                                                               └─▶ invalid ─▶ [Kafka: retail-dlq]
                                                                                                       │
              Spark SQL (Unit IV): join with product catalogue, GROUP BY, HAVING  ◀────────────────────┘ (batch read of stored files)
              MLlib (Unit VI): K-means customer/product segments, or a saved PipelineModel scoring the stream
              Spark core (Units I-III): RDD/DataFrame engine, caching, shared variables, lineage, fault tolerance
```

### Steps

1. **Start the platform:** ZooKeeper → Kafka (P7); create topics `retail-sales`, `retail-alerts`, `retail-dlq`.
2. **Ingest:** run the Python generator, which publishes CSV events `order_id,product,city,quantity,price`.
3. **Process (P8 Part C):** read, cast, parse, **validate** (NULL fields → dead-letter topic), aggregate revenue per product (`complete` mode), route high-value orders to `retail-alerts`, and write clean rows to files with a **checkpoint**.
4. **Analyse (P6):** read the stored files as a batch DataFrame, register a view, join with a product table, and report revenue per category.
5. **Learn (P9):** cluster products or customers with K-means, or train a model offline and apply the saved `PipelineModel` to the stream with `model.transform(stream_df)`.

```scala
// Step 4: batch analysis of the files written by the streaming job (Unit IV)
val clean = spark.read.option("header", "false").schema(schema).csv("C:/sparklabs/out/clean")
clean.createOrReplaceTempView("sales")
val cat = Seq(("Laptop", "Computing"), ("Tablet", "Computing"), ("Phone", "Mobile")).toDF("product", "category")
cat.createOrReplaceTempView("categories")
spark.sql("""SELECT c.category, SUM(s.quantity * s.price) AS revenue
             FROM sales s JOIN categories c ON s.product = c.product
             GROUP BY c.category ORDER BY revenue DESC""").show()
```

✅ For the 7 valid sample events of P8 (Laptop 330,000 + Tablet 105,000 + Phone 140,000) the result is `Computing 435000 | Mobile 140000` (verified logic).

### Design reasons (what to say in an interview)

| Choice | Reason |
| --- | --- |
| Kafka between producer and Spark | Decouples, buffers bursts, retains events for replay, supports many consumers |
| Raw topic kept untouched | Allows reprocessing after a bug fix (replay with a new checkpoint) |
| Dead-letter topic | One bad record never blocks the pipeline and is never lost silently |
| Checkpoints | Exactly-once processing and recovery after a crash |
| `reduceByKey`/aggregations instead of `groupByKey` | Less shuffling |
| De-duplication by `order_id` | The Kafka sink is at-least-once |
| Model trained offline, scored online | Standard ML deployment pattern |

### Viva pointers for P10

*Explain the data flow end to end; where fault tolerance comes from at each stage; how you would scale it (more partitions/brokers/executors); how you would monitor it (consumer lag, batch duration, input vs processing rate).*

---

# PART B — THE VIVA AND INTERVIEW QUESTION BANK

> **How to answer (the structure examiners and interviewers like):**
> 1. **Define** it in one clear sentence.
> 2. **Explain how it works** (the mechanism, not just the name).
> 3. **Give a concrete example** (use the data from the practicals).
> 4. **State the trade-off or limitation** (this is what signals real understanding).
>
> Each answer below follows that order. Say it in your own words; do not memorise it word for word.

---

## UNIT I: Introduction to Spark

**Q1. What is Apache Spark, and why was it created?**

Apache Spark is an open-source, distributed, in-memory data-processing engine for large-scale analytics. It was created at UC Berkeley because **MapReduce was a poor fit for two growing workloads**: *iterative* algorithms (machine learning, graph processing), which re-read the same data many times, and *interactive* analysis, which needs answers in seconds. MapReduce writes every intermediate result to disk (HDFS) and re-reads it for the next job, so multi-step pipelines pay a heavy disk and latency cost. Spark keeps working data **in memory** across steps, plans a whole job as a **DAG**, and gives one API for SQL, streaming, machine learning and graphs. In short: *Spark makes multi-step and iterative big-data processing fast and simple.*

**Q2. What are the limitations of MapReduce in Hadoop?**

There are five main ones. **(1) Disk I/O between stages:** each job reads from and writes to HDFS, which dominates run time. **(2) Rigid two-step model:** everything must be expressed as map and reduce, so a complex pipeline becomes a chain of jobs. **(3) Poor for iterative work:** an algorithm that loops 50 times reads from disk 50 times. **(4) High latency:** jobs take minutes, so it is unsuitable for interactive or real-time use. **(5) Verbose programming:** a word count takes dozens of lines of Java, and there is no interactive shell. These are exactly the gaps Spark closes with in-memory caching, a DAG engine and concise APIs.

**Q3. How does Spark compare with Hadoop MapReduce? Does Spark replace Hadoop?**

Spark is faster (in-memory, up to 10 to 100× for iterative jobs), easier (concise APIs and a shell), and broader (batch, streaming, SQL, ML and graph in one engine, whereas MapReduce is batch only). However, **Spark does not replace the whole of Hadoop; it replaces MapReduce as the processing engine.** Spark has no storage of its own, so it usually reads from **HDFS** and often runs on **YARN**, which are Hadoop components. MapReduce can still be economical for huge, one-pass batch jobs on commodity disks where memory is scarce. The best summary: *Spark and Hadoop are complementary, and the Spark-on-YARN-over-HDFS stack is very common.*

**Q4. What is the difference between batch and real-time analytics, and where is stream processing applied?**

**Batch analytics** processes a finite, already-collected dataset on a schedule (for example a nightly sales report) and tolerates latency of minutes to hours. **Real-time (stream) analytics** processes unbounded data **as it arrives** and responds in milliseconds to seconds. Stream processing is applied wherever the value of an answer decays quickly: **fraud detection** on card swipes, live **recommendations**, **IoT** alerts (a machine overheating), **log monitoring**, trending topics on social media, and dynamic pricing. In Spark, batch uses RDDs/DataFrames/SQL, while streaming uses **Structured Streaming** (micro-batches), so the same engine and code style serve both.

**Q5. What is in-memory processing, why is it faster, and when does it not help?**

In-memory processing means intermediate data stays in **RAM** between operations instead of being written to disk. RAM access is orders of magnitude faster than disk, and Spark also avoids repeated serialization and HDFS replication between steps. A dataset used repeatedly (for example by a machine-learning loop) is `cache()`d once and then read from memory every iteration. It does **not** help when data is used **once** (a single pass gains little), when the data **does not fit** in memory (Spark spills to disk, losing the advantage), or when the job is dominated by a **shuffle** across the network. "In-memory" therefore means *memory is used wherever possible*, not that disk is never touched.

**Q6. What are the features and benefits of Spark?**

**Speed** (in-memory DAG execution), **ease of use** (high-level APIs in Scala, Java, Python, R and SQL, plus an interactive shell), **generality** (Spark SQL, Streaming, MLlib and GraphX share one engine and data model), **fault tolerance** (lost partitions are recomputed from lineage rather than replicated), **lazy evaluation** (the optimizer sees the whole plan before running), and **portability** (runs on local mode, Standalone, YARN or Kubernetes, and reads HDFS, S3, Kafka, JDBC and more). The benefit to a team is one platform and one skill set for ETL, analytics, streaming and machine learning, instead of stitching together separate systems.

**Q7. Describe the Spark architecture and what happens when you run a job.**

A Spark application has a **driver** (runs your `main` program, builds the plan and schedules work), a **cluster manager** (Standalone, YARN or Kubernetes, which allocates resources), and **executors** (JVM processes on worker nodes that run **tasks** and hold cached data). When you call an action, the driver turns the chain of transformations into a **DAG**, the **DAG scheduler** splits it into **stages** at shuffle boundaries, each stage into **tasks** (one per partition), and the task scheduler sends tasks to executors. Results return to the driver. For example, in word count, `flatMap` and `map` form one stage, and the shuffle for `reduceByKey` starts the next. Understanding *job → stage → task* is the key to reading the Spark UI.

**Q8. How do you install Spark as a standalone user, and what modes can it run in?**

Install JDK 8 and set `JAVA_HOME`, download the pre-built Spark package, extract it to a short path (for example `C:\spark`), set `SPARK_HOME` and add `bin` to `PATH`; on Windows also provide `winutils.exe` through `HADOOP_HOME`. Verify with `spark-shell` (the banner shows the Spark and Scala versions) and open the UI at port 4040. Spark runs in **local mode** (`local[N]`, one machine, for development), **Standalone** (Spark's own master and workers), **YARN** (shared Hadoop cluster) or **Kubernetes** (containers). Applications are submitted with `spark-submit --master <url> app`. No cluster or admin rights are needed for local and standalone learning.

**Q9. What are the components of the Spark ecosystem?**

**Spark Core** (RDDs, scheduling, memory management, fault recovery) is the foundation. On top sit **Spark SQL** (structured data, DataFrames), **Spark Streaming / Structured Streaming** (live data), **MLlib** (machine learning) and **GraphX** (graph processing). Cluster managers (Standalone, YARN, Kubernetes) and storage systems (HDFS, S3, Kafka, JDBC sources) sit beside it. The design point is that all libraries share the same engine and data abstractions, so one application can read from Kafka, clean with SQL, train a model and write results without moving data between systems.

**Q10. Why is Spark's lazy evaluation important, and how does it relate to fault tolerance?**

Spark records transformations as a plan and executes only when an **action** requires a result. That lets the optimizer see the whole pipeline: it can pipeline narrow transformations in one pass, skip unused work and push filters down. The recorded plan is the **lineage** of each dataset, which also provides fault tolerance: if a partition is lost, Spark replays just the transformations needed to rebuild it, instead of keeping multiple replicas. So laziness gives *efficiency*, and lineage gives *recovery*, both from the same idea.

---

## UNIT II: Programming in Scala

**Q1. What are the main features of Scala, and why is Spark written in it?**

Scala is a **statically typed language that blends object-oriented and functional programming** on the JVM. Its key features are **type inference** (concise code with compile-time safety), **immutability by default** (`val`, immutable collections), **first-class functions** (functions are values you can pass around), **pattern matching and case classes**, **traits**, and **full Java interoperability**. Spark is written in Scala because functional operations (`map`, `filter`, `reduce`) map naturally onto distributed collections, the JVM gives good performance and access to the Hadoop ecosystem, and the language is expressive enough to build a fluent API. As a result, Scala exposes the newest Spark features first, and its code is short.

**Q2. What is the difference between `val` and `var`, and why prefer `val`?**

`val` declares an **immutable reference**: once assigned it cannot be reassigned. `var` is **mutable** and can be reassigned. Prefer `val` because immutable values are easier to reason about, cannot be changed unexpectedly by other code, and are **safe to share across threads and executors**, which matters in parallel computing. Spark's own data structures (RDDs, DataFrames) are immutable: every transformation returns a *new* dataset, which is what makes lineage-based recovery possible. Use `var` only for small local state such as counters in a loop.

**Q3. What is type inference in Scala?**

Type inference means the compiler **works out the type of an expression from its value**, so you need not write it: `val x = 42` is inferred as `Int` and `val names = List("a", "b")` as `List[String]`. The program is still **statically typed**: the type is fixed at compile time and errors are caught early. You still write types explicitly for **function parameters and recursive functions** (`def square(n: Int): Int`) and where inference would be ambiguous. This gives the safety of Java with much of the brevity of Python.

**Q4. List the basic data types and literals in Scala. How does this differ from Java?**

`Int` (32-bit, `42`), `Long` (`5000000000L`), `Short`, `Byte`, `Double` (`3.14`), `Float` (`2.5f`), `Char` (`'A'`), `String` (`"Spark"`), `Boolean` (`true`), plus `Unit` (no value) and `Any`/`Nothing` at the top and bottom of the hierarchy. The main difference from Java is that **in Scala everything is an object**: there are no separate "primitive" types, even `5` has methods (`5.toString`, `5 max 3`). The compiler still uses efficient JVM primitives underneath. Also `String` interpolation (`s"Hello $name"`) and multi-line strings (`"""..."""`) are built in.

**Q5. What are operators and methods in Scala?**

In Scala, **operators are methods**: `a + b` is just `a.+(b)`, and any single-argument method can be called with infix notation (`list contains 3`). So arithmetic (`+ - * / %`), comparison (`> < == !=`), logical (`&& || !`) and bitwise operators are all methods defined on the types. Note that `10 / 3` is integer division giving `3`, while `10 / 3.0` gives `3.333`. This uniform design lets libraries define their own operators (as Spark does with `===` for columns) and makes the language very consistent.

**Q6. Explain functions in Scala, including anonymous and higher-order functions.**

A named function is declared with `def square(n: Int): Int = n * n`. An **anonymous function (lambda)** has no name: `(a: Int, b: Int) => a + b`, often abbreviated with the placeholder `_` as in `_ * 2`. A **higher-order function** takes a function as an argument or returns one: `def applyTwice(f: Int => Int, v: Int) = f(f(v))`. Functions can also have **default** and **named** parameters. This is the heart of Spark programming: `rdd.map(_ * 2)` passes a function to a distributed collection, and Spark ships that function (a **closure**) to the executors to run on each partition.

**Q7. What is the difference between mutable and immutable collections?**

**Immutable** collections (the default, in `scala.collection.immutable`) cannot be changed after creation: operations such as `list :+ 4` return a **new** collection and leave the original untouched. **Mutable** collections (`ListBuffer`, `ArrayBuffer`, `mutable.Map`, `Array`) are modified in place. Immutable ones are safer in concurrent code and match the functional style Spark uses; mutable ones can be more efficient for local, single-threaded building of data (for example filling a buffer in a loop). The usual practice is to build with a mutable collection if needed, then convert to an immutable one (`buf.toList`).

**Q8. Compare `List`, `Array` and `Vector`.**

A `List` is an immutable **singly linked list**: very fast to prepend (`::`) and to take the head and tail, but slow for random access (O(n)). An `Array` is a mutable, fixed-size, contiguous structure with fast indexed access (O(1)), mapping to a Java array. A `Vector` is an immutable structure with **effectively constant-time** indexed access and updates. Use `List` for recursive/functional processing, `Array` when you need in-place indexed performance, and `Vector` for general-purpose immutable sequences. Spark's `collect()` returns an `Array`; `parallelize` accepts any `Seq`.

**Q9. What are `Map`, `Option` and pattern matching, and how are they used with Spark?**

A **`Map`** stores key-value pairs (`Map("a" -> 1)`); immutable maps return a new map on `+`. **`Option`** represents a value that may be absent, `Some(x)` or `None`, which replaces `null` and forces the caller to handle the missing case (`opt.getOrElse(0)`). **Pattern matching** (`match` with `case`) deconstructs values: `case (k, v) => ...` is everywhere in Spark, for example `rdd.map { case (dept, marks) => ... }` on key-value pairs. Outer joins return `Option` values (`Some(CSE)` or `None`), so understanding `Option` is needed to read join results correctly.

**Q10. What is a Stream (LazyList) in Scala and how does it differ from a List?**

A `Stream` (called **`LazyList`** from Scala 2.13) is a sequence whose elements are **computed lazily**, only when needed, and then memoized. That allows **infinite sequences**: `Stream.from(1).map(_ * 2)` is endless, but `take(5)` computes only five elements. A `List` is **strict**: all elements exist in memory immediately. Streams suit generating sequences on demand (Fibonacci numbers, large or unbounded data) without holding everything. The idea of lazy evaluation is the same one Spark uses for transformations.

---

## UNIT III (Part 1): RDDs and Spark programming

**Q1. What is an RDD, and what do "Resilient", "Distributed" and "Dataset" mean?**

An RDD (Resilient Distributed Dataset) is Spark's fundamental data abstraction: an **immutable, partitioned collection of records** that can be processed in parallel across a cluster. **Distributed** means its data is split into partitions spread over many machines. **Resilient** means it is fault tolerant: each RDD remembers the **lineage** of transformations that produced it, so a lost partition is **recomputed** from its parents rather than restored from a replica. **Dataset** means a collection of typed records. Immutability is essential, since a transformation creates a new RDD, and so lineage stays well defined.

**Q2. What are the main features of RDDs?**

RDDs are **in-memory** (cacheable), **lazily evaluated** (transformations only build a plan), **immutable**, **partitioned** (the unit of parallelism), **fault tolerant** through lineage, **persistable** with several storage levels, and **location aware** (tasks are scheduled near their data). They also support **coarse-grained** operations (`map`, `filter`, `join`) that apply to the whole dataset, which keeps lineage small. Their drawback is that they carry **no schema** and no automatic optimizer, so for structured data DataFrames are usually better.

**Q3. In how many ways can you create an RDD?**

Three. **(1) Parallelize a collection** already in the driver: `sc.parallelize(List(1,2,3), 3)`. **(2) Read external data**: `sc.textFile("path")` (HDFS, local, S3), plus `wholeTextFiles` and Hadoop input formats. **(3) Transform an existing RDD**: any transformation returns a new RDD. Also, a DataFrame exposes `df.rdd`. For `textFile` each line becomes one record, and the number of partitions follows the input splits (or the `minPartitions` argument).

**Q4. What is the difference between a transformation and an action? What is lazy evaluation?**

A **transformation** (`map`, `filter`, `flatMap`, `reduceByKey`, `join`) takes an RDD and returns a **new RDD**; it is **lazy**, meaning Spark merely records it in the lineage graph. An **action** (`collect`, `count`, `take`, `reduce`, `saveAsTextFile`) **triggers execution** and returns a result to the driver or writes to storage. Lazy evaluation lets Spark build the full plan first, combine narrow transformations into a single pass, and avoid computing results nobody asks for. A common beginner confusion is that `rdd.map(...)` appears to "do nothing": nothing runs until an action such as `count()` is called.

**Q5. What are narrow and wide transformations? What is a shuffle?**

In a **narrow** transformation each output partition depends on **one** input partition (`map`, `filter`, `flatMap`), so no data moves between machines and a chain of them runs in a single stage. In a **wide** transformation output partitions depend on **many** input partitions (`reduceByKey`, `groupByKey`, `join`, `distinct`), so records with the same key must be brought together: this movement across the network is a **shuffle**. Shuffles write to disk and use the network, so they are the most expensive operations and mark the **stage boundaries** in the DAG. Performance tuning in Spark is largely about reducing the number and size of shuffles.

**Q6. Explain `map` vs `flatMap`, and `reduceByKey` vs `groupByKey`.**

`map` produces **exactly one** output element per input (a line stays a single string), while `flatMap` may produce **zero or many** and flattens them (a line becomes its words), which is why word count starts with `flatMap`. `groupByKey` shuffles **every value** to the reducer and builds a collection per key, which can cause out-of-memory errors on skewed keys. `reduceByKey` first **combines values within each partition** (map-side combine) and shuffles only the partial results, so far less data crosses the network. Prefer `reduceByKey`, `aggregateByKey` or `combineByKey` for aggregations, and use `groupByKey` only when you genuinely need all the values.

**Q7. What is lineage and how does Spark achieve fault tolerance?**

The lineage is the **graph of transformations** that created an RDD from its sources. If an executor dies and a partition is lost, Spark does not need a stored copy: it re-runs only the transformations required to rebuild **that partition** from its parent data (and re-fetches from the source if needed). This is cheaper than replicating all data, because RDDs are immutable and transformations deterministic. For very long lineages, or after wide dependencies, you can **checkpoint** an RDD to reliable storage, which truncates the lineage. You can inspect it with `rdd.toDebugString`.

**Q8. What are the DAG, stages and tasks?**

When an action is called, Spark converts the lineage into a **DAG (directed acyclic graph)** of operations. The **DAG scheduler** divides it into **stages** at every shuffle boundary; within a stage all narrow transformations are pipelined together. Each stage is divided into **tasks**, one per partition, which run in parallel on executors. For word count there are two stages: stage 1 (read, `flatMap`, `map`, map-side combine) and stage 2 (after the shuffle: the final reduce). Number of tasks = number of partitions, which is why partition count controls parallelism.

**Q9. What is the difference between `cache()` and `persist()`? When should you cache?**

`cache()` is shorthand for `persist(StorageLevel.MEMORY_ONLY)`. `persist()` lets you choose a **storage level**: `MEMORY_ONLY`, `MEMORY_AND_DISK` (spill what does not fit), `DISK_ONLY`, serialized variants (`_SER`, smaller but slower to read) and replicated variants (`_2`). Caching is lazy too: the data is stored when the first action runs. **Cache an RDD that is reused several times** (iterative algorithms, interactive exploration); caching a dataset used once only wastes memory. Release it with `unpersist()`, and check the **Storage** tab of the UI. If memory is tight, prefer `MEMORY_AND_DISK`, so evicted partitions are not recomputed.

**Q10. What are partitions? Explain `repartition` vs `coalesce`.**

A partition is a chunk of an RDD processed by **one task**, so partitions determine parallelism. Too few underuses the cluster; too many adds scheduling overhead (a common guide is 2 to 4 partitions per CPU core). `repartition(n)` can **increase or decrease** the number and does a **full shuffle**, producing even partitions. `coalesce(n)` only **decreases** it, merging existing partitions **without a full shuffle**, so it is cheaper, for example before writing out a small result. A **partitioner** (`HashPartitioner`, `RangePartitioner`) controls how key-value data is placed, and pre-partitioning avoids repeated shuffles in joins.

**Q11. What are shared variables? Explain broadcast variables and accumulators.**

Normally Spark copies each variable used in a task to every task. **Shared variables** are two exceptions for common patterns. A **broadcast variable** ships a **read-only** value (for example a large lookup table) to each executor **once**, instead of once per task, which saves network and memory; it is also the basis of map-side (broadcast) joins. An **accumulator** is a **write-only** variable that tasks can only **add** to, and only the driver reads: it is used for counters and sums, such as counting malformed records across all partitions. Update accumulators only inside **actions**; updates inside transformations can be double-counted when tasks are retried.

**Q12. What are the limitations of RDDs, and when would you still use them?**

RDDs have **no schema** and are opaque to Spark, so there is **no Catalyst optimization**, they use more memory through Java serialization, and they are verbose compared with SQL and DataFrames. For structured data, DataFrames/Datasets are normally faster and shorter. Use RDDs when you need **low-level control** (custom partitioning, fine control over physical placement), when processing **unstructured** data (raw text, logs) with custom parsing, or when an API only offers RDDs. A common pattern is to parse raw text with RDDs, convert to a DataFrame, then continue with SQL.

**Q13. How do you invoke the Spark shell, and how does it differ from `spark-submit`?**

`spark-shell` starts an **interactive Scala REPL** with `spark` (SparkSession) and `sc` (SparkContext) pre-created, ideal for learning and exploration; `pyspark` is the Python equivalent. You can pass options such as `--master local[4]`, `--packages ...` or `--conf ...`. `spark-submit` runs a **packaged application** (a JAR or Python script) non-interactively, with options for the cluster manager, deploy mode and resources, and is how production jobs are run. In short: the shell is for **developing and testing**, `spark-submit` is for **deploying**.

---

## UNIT III (Part 2): GraphX and feature extraction

**Q1. What is GraphX, and what is a property graph?**

GraphX is Spark's component for **graphs and graph-parallel computation**. It extends the RDD with a **directed multigraph** in which every vertex and every edge carries a user-defined **property**; this is the **property graph**. Internally a graph is a pair of RDDs, a `VertexRDD[(VertexId, VD)]` and an `EdgeRDD[Edge[ED]]`, plus an `EdgeTriplet` view that joins an edge with the properties of both its end vertices. Vertex ids are `Long`s. Because the graph is built on RDDs it is immutable and fault tolerant, and one data set can be viewed both as a graph and as tables, so you can mix SQL-style and graph operations in the same program.

**Q2. What are the features and benefits of GraphX?**

GraphX gives a **unified** system (graphs and ordinary data processing in one engine, no export between a graph database and Spark), **fast in-memory iteration** (essential for algorithms such as PageRank that repeat many times), a rich **operator set** (structural, property, join, aggregation, neighbourhood), a **Pregel-style API** for iterative message-passing algorithms, **built-in algorithms** (PageRank, connected components, triangle count, SVD++, label propagation, shortest paths), and **distributed graph partitioning** that scales to very large graphs. The trade-offs are that GraphX is **Scala/Java only** (Python users use GraphFrames) and is in maintenance mode compared with newer graph systems.

**Q3. List the main categories of GraphX operators.**

**Structural operators** change the graph's shape without changing data: `reverse`, `subgraph`, `mask`. **Property operators** change data while keeping structure: `mapVertices`, `mapEdges`, `mapTriplets`. **Join operators** bring outside data in: `joinVertices` (updates only matching vertices, same type) and `outerJoinVertices` (all vertices, can change type, unmatched get `None`). **Aggregation** uses `aggregateMessages`, and **neighbourhood** operators use `collectNeighbors`/`collectNeighborIds`. Plus degree operators (`inDegrees`, `outDegrees`, `degrees`) and algorithms. In the practical, `subgraph(vpred = age > 30)` kept 4 vertices and only the 2 edges whose both ends survived.

**Q4. Explain `aggregateMessages` and how it differs from Pregel.**

`aggregateMessages` is the **core primitive**: a `sendMsg` function runs on every edge triplet and may send a message to the source or destination vertex, and a `mergeMsg` function combines all messages arriving at the same vertex. It performs **one round**, for example counting incoming edges with `sendToDst(1)` and merging with `_ + _`. **Pregel** wraps it in a **loop**: each superstep runs a vertex program, sends messages along edges and merges them, and it stops when no more messages are sent. So `aggregateMessages` is one communication step; Pregel is the iterative framework used for shortest paths or PageRank. In the practical, Pregel computed hop distances from Alice (0, 1, 2, 3, and infinity for the unreachable component).

**Q5. Explain PageRank, Connected Components and Triangle Count, with uses.**

**PageRank** ranks vertices by importance: a vertex is important if important vertices link to it. In GraphX, `PR(v) = 0.15 + 0.85 × Σ PR(u)/outDegree(u)` over incoming neighbours `u`, where 0.85 is the damping factor; it iterates until changes fall below a tolerance. Uses: web ranking, influencer detection, fraud rings. **Connected Components** labels every vertex with the smallest id in its component, so vertices with the same label are mutually reachable (ignoring direction); uses: community and cluster detection (our example found two components). **Triangle Count** counts triangles through each vertex, measuring local clustering; uses: social-network cohesion, spam detection.

**Q6. What is the difference between `joinVertices` and `outerJoinVertices`?**

Both attach external `(VertexId, value)` data to a graph. `joinVertices` updates **only the vertices that appear** in the supplied RDD; all others keep their old attribute, and the result type must be the **same** as the original. `outerJoinVertices` runs the function on **every** vertex, passing an `Option` (`None` when there is no match), and the new attribute may have a **different type**. For instance, attaching each vertex's degree uses `outerJoinVertices(graph.degrees)` with `deg.getOrElse(0)`, because vertices without edges are missing from `degrees`.

**Q7. How does GraphX compare with GraphFrames, and can I use graphs from Python?**

GraphX works on RDDs and has **no Python API**. **GraphFrames** is a separate package built on **DataFrames**, available in Scala, Java and Python, which allows SQL-like queries, **motif finding** (pattern matching in the graph) and the same core algorithms. GraphFrames benefits from Catalyst optimization and integrates with Spark SQL, which is why it is popular for PySpark users. GraphX remains the choice for Scala programs needing low-level control or the Pregel API.

**Q8. What are feature extraction, transformation and selection?**

**Feature extraction** creates numeric features from raw data (for example turning text into TF-IDF vectors). **Feature transformation** modifies existing features (scaling, encoding categories, bucketing, normalizing). **Feature selection** chooses a useful subset (for example `ChiSqSelector`). Machine-learning algorithms need numeric features in one vector column, so these steps convert raw tables or text into that form; in Spark ML they are Transformers or Estimators that can be chained in a Pipeline. Good features often matter more than the choice of algorithm.

**Q9. Explain TF-IDF and its formula in Spark.**

TF-IDF scores how important a word is to a document within a collection. **Term Frequency (TF)** is how often the word appears in the document; **Inverse Document Frequency (IDF)** down-weights words that appear in many documents: Spark uses `IDF(t) = ln((m + 1)/(df(t) + 1))`, with `m` documents and `df(t)` documents containing `t`. The weight is `TF × IDF`. Common words ("spark" in 2 of 3 documents) get a low IDF (0.288), while distinctive words ("fast", in 1 of 3) get a high IDF (0.693). It is the standard way to turn text into features, and `CountVectorizer` or `HashingTF` produce the TF part.

**Q10. Why do we scale, encode and reduce features? Explain PCA briefly.**

**Scaling** (`StandardScaler`: `z = (x − μ)/σ`, `MinMaxScaler`: `(x − min)/(max − min)`) prevents features with large ranges from dominating distance- or gradient-based algorithms such as K-means, SVM and regularized regression. **Encoding** (`StringIndexer`, `OneHotEncoder`) converts categories to numbers without implying a false order. **PCA** (Principal Component Analysis) projects the data onto a few new axes (principal components) that capture most of the variance, reducing dimensionality and noise; the `explainedVariance` tells how much information each component keeps. Always **scale before PCA**, or the largest-valued column will dominate.

---

## UNIT IV: Spark SQL

**Q1. What are the importance and main features of Spark SQL?**

Spark SQL lets you process **structured and semi-structured data** (CSV, JSON, Parquet, Hive tables, JDBC) with either **SQL** or the **DataFrame API**. Its importance is that it brings database-style optimization to big-data processing: the **Catalyst optimizer** rewrites queries (predicate pushdown, column pruning, join reordering) and the **Tungsten** engine generates efficient code and manages memory off-heap. SQL and DataFrame code compile to the **same plan**, so performance is identical. It also integrates with the rest of Spark (RDDs, Streaming, MLlib), so a single program can mix SQL and ML. It is the default interface for most data-engineering work.

**Q2. What is the difference between RDD, DataFrame and Dataset?**

An **RDD** is a low-level collection of objects with **no schema** and no Catalyst optimization. A **DataFrame** is a distributed table with **named, typed columns** (a `Dataset[Row]`) that Catalyst can optimize; it is available in every language. A **Dataset** adds **compile-time type safety** for typed objects (Scala/Java). Use DataFrames/SQL for structured work (the default), Datasets when you want compile-time checking of fields in Scala, and RDDs when you need low-level control or have unstructured data. DataFrames are generally faster than RDDs because Spark understands their structure and can optimize.

**Q3. What are the methods to convert an RDD into a DataFrame?**

There are four common ways. **(1)** `rdd.toDF("col1", "col2")` after `import spark.implicits._` (the fastest). **(2)** `spark.createDataFrame(rdd)` (columns named `_1`, `_2`… unless renamed). **(3)** **Case class reflection**: map the RDD to case-class objects and call `toDF()`, so the field names become columns. **(4)** **Programmatic schema**: build an `RDD[Row]` and a `StructType` of `StructField`s, then `spark.createDataFrame(rowRDD, schema)`, used when the schema is only known at run time or exact control of types and nullability is needed. When parsing text, remember to convert fields with `.toInt` or the columns become strings.

**Q4. Explain temporary views and global temporary views.**

SQL cannot see a Scala DataFrame variable, so we register it under a name: `df.createOrReplaceTempView("students")`. A **temporary view** is **session-scoped** (it disappears when the SparkSession ends) and stores no copy of the data, only a named handle to the plan. A **global temporary view** (`createOrReplaceGlobalTempView`) is shared across **all sessions in the same application** and is queried with the `global_temp.` prefix. `createTempView` fails if the name exists, whereas `createOrReplaceTempView` replaces it, so prefer the latter in interactive work. `spark.sql()` returns a DataFrame and is **lazy**.

**Q5. What is the difference between `WHERE` and `HAVING`, and what is the logical order of a SQL query?**

`WHERE` filters **individual rows before grouping** and cannot use aggregate functions. `HAVING` filters **groups after aggregation** and can use aggregates, for example `HAVING AVG(marks) > 70`. The logical execution order is `FROM/JOIN → WHERE → GROUP BY → HAVING → SELECT → ORDER BY → LIMIT`, which also explains why a column alias defined in `SELECT` can be used in `ORDER BY` but not in `WHERE`. Filtering early with `WHERE` is more efficient, since it reduces the data that must be grouped.

**Q6. Explain `GROUP BY` and aggregate functions, including NULL behaviour.**

`GROUP BY` divides rows into groups with equal values in the grouping columns, and aggregate functions (`COUNT`, `SUM`, `AVG`, `MIN`, `MAX`) compute one value per group. Every column in `SELECT` must either appear in `GROUP BY` or be inside an aggregate, otherwise Spark raises `MISSING_AGGREGATION`. Aggregate functions **ignore NULLs**, except `COUNT(*)`, which counts all rows; `COUNT(col)` counts only non-null values and `COUNT(DISTINCT col)` counts unique ones. So `AVG` of a column with a NULL averages only the non-null values (82.67 instead of 62 in our example). After a LEFT JOIN, count a **right-table column** rather than `COUNT(*)`, or unmatched rows are counted as one.

**Q7. How does `ORDER BY` work, and what are its costs?**

`ORDER BY` sorts the final result by one or more columns, ascending by default (`DESC` reverses), each column with its own direction. In Spark, NULLs sort **first** in ascending order and **last** in descending order unless `NULLS FIRST/LAST` is given. A global sort requires a **range-partitioning shuffle** across the cluster, which is expensive; `LIMIT` after `ORDER BY` lets Spark use a cheaper top-N. Without `ORDER BY`, row order is **not guaranteed**. If you only need ordering inside groups, use `sortWithinPartitions` or window functions instead of a global sort.

**Q8. What are SQL wildcards? Explain `%` and `_` with the case-sensitivity caveat.**

Wildcards are special characters used with `LIKE` to match patterns. `%` matches **any number of characters** (including none) and `_` matches **exactly one**. `'A%'` is "starts with A", `'%e'` "ends with e", `'%li%'` "contains li", `'_ob'` matches "Bob", and `'D____'` matches five-letter names starting with D. In Spark SQL **`LIKE` is case-sensitive**: `'%a%'` does **not** match "Alice" because it has only a capital A. For case-insensitive matching use `LOWER(name) LIKE '%a%'` or `ILIKE`. `RLIKE` accepts full regular expressions.

**Q9. Explain the types of joins in Spark SQL and how Spark executes a join.**

`INNER` keeps matching rows; `LEFT`/`RIGHT` keep all rows of one side with NULLs where no match exists; `FULL OUTER` keeps all rows of both; `LEFT SEMI` returns left rows that **have** a match (left columns only); `LEFT ANTI` returns left rows with **no** match (ideal for "find missing records"); `CROSS` gives every combination (m × n). Physically, Spark picks a **broadcast hash join** when one side is small (the small table is sent to every executor, avoiding a shuffle), otherwise a **sort-merge join** (both sides shuffled and sorted by key). You can hint `/*+ BROADCAST(d) */`. Ambiguous columns must be qualified with aliases, and NULL keys never match.

**Q10. How does the Catalyst optimizer work?**

Catalyst turns a query into an efficient execution in four phases. **(1) Analysis:** resolve table and column names against the catalog and produce a logical plan. **(2) Logical optimization:** apply rule-based rewrites such as **predicate pushdown** (filter early), **column pruning** (read only the needed columns), constant folding and join reordering. **(3) Physical planning:** generate candidate physical plans and choose by cost (for example broadcast vs sort-merge join). **(4) Code generation** (Tungsten's whole-stage codegen) compiles the plan into tight JVM bytecode. You can see the result with `df.explain()`. Because SQL and DataFrame code share the same optimizer, they perform the same.

**Q11. How would you improve the performance of a slow Spark SQL job?**

First inspect the plan (`explain`) and the **Spark UI** to find the slow stage. Then: use **columnar formats** such as Parquet (column pruning, compression); **filter early** and select only needed columns; **broadcast** small tables in joins; tune `spark.sql.shuffle.partitions` (the default 200 is too many for small data and too few for huge data); **cache** a DataFrame reused several times; avoid Python **UDFs** when a built-in function exists (they bypass the optimizer); handle **data skew** (salting keys, adaptive query execution); and avoid unnecessary `collect()` and global sorts. Adaptive Query Execution (on by default in Spark 3) coalesces shuffle partitions and handles skewed joins automatically.

**Q12. What is the difference between schema inference and an explicit schema?**

With **inference** (`option("inferSchema","true")`) Spark scans the data to guess types, which is convenient but costs an extra pass, can guess wrongly (a numeric-looking ID may become `int`), and may differ between files. An **explicit schema** (`StructType`) states column names, types and nullability, so it is **faster, predictable and safer**, and it is mandatory for streaming file sources. Without either, a CSV read gives every column as `string`, the equivalent of forgetting `.toInt` when parsing text. In production, always provide the schema.

---

## UNIT V: Spark Streaming with Apache Kafka

**Q1. What is Apache Kafka, and why is it used in data pipelines?**

Kafka is a **distributed, fault-tolerant, high-throughput event streaming platform** that stores streams of records durably and lets many applications publish and read them in real time. In a pipeline, connecting producers directly to consumers creates M × N fragile links, loses data when a consumer is down and cannot replay history. Placing Kafka in the middle **decouples** the systems, **buffers** bursts, **stores** events (so they can be replayed), and lets many consumers read the same data independently. It was created at LinkedIn and is the standard backbone for event-driven architectures. Spark then processes what Kafka transports.

**Q2. How does Kafka differ from a traditional message queue such as RabbitMQ?**

In a traditional queue a message is usually **deleted once consumed** and the broker **pushes** to consumers; replay is hard and scaling is limited. Kafka is a **distributed commit log**: records are **retained** for a configured time (default 7 days) regardless of consumption, consumers **pull** at their own pace and keep their own offsets, so any consumer can **replay** from any offset, and many independent consumer groups can read the same topic. Throughput is much higher thanks to sequential disk writes and partitioning. Queues are well suited to task distribution and complex routing, while Kafka is suited to **event streaming, log aggregation and data pipelines**.

**Q3. Explain topic, partition and offset.**

A **topic** is a named stream of records (like a table or folder). It is divided into **partitions**, each an **ordered, append-only log**; partitions give parallelism and let a topic scale across brokers. Every record in a partition gets a sequential number called its **offset**, starting at 0, which is **unique only within that partition**. A record is therefore identified by (topic, partition, offset). A consumer's progress is simply the offset it has reached, which it can store, so it can resume after a restart or rewind to reprocess.

**Q4. What ordering does Kafka guarantee, and how do keys help?**

Kafka guarantees order **only within a single partition**, not across the whole topic. To keep related events in order, give them the **same key**: the producer's partitioner computes `hash(key) mod numberOfPartitions`, so every record with that key lands in the same partition, preserving its sequence (for example all events of customer 1001: order placed, paid, shipped). Records without a key are spread across partitions. Note that **increasing the partition count later changes the mapping** and can break per-key ordering for future records.

**Q5. What is a consumer group, and how does Kafka support both queue and publish-subscribe models?**

A consumer group is a set of consumers that share a `group.id` and cooperate to read a topic: **each partition is read by exactly one consumer of the group**, and one consumer may read several partitions. Consumers in the **same group** therefore split the records (queue behaviour, used to scale processing); **different groups** each receive **all** the records independently, with their own offsets (publish-subscribe behaviour, used to serve different applications). The maximum useful number of consumers in a group equals the number of partitions; extra consumers sit idle. When a consumer joins or fails, the group **rebalances** the partitions.

**Q6. Describe the Kafka cluster architecture: brokers, controller and ZooKeeper/KRaft.**

A **broker** is a Kafka server that stores partitions and serves producers and consumers; a **cluster** is several brokers. Clients need only a few **bootstrap servers** to discover the full metadata. One broker acts as the **controller**, detecting failures and electing partition leaders. In the classic design **ZooKeeper** (port 2181) stores cluster metadata, tracks live brokers by heartbeat and helps elect the controller (it does not store messages). **KRaft** is Kafka's built-in Raft-based replacement that removes ZooKeeper (the only mode from Kafka 4.0). Your Kafka 3.4.0 uses ZooKeeper mode by default and supports KRaft.

**Q7. Explain replication, leader, follower and ISR, and what happens when a broker fails.**

Each partition is copied to several brokers according to the **replication factor** (which cannot exceed the number of brokers). One copy is the **leader**, handling all reads and writes; the others are **followers** that replicate it. The **ISR** (in-sync replicas) is the set of replicas fully caught up with the leader. If the leader's broker fails, the **controller promotes a follower from the ISR** as the new leader, clients refresh metadata and continue, and no committed data is lost because ISR members had it. The failed broker later rejoins as a follower and catches up. Only ISR members may become leader, to guarantee no committed record is lost.

**Q8. Explain the producer `acks` setting and `min.insync.replicas`.**

`acks` defines when a send is considered successful. **`acks=0`**: do not wait (fastest, may lose data silently). **`acks=1`**: wait for the leader only (data lost if the leader dies before replicating). **`acks=all`**: wait for **all in-sync replicas** (safest, slowest). `min.insync.replicas` is the minimum ISR size required for an `acks=all` write to succeed. The common safe combination is **replication factor 3, `min.insync.replicas=2`, `acks=all`**, plus `enable.idempotence=true` to prevent duplicates caused by retries. You choose per use case: click streams can use `acks=1`, financial events need `acks=all`.

**Q9. What are the delivery semantics in Kafka, and how is exactly-once achieved?**

**At most once**: the consumer commits offsets before processing, so a crash can lose records. **At least once**: it commits after processing, so a crash causes reprocessing and possible **duplicates**. **Exactly once**: each record affects the result once. In Kafka it is achieved with the **idempotent producer** plus **transactions** (atomic writes and offset commits); in Spark Structured Streaming it is achieved with a **replayable source (Kafka) + checkpointed offsets + an idempotent or transactional sink**. In practice most pipelines run at-least-once and make processing **idempotent** (for example de-duplicating by a unique id) to get effectively-once results.

**Q10. Outline the Kafka installation on Windows and the common pitfalls.**

Install JDK 8, extract Kafka to a short path (`C:\kafka`), set `dataDir` in `zookeeper.properties` and `broker.id`, `listeners`, `log.dirs` and `zookeeper.connect` in `server.properties` using forward slashes, then **start ZooKeeper first and Kafka second**, verifying with `jps` (`QuorumPeerMain` and `Kafka`) and the ports 2181 and 9092. Stop in reverse order. Pitfalls: *"input line is too long"* (path too long), `InconsistentClusterIdException` (deleted only one of the two data folders), connection refused (broker not running), and **never delete topics on Windows**, which can crash the broker through file locking; reset the data folders instead.

**Q11. Explain the producer and consumer messaging model, including offset commits.**

A **producer** builds a record, **serializes** key and value to bytes, the partitioner selects a partition, records are **batched** per partition and sent to the leader, which acknowledges according to `acks`. A **consumer** subscribes to topics as part of a group, **pulls** batches with `poll()`, deserializes, processes, and then **commits its offset** (automatically or manually) so it can resume where it stopped; `auto.offset.reset` (`earliest`/`latest`) decides the start when no offset exists. The pull model lets a slow consumer fall behind without hurting others; the gap is the **lag** (`log-end-offset − current-offset`), the main health metric.

**Q12. What is the difference between DStreams (Spark Streaming) and Structured Streaming?**

Both use **micro-batching**. A **DStream** is a sequence of RDDs, one per batch interval, programmed with RDD-style operations, with manual state management (`updateStateByKey`), **processing-time** semantics and no optimizer; it is now **legacy**. **Structured Streaming** treats the stream as an **unbounded table** queried with DataFrame/SQL operations, is optimized by Catalyst, manages **state automatically**, supports **event-time windows and watermarks**, and provides **end-to-end exactly-once** guarantees with replayable sources and checkpoints. For new work always use Structured Streaming; DStreams are covered because they appear in the "Spark Streaming" syllabus and show the micro-batch idea clearly.

**Q13. Explain output modes, triggers, watermarks and checkpointing in Structured Streaming.**

**Output modes** decide what is written each batch: `append` (only new rows), `update` (only changed rows), `complete` (the entire result table, aggregations only); an aggregation in append mode is rejected because append writes a row once and an aggregate keeps changing. **Triggers** control when batches run (default as fast as possible, a fixed `ProcessingTime`, or `AvailableNow` to process everything and stop). A **watermark** states how late data may arrive in event time, letting Spark drop old window state and bound memory. **Checkpointing** (`checkpointLocation`) stores offsets and state, so a restarted query resumes without loss or duplication; use one checkpoint folder per query.

**Q14. How does Spark integrate with Kafka, and how are offsets handled?**

Add the connector `org.apache.spark:spark-sql-kafka-0-10_2.13:3.5.9` (the `_2.13` must match Spark's Scala version, and `3.5.9` Spark's version) and read with `spark.readStream.format("kafka")` using `kafka.bootstrap.servers` and `subscribe`. The resulting DataFrame has fixed columns `key`, `value` (**binary**), `topic`, `partition`, `offset`, `timestamp`, so you **cast** `value` to string and parse it (`split` or `from_json`). Spark creates **one read task per Kafka partition**. It manages offsets itself **in the checkpoint**, not in consumer-group commits, so `startingOffsets` applies only to the first run, and `kafka-consumer-groups` does not show Spark's lag. Limit batch size with `maxOffsetsPerTrigger`. After resetting Kafka data, delete the old checkpoints.

**Q15. What is a Kafka pipeline, and what ingestion patterns do you know?**

A pipeline moves data from sources through **ingestion**, a **transport/buffer** (Kafka), **processing** (Spark) and **storage/serving**, with monitoring. Ingestion can be **batch**, **micro-batch** or **real-time**. Patterns include: **file/batch ingestion** into a topic; **real-time event ingestion** by application producers; **log aggregation** (fan-in: many servers to one topic); **change data capture** (database changes streamed as events, typically via Kafka Connect and Debezium); **fan-out** (many readers of one topic); **filter-and-route**; **enrichment** (stream joined with a static table); the **dead-letter queue** (invalid records sent to a separate topic so the pipeline keeps running); **de-duplication**; and **replay** (reprocess history by starting a new checkpoint from `earliest`).

**Q16. What are Lambda and Kappa architectures, and how do Kafka Connect and Kafka Streams fit in?**

**Lambda** runs a **batch layer** (accurate, slow) and a **speed layer** (fast, approximate) and merges them in a serving layer, at the cost of maintaining **two code bases**. **Kappa** uses a **single streaming path** and reprocesses history by **replaying the Kafka log**, so one code base suffices; Structured Streaming's unified batch/stream API suits it. **Kafka Connect** is a framework of ready-made connectors that move data between Kafka and external systems (databases, files, cloud storage) without code, used for ingestion and CDC. **Kafka Streams** is a Java/Scala **library** for stream processing inside an application (no separate cluster), whereas Spark is a distributed engine for heavy analytics, joins and machine learning.

---

## UNIT VI: Spark ML and PySpark

**Q1. What is machine learning, and what are its main types?**

Machine learning is building programs that **learn patterns from data** to make predictions or decisions without being explicitly programmed with rules: instead of giving rules and data to get answers, we give **data and answers** and get the **rules (the model)**. The main types are **supervised** learning (labelled data: regression for numbers, classification for categories), **unsupervised** learning (no labels: clustering, dimensionality reduction, association), **semi-supervised** learning (few labels, many unlabelled) and **reinforcement** learning (an agent learns by reward and penalty). In our practicals, predicting final marks is regression, predicting pass/fail is classification, and segmenting customers is clustering.

**Q2. What is PySpark and how does it work internally?**

PySpark is the **Python API of Apache Spark**, covering DataFrames, SQL, streaming and MLlib. Your Python code runs in a driver process and sends commands to the Spark **JVM through Py4J**; DataFrame and MLlib operations then execute **inside the JVM** on the executors, so their performance is close to Scala. Only Python-specific pieces (Python UDFs, RDD lambdas) need separate Python worker processes and are slower. That is why PySpark ML scales well even though the code is Python. Typical environment issues are mismatched Python versions between driver and workers, fixed with `PYSPARK_PYTHON`.

**Q3. Explain regression vs classification, with algorithms for each.**

Regression predicts a **continuous number** ("how much?"), such as final marks; classification predicts a **category** ("which one?"), such as pass or fail. Regression algorithms in the syllabus: **Linear Regression** (also Decision Tree regressors). Classification algorithms: **Logistic Regression, Decision Tree, SVM and Naive Bayes**. Regression is judged with **RMSE, MAE and R²**; classification with the **confusion matrix, accuracy, precision, recall, F1 and AUC**. A common trap: **logistic regression is a classification algorithm** despite its name: it predicts a probability and then a class.

**Q4. What are overfitting, underfitting and the bias-variance trade-off? How do you fix them?**

**Underfitting** means the model is too simple (high bias) and performs poorly on both training and test data. **Overfitting** means it is too complex (high variance): it memorises noise, with excellent training but poor test performance (in our tree experiment, depth 8 gave 100% training and 82.9% test accuracy). The expected error decomposes as `Bias² + Variance + noise`, so we want the complexity that minimizes test error. Fixes for overfitting: **more data, a simpler model, regularization** (L1/L2, `regParam`), **limiting tree depth**, **cross-validation**, and **ensembles**. Fixes for underfitting: more features or a more flexible model. Diagnose by comparing training and test scores.

**Q5. Why do we split data into train, validation and test sets, and what is data leakage?**

The **training** set fits the model's parameters; the **validation** set (or cross-validation folds) is used to **choose hyperparameters**; the **test** set, touched only once at the end, gives an honest estimate of performance on unseen data. Without a held-out set, a model can simply memorise. **Data leakage** is when information the model should not have (test data, or something derived from the label or from the future) influences training, producing unrealistically good results: for example scaling with statistics from the whole dataset, or using `final_marks` as a feature to predict `passed`. Prevent it with Pipelines (statistics learned only from training data) and careful feature review.

**Q6. What is Spark MLlib, and how do Transformer, Estimator and Pipeline fit together?**

MLlib is Spark's scalable machine-learning library; the **DataFrame-based `spark.ml`** API is the primary one (the RDD-based `spark.mllib` is in maintenance mode). A **Transformer** has `transform()` and converts one DataFrame into another by adding columns (`VectorAssembler`, any fitted model). An **Estimator** has `fit()`, learns from data and returns a **Model**, which is itself a Transformer (`LinearRegression` → `LinearRegressionModel`). A **Pipeline** chains stages (prepare features → algorithm); fitting it returns a `PipelineModel`. Pipelines run everything in one call, learn preprocessing statistics only from training data (preventing leakage), and can be saved, loaded and tuned.

**Q7. Why is `VectorAssembler` needed, and why do we scale and encode features?**

Spark ML algorithms read all features from **one Vector column**, which `VectorAssembler` builds from the separate columns (the label stays separate). **Scaling** puts features on comparable ranges: `StandardScaler` gives `z = (x − μ)/σ`, `MinMaxScaler` gives `(x − min)/(max − min)`; it matters for K-means, SVM and regularized models because they depend on distances or penalties, not for trees or Naive Bayes. **Encoding** (`StringIndexer` then `OneHotEncoder`) converts categories into numbers without implying an order. Fit the scaler on the **training data only**, then reuse it on test data: a Pipeline does this automatically.

**Q8. Explain linear regression: model, loss, solving methods and regularization.**

Linear regression models the label as `ŷ = β₀ + β₁x₁ + … + βₖxₖ` and finds the coefficients that **minimize the sum of squared residuals** `SSE = Σ(y − ŷ)²` (least squares). It can be solved in closed form with the **normal equation** `β = (XᵀX)⁻¹Xᵀy` (for simple regression `β₁ = Sxy/Sxx`, `β₀ = ȳ − β₁x̄`), or iteratively with **gradient descent**, which Spark's L-BFGS solver does for many features. Our worked example gave `marks = 31.5 + 8.5·hours`. **Regularization** adds a penalty to prevent overfitting: Ridge (L2, `λΣβ²`) shrinks coefficients, Lasso (L1, `λΣ|β|`) can zero some out; in Spark `regParam` is λ and `elasticNetParam` is 0 for Ridge and 1 for Lasso. Assumptions: linearity, independence, constant variance of residuals, normal residuals and no multicollinearity.

**Q9. Explain logistic regression: sigmoid, odds, loss and threshold.**

It computes a linear score `z = β₀ + β·x` and passes it through the **sigmoid** `p = 1/(1 + e⁻ᶻ)` to get the **probability** of the positive class, predicting 1 when `p ≥ threshold` (default 0.5). The model assumes the **log-odds** `ln(p/(1−p))` are linear in the features, so `e^β` is the **odds ratio** (in our tiny example each extra study hour multiplied the odds of passing by 3.6). It is trained by minimizing **log-loss** `−Σ[y ln p + (1−y) ln(1−p)]`, derived from maximum likelihood, whose gradient is `(1/n)Σ(p − y)x`. Raising the threshold increases precision and lowers recall; for more than two classes it uses **softmax**. Its decision boundary is linear.

**Q10. Explain decision trees: how splits are chosen, and how overfitting is controlled.**

A decision tree splits the data recursively with feature tests, choosing at each node the split with the **highest information gain**, i.e. the largest drop in impurity. **Entropy** is `−Σ p log₂ p` (0 for pure, 1 for a 50/50 node) and **Gini** is `1 − Σp²`; gain = parent impurity − weighted child impurity. In the weather data the root is **Outlook** (gain 0.2467), with Overcast already a pure "Yes" leaf. Regression trees split by variance (SSE) reduction and predict the leaf mean. Unlimited growth memorises noise, so control it with `maxDepth`, `minInstancesPerNode`, `minInfoGain`, pruning, or use a **random forest** (many trees averaged). Trees need no scaling and give readable rules but are unstable and produce step-like predictions.

**Q11. Explain K-means: the algorithm, choosing K and limitations.**

K-means partitions unlabelled data into **K clusters**. After choosing K and initial centroids (Spark uses **k-means||**, a parallel k-means++), it repeats two steps: **assign** each point to its nearest centroid, then **recompute** each centroid as the mean of its points, until assignments stop changing. It minimizes **WCSS** (`Σ‖x − μ‖²`) and always converges, but to a **local** minimum. Choose K with the **elbow method** (the bend in WCSS vs K) and the **silhouette** `(b − a)/max(a, b)` (higher is better): our customers gave a sharp elbow and a silhouette of 0.988 at K = 4. Limitations: K must be chosen, sensitivity to initialization and outliers, an assumption of round similar-sized clusters, and **features must be scaled**.

**Q12. Explain the SVM: margin, support vectors, soft margin and kernels.**

An SVM finds the **hyperplane** `w·x + b = 0` separating two classes with the **maximum margin**, the width `2/‖w‖` of the empty road around it; minimizing `½‖w‖²` therefore maximizes the margin. The points closest to the boundary, lying on `w·x + b = ±1`, are the **support vectors**, and only they determine the hyperplane. Real data overlaps, so the **soft margin** allows violations with slack variables, minimizing `½‖w‖² + CΣξ` (equivalently **hinge loss** `max(0, 1 − y·f)` plus an L2 penalty): large C means a narrow margin and overfitting risk, small C a wide margin and underfitting risk (Spark's `regParam` acts inversely). **Kernels** allow non-linear boundaries, but Spark's `LinearSVC` offers only the linear kernel and binary classification (use `OneVsRest` for multiclass) and gives no probabilities.

**Q13. Explain Naive Bayes: Bayes' theorem, the assumption, smoothing and variants.**

Naive Bayes classifies by **Bayes' theorem**: `P(C|X) = P(X|C)·P(C)/P(X)` (posterior = likelihood × prior / evidence). It is "**naive**" because it assumes the features are **conditionally independent given the class**, so `P(X|C) = ΠP(xᵢ|C)`, which makes training just counting; the predicted class maximizes `P(C)ΠP(xᵢ|C)`. For the weather query (Sunny, Cool, High, Strong) the scores were 0.0053 (Yes) and 0.0206 (No), giving P(No) = 0.795. **Laplace smoothing** `(count + α)/(N + αk)` prevents a zero count from wiping out the product. Spark offers **multinomial** (counts, text), **Bernoulli** (0/1), **Gaussian** (continuous) and complement variants. It is very fast and excels at text, but independence is rarely exact and its probabilities can be over-confident.

**Q14. What is a confusion matrix, and how do you derive precision, recall, F1 and accuracy? When does accuracy mislead?**

A confusion matrix tabulates **actual vs predicted** classes: TP (predicted positive, really positive), TN, FP (false alarm, Type I) and FN (missed positive, Type II). **Accuracy** = (TP + TN)/all; **precision** = TP/(TP + FP) ("when I say positive, how often am I right?"); **recall** = TP/(TP + FN) ("how many real positives did I find?"); **F1** = 2PR/(P + R), a harmonic mean that is high only when both are high. **Accuracy misleads on imbalanced data**: with 1% fraud, a model that always predicts "genuine" scores 99% yet catches nothing. Choose the metric by the cost of errors: recall for cancer screening, precision for spam filtering; moving the threshold trades one for the other.

**Q15. What are the ROC curve and AUC?**

The **ROC curve** plots the **true positive rate** (recall) against the **false positive rate** for every possible threshold; a good model hugs the top-left corner and the diagonal is random guessing. **AUC** is the area under it (0.5 = random, 1.0 = perfect). It has a neat meaning: *the probability that a randomly chosen positive example gets a higher score than a randomly chosen negative one* (for scores 0.9, 0.8, 0.4 vs 0.7, 0.3, 0.2 it is 8/9 = 0.889). AUC is **threshold-independent**, so it measures ranking quality. For very rare positives, the **precision-recall AUC** is more informative. In Spark give the evaluator the **score** (`rawPrediction`/`probability`), not the hard 0/1 prediction.

**Q16. Define RMSE, MAE and R². How do you choose among them?**

**MSE** = `Σ(y − ŷ)²/n`; **RMSE** = `√MSE`, the typical error in the label's own units, which punishes large errors; **MAE** = `Σ|y − ŷ|/n`, the average absolute error, more robust to outliers; **R²** = `1 − SSE/SST`, the **fraction of variance explained** (1 perfect, 0 no better than predicting the mean, negative worse than the mean). RMSE ≥ MAE always, and a big gap signals a few large errors. Use RMSE when large errors are costly, MAE when outliers should not dominate, R² to compare across scales, and **adjusted R²** with many features. On our test set: RMSE 2.84, MAE 2.26, R² 0.929.

**Q17. What is the correlation heat map and why is it useful?**

The Pearson **correlation** `r = cov(X,Y)/(σₓσᵧ)` measures **linear** association between two variables on a unit-free scale from −1 to +1. A **correlation heat map** is a colour-coded matrix of all pairwise correlations (built in Spark with `Correlation.corr` and drawn with `seaborn.heatmap`). It shows which features relate to the **label** (hours vs final marks, r = 0.834) and which features **duplicate each other** (|r| > 0.8 signals **multicollinearity**, which makes regression coefficients unstable). Limits: it detects only linear relationships (a perfect U-curve has r = 0) and **correlation is not causation**. Spearman correlation uses ranks for monotonic relationships.

**Q18. How do you tune hyperparameters in Spark ML, and what is cross-validation?**

Build a grid with `ParamGridBuilder` (for example `regParam` ∈ {0, 0.01, 0.1}), and give it to a `CrossValidator` with an evaluator and `numFolds`. **k-fold cross-validation** splits the training data into k folds, trains on k − 1 and validates on the remaining one in turn, and averages the k scores; the combination with the best average wins and is **refit on all training data**. A 6-combination grid with 3 folds trains 18 models plus the final refit, so keep grids small or use `TrainValidationSplit` (one split, cheaper but less reliable). Tune using the training data only and evaluate on the **test set once**. Put the preprocessing inside the Pipeline so each fold learns its statistics only from its own training portion.

---

## CROSS-UNIT INTERVIEW QUESTIONS (industry level)

**Q1. Walk me through a project you did with this stack.**

*Structure your answer as: problem → architecture → your part → decisions → result → lessons.* For example: "We built a real-time retail analytics pipeline. A Python producer published sales events to **Kafka** (3 partitions, keyed by city). A **Spark Structured Streaming** job read the topic, cast and parsed the CSV, validated records (bad ones went to a **dead-letter topic**), aggregated revenue per product in complete mode with a **checkpoint**, and routed high-value orders to an alerts topic. Clean data was stored as files and analysed in batch with **Spark SQL** joins; a **K-means** pipeline segmented customers. I chose Structured Streaming over DStreams for Catalyst and exactly-once semantics, and learned to delete checkpoints after resetting Kafka." Practise this with your own details.

**Q2. How does Spark achieve fault tolerance?**

At several levels. **Data:** RDDs and DataFrames are immutable and carry **lineage**, so lost partitions are recomputed from their parents. **Tasks:** failed tasks are retried on other executors; slow tasks can be speculatively re-run. **Executors:** the cluster manager replaces failed executors. **Driver:** can be restarted with checkpointed metadata in streaming. **Streaming state:** **checkpoints** save offsets and state to reliable storage, and replayable sources like Kafka allow reprocessing from a known offset. For long lineages, explicit `checkpoint()` truncates them. The principle is *recompute rather than replicate*, which keeps the normal path fast.

**Q3. A Spark job is slow. How do you diagnose and fix it?**

Start with the **Spark UI**: find the slow **stage**, then look at task time distribution (skew shows as a few very long tasks), shuffle read/write size, spill to disk and GC time. Common causes and fixes: **too many or too few shuffle partitions** (tune `spark.sql.shuffle.partitions` or enable AQE); **data skew** (salting, broadcasting, AQE skew join); **wide transformations** where narrow ones would do (`reduceByKey` instead of `groupByKey`); **a large join** that could broadcast a small table; **no caching** of a reused dataset; **inefficient formats** (use Parquet, push down filters, select only needed columns); **Python UDFs** (replace with built-ins); and **insufficient memory/cores**. Measure after each change.

**Q4. What is data skew and how do you handle it?**

Skew means a few keys hold a disproportionate share of the data, so one task processes far more than the others and the stage waits for it. Symptoms are a handful of very long tasks and executor out-of-memory errors. Remedies: **broadcast** the small side of a join; **salt** hot keys (append a random suffix to spread them, aggregate in two steps); pre-aggregate with `reduceByKey`; filter or separate the hot keys; and use **Adaptive Query Execution**'s skew-join handling. In Kafka the same problem appears as a **hot partition** when one key dominates, so choose a well-distributed key.

**Q5. When would you choose RDDs, DataFrames or Datasets?**

Default to **DataFrames/SQL** for structured data: they are optimized by Catalyst and Tungsten and are concise. Use **Datasets** (Scala/Java) when compile-time type safety on domain objects matters. Use **RDDs** for low-level control (custom partitioners, fine placement), unstructured data with custom parsing, or legacy code. In practice many jobs start with RDDs to parse messy text, then convert to a DataFrame. In PySpark, prefer DataFrames, because Python lambdas on RDDs run in Python worker processes and are much slower than JVM-native DataFrame operations.

**Q6. How would you design a real-time fraud-detection system with Kafka, Spark and ML?**

Card transactions are produced to a **Kafka topic** keyed by card id (so each card's events stay ordered). A **Structured Streaming** job reads the topic, parses and validates events (dead-letter topic for bad ones), enriches them with **static customer data** (stream-static join), computes **windowed features** (transaction count and amount per card over the last 10 minutes, using event time and watermarks), and applies a **saved MLlib `PipelineModel`** (for example logistic regression or random forest trained offline on labelled history) with `model.transform(stream_df)`. Transactions above a probability threshold go to an **alerts topic**. Key choices: favour **recall**, handle **class imbalance** (weights, PR-AUC), keep the raw topic for replay, use checkpoints for exactly-once, monitor lag and **model drift**, and retrain periodically.

**Q7. How do you achieve end-to-end exactly-once processing?**

Combine three things: a **replayable source** (Kafka, so the same offsets can be re-read), **checkpointing** (Spark records the offsets of each batch before processing and commits afterwards, so a restart repeats or skips nothing), and an **idempotent or transactional sink** (file sink tracks committed batches; with `foreachBatch` or databases, use upserts keyed by a unique id and the `batchId`). The Kafka **sink** is only at-least-once, so downstream consumers should de-duplicate by a unique event id. Producers should use `acks=all` and idempotence. So exactly-once is a property of the **whole chain**, not of one component.

**Q8. How do you monitor a Kafka and Spark pipeline in production?**

For **Kafka**: consumer **lag** per group and partition, broker health, under-replicated partitions, disk usage and request latency. For **Spark Streaming**: the Spark UI **Structured Streaming** tab and `query.lastProgress`, comparing **`inputRowsPerSecond` with `processedRowsPerSecond`** (if processing is slower, the pipeline falls behind), batch duration vs trigger interval, state size, and executor memory/GC. Add alerts on lag growth and failed batches, log bad-record rates (dead-letter volume), and track data-quality metrics. For **ML models**: prediction distribution, accuracy against delayed labels, and feature drift.

**Q9. What are driver and executor out-of-memory errors, and how do you fix them?**

**Driver OOM** usually comes from `collect()`/`toPandas()` on large data, huge broadcast variables, or too many small files/partitions in metadata; fix by aggregating or sampling first, writing to storage instead of collecting, and raising `spark.driver.memory`. **Executor OOM** comes from skewed or oversized partitions, large shuffles, `groupByKey` collecting many values, or caching too much; fix by increasing partitions, using `reduceByKey`, spilling with `MEMORY_AND_DISK`, raising `spark.executor.memory`/overhead, and handling skew. Always look at the failing stage in the UI before changing configuration blindly.

**Q10. Why PySpark versus Scala for Spark work?**

Scala is the native language: it gives access to the newest APIs (for example GraphX and typed Datasets), compile-time type safety and no Python-to-JVM serialization for custom functions. PySpark gives access to the **Python data-science ecosystem** (pandas, numpy, scikit-learn, matplotlib), is easier to learn and dominates data-science teams. For **DataFrame and MLlib** code the performance is nearly equal, because the work runs in the JVM; the difference appears with **Python UDFs and RDD lambdas**, which are slower (use built-in functions or pandas UDFs with Arrow). Teams often use PySpark for ML and analytics and Scala for core data-engineering libraries.

**Q11. How do you ensure a machine-learning model keeps performing well in production?**

Use a **Pipeline** so training and serving share identical preprocessing; keep a **held-out test set** and a **baseline** model; version datasets, code and models; and deploy the saved `PipelineModel` for batch or streaming scoring. After deployment, **monitor** input data (drift in feature distributions), predictions and, when labels arrive later, real accuracy, precision and recall; set alerts and a **retraining** schedule. Guard against **leakage** and class imbalance, and keep the ability to roll back to a previous model version. Treat the model as software plus data: both change over time.

**Q12. What challenges did you face while learning or building this, and how did you overcome them?**

*Pick real examples and show a debugging mindset.* Examples: multi-line Scala chains only executing their first line in the shell, solved with `:paste`; Kafka failing to start because only one of the two data folders was deleted (`InconsistentClusterIdException`), solved by resetting both and the Spark checkpoints; K-means clusters coming out wrong because features were unscaled, solved with `StandardScaler` in a Pipeline; and misleading accuracy on a small test set, solved by checking a baseline, the confusion matrix and cross-validation. Interviewers value *how you diagnosed* (read the error, isolate, check the UI/logs, fix the root cause) more than the problem itself.

---

# PART C: FINAL REVISION TOOLS

## C.1 One-page master cheat sheet

```text
UNIT I    Spark = in-memory, DAG, lazy, unified (SQL/Streaming/MLlib/GraphX) ; MapReduce = disk between stages
          Driver → cluster manager → executors ; job → stage (split at shuffle) → task (per partition)
UNIT II   val (immutable) / var ; type inference ; operators are methods ; immutable List/Map vs mutable Buffer/Map
          higher-order functions ; Stream/LazyList lazy ; Option, case class, pattern matching
UNIT III  RDD = immutable, partitioned, lineage ; transformations (lazy) vs actions ; narrow vs wide (shuffle)
          reduceByKey > groupByKey ; cache/persist ; broadcast (read-only) / accumulator (add-only)
          GraphX: VertexRDD + EdgeRDD + triplet ; aggregateMessages ; Pregel ; PageRank 0.15+0.85ΣPR/out
          Features: TF-IDF = TF × ln((m+1)/(df+1)) ; StringIndexer→OneHot→Assembler→Scaler ; PCA
UNIT IV   DataFrame + Catalyst ; RDD→DF: toDF / createDataFrame / case class / Row+StructType
          createOrReplaceTempView ; WHERE (rows) vs HAVING (groups) ; LIKE % _ (case-sensitive) ; joins incl. SEMI/ANTI
UNIT V    Kafka: topic→partitions→offsets ; order only within partition ; same key → same partition
          group: same = share, different = all ; leader/follower/ISR ; acks=all + min.insync.replicas
          Start ZK → Kafka ; connector spark-sql-kafka-0-10_2.13:3.5.9 ; key/value binary → CAST
          Structured Streaming: readStream→transform→writeStream ; append/update/complete ; checkpoint ; watermark
UNIT VI   Supervised (regression/classification) vs unsupervised (clustering) ; overfit vs underfit ; CV, leakage
          Linear: min SSE ; Logistic: sigmoid + log-loss ; Tree: entropy/Gini/gain ; K-means: WCSS, elbow, silhouette
          SVM: margin 2/‖w‖, hinge ; NB: P(C)ΠP(x|C), smoothing
          Metrics: confusion matrix → precision/recall/F1 ; AUC ; RMSE/MAE/R² ; correlation heat map
```

## C.2 Exam-day checklist

| ✔ | Item |
| --- | --- |
| ☐ | `java -version`, `spark-shell` banner, `pyspark` start OK |
| ☐ | `C:\sparklabs` has `words.txt`, `students.txt`, `make_data.py` |
| ☐ | For Kafka practicals: ZooKeeper **first**, Kafka **second**, `jps` shows both |
| ☐ | Spark started with `--packages org.apache.spark:spark-sql-kafka-0-10_2.13:3.5.9` for Kafka work |
| ☐ | Fresh checkpoint folder names (or delete old ones) |
| ☐ | Multi-line chains typed with `:paste` + Ctrl + D |
| ☐ | `pip install numpy pandas matplotlib seaborn kafka-python` done |
| ☐ | Know the **one-line AIM** and the **viva pointers** of the practical you are asked |

## C.3 Five habits that impress examiners and interviewers

1. **State the aim first**, then build step by step, running each block.
2. **Explain while you type**: "this is a transformation, so it is lazy; the action `count` triggers the job."
3. **Read the output aloud** and connect it to the theory (partitions, shuffle, offsets, confusion matrix).
4. **Mention the trade-off** (reduceByKey vs groupByKey, at-least-once vs exactly-once, accuracy vs recall).
5. **Compare with a baseline and check for overfitting and leakage** in anything involving machine learning.

---

> **You now have the whole course in one place:** ten practicals (installation to a full pipeline) and about a hundred interview-grade answers. Run each practical once, speak each answer once, and you will be ready for both the lab examiner and the interviewer. Good luck, mate. 🔥
