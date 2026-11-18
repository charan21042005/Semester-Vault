# Unit V — Phase 6: Spark Streaming Foundations

> **Continuing from Phase 5.** We can produce and consume Kafka messages. Before connecting Spark to Kafka, we must learn **how Spark itself processes a stream**. In this phase we use simple sources (a socket, files, a built-in generator) so that you master the engine first. Then Phase 7 only swaps the source for Kafka.
>
> **Everything in this phase was run in a real `spark-shell`** (Spark 3.5.x, Scala). The outputs are genuine. This phase needs **no Kafka**, so you can practise it even when Kafka is not running.

---

## Phase 6 — Roadmap

| # | Topic |
| --- | --- |
| 1 | How Spark processes streams: micro-batches |
| 2 | Our test tools: a socket server in Python |
| **Part A** | **Spark Streaming (DStreams)** |
| 3 | The DStream idea |
| 4 | StreamingContext and the first word count |
| 5 | Stateless vs stateful: `updateStateByKey` |
| 6 | Window operations |
| 7 | DStream rules and limits |
| **Part B** | **Structured Streaming** |
| 8 | The unbounded-table idea |
| 9 | Your first streaming query: the `rate` source |
| 10 | Reading and writing: sources and sinks |
| 11 | Streaming word count from a socket |
| 12 | Output modes: append, update, complete |
| 13 | Triggers |
| 14 | Streaming from files |
| 15 | Managing queries |
| 16 | Event time, windows and watermarks |
| 17 | Checkpointing and fault tolerance |
| 18 | `foreachBatch` |
| 19 | DStreams vs Structured Streaming |
| 20 | Common errors |
| 21 | Practical scenarios |
| 22 | Viva questions |
| 23 | Master revision |

---

### 1. How Spark processes streams: micro-batches

Spark does not process one record at a time. It collects records for a short time (the **batch interval**) and processes that slice as a **small batch job**.

```text
Input stream ──▶ | batch 1 | batch 2 | batch 3 | batch 4 | ──▶ each batch = a tiny Spark job
                  (0–2 s)    (2–4 s)   (4–6 s)   (6–8 s)
```

| Property | Micro-batch model |
| --- | --- |
| Latency | From about 100 ms to a few seconds |
| Throughput | Very high (reuses Spark's batch engine) |
| Fault tolerance | A failed batch is simply recomputed |
| Code reuse | Same RDD/DataFrame/SQL operations as batch Spark |

> **Remember:** Spark Streaming = *a series of small batch jobs*. That is why everything you learned about RDDs and DataFrames still applies.

---

### 2. Our test tools: a socket server in Python

For practice we need a **live text source**. A **socket** (a network connection on a port) is the simplest one. On Linux people use `nc -lk 9999`, but Windows has no `nc`. We will use a tiny **Python server** instead.

Create `C:\kafka-labs\socket_server.py`:

```python
import socket

server = socket.socket()
server.bind(("localhost", 9999))
server.listen(1)
print("Waiting for Spark to connect on port 9999 ...")

conn, addr = server.accept()
print("Spark connected:", addr)

while True:
    line = input("type a line> ")
    conn.sendall((line + "\n").encode("utf-8"))
```

**How to use it**

1. Run it first in **CMD A**: `python C:\kafka-labs\socket_server.py`. It prints *Waiting...*.
2. Start the Spark query in **CMD B** (it connects to port 9999).
3. Back in CMD A, type lines (for example `hello spark hello kafka`) and press Enter. Spark processes them.
4. Stop with **Ctrl + C**.

```text
CMD A: Python server (you type)  ──▶  port 9999  ──▶  CMD B: Spark reads
```

> Start the server **before** the Spark query. If Spark starts first you get *Connection refused*.

---

## PART A — SPARK STREAMING (DStreams)

### 3. The DStream idea

A **DStream** (Discretized Stream) is Spark Streaming's core abstraction:

> **A DStream is a continuous sequence of RDDs, one RDD for each batch interval.**

```text
time:      0–2s        2–4s        4–6s        6–8s
DStream:   [RDD 1] →  [RDD 2] →  [RDD 3] →  [RDD 4] → ...
```

Every operation on a DStream (`map`, `flatMap`, `reduceByKey`...) is applied to **each RDD** in turn. So the Unit III RDD operations work again.

**Main parts**

| Part | Role |
| --- | --- |
| `StreamingContext` | The entry point (like `SparkContext`); sets the batch interval |
| Input DStream | Receives data (socket, files, Kafka) |
| Transformations | `map`, `flatMap`, `filter`, `reduceByKey`, `window` ... |
| Output operations | `print()`, `saveAsTextFiles()`, `foreachRDD()` |
| `start()` / `awaitTermination()` / `stop()` | Control the run |

---

### 4. StreamingContext and the first word count

**Step 1 — Imports**

```scala
import org.apache.spark.streaming.{StreamingContext, Seconds}
```

**Step 2 — Create a StreamingContext with a 2-second batch interval**

```scala
val ssc = new StreamingContext(sc, Seconds(2))
```

It reuses the shell's existing `sc`. The interval `Seconds(2)` means: *cut the stream into 2-second batches*.

**Step 3 — Connect to the socket**

```scala
val lines = ssc.socketTextStream("localhost", 9999)
```

`lines` is a DStream of Strings; each line typed on the server is one element.

**Step 4 — The classic word count**

```scala
val words  = lines.flatMap(_.split(" "))
val pairs  = words.map(w => (w, 1))
val counts = pairs.reduceByKey(_ + _)
counts.print()
```

This is exactly Unit III's word count, applied to a stream.

**Step 5 — Start, wait, stop**

```scala
ssc.start()
ssc.awaitTerminationOrTimeout(30000)       // run for 30 seconds
ssc.stop(stopSparkContext = false)         // keep the shell's sc alive
```

**Output** (after typing `hello spark hello kafka`, then later `kafka streams spark`)

```text
-------------------------------------------
Time: 1794969802000 ms
-------------------------------------------

-------------------------------------------
Time: 1794969806000 ms
-------------------------------------------
(hello,2)
(kafka,1)
(spark,1)

-------------------------------------------
Time: 1794969808000 ms
-------------------------------------------

-------------------------------------------
Time: 1794969812000 ms
-------------------------------------------
(kafka,1)
(spark,1)
(streams,1)
```

**Observations**

1. A block is printed **every 2 seconds**, even when no data arrived (empty batches).
2. Each batch counts **only its own data**. The second message's batch shows `(kafka,1)`, not `(kafka,2)`. It does not remember the first batch.
3. The time stamp is in milliseconds, the start of the batch.

> **Stateless!** By default every batch is independent. To remember across batches we need *state* (next section).

**Rules for the shell**

| Rule | Reason |
| --- | --- |
| Only **one** active StreamingContext per JVM | Spark limitation |
| A stopped StreamingContext **cannot be restarted** | Create a new `ssc` instead |
| Use `stopSparkContext = false` | Otherwise `sc` and `spark` die too |
| At least **2 cores** for a receiver-based source | One core receives, another processes (`local[2]`) |

---

### 5. Stateless vs stateful: `updateStateByKey`

To keep **running totals** across batches, we use a **stateful** operation. It needs a **checkpoint directory**, where Spark saves the state.

```scala
import org.apache.spark.streaming.{StreamingContext, Seconds}

val ssc = new StreamingContext(sc, Seconds(2))
ssc.checkpoint("C:/sparkdata/dstream-checkpoint")      // required for stateful operations

val lines = ssc.socketTextStream("localhost", 9999)
val pairs = lines.flatMap(_.split(" ")).map(w => (w, 1))

val running = pairs.updateStateByKey[Int](
  (newValues: Seq[Int], state: Option[Int]) => Some(newValues.sum + state.getOrElse(0))
)
running.print()

ssc.start()
ssc.awaitTerminationOrTimeout(30000)
ssc.stop(stopSparkContext = false)
```

**How the update function works**

| Parameter | Meaning |
| --- | --- |
| `newValues` | Counts for this word in the **current batch** |
| `state` | The **previous total** for this word (None the first time) |
| Return | The **new total** = previous + new |

**Output** (same two messages typed)

```text
Time: ...806000 ms
(hello,2)
(kafka,1)
(spark,1)

Time: ...812000 ms
(hello,2)         ← remembered from earlier!
(kafka,2)         ← 1 + 1
(spark,2)         ← 1 + 1
(streams,1)
```

The totals **accumulate**. `hello` stays at 2 even though it did not appear in the second message, because the state is remembered.

---

### 6. Window operations

Often we do not want "everything since the start" but **the last N seconds**. That is a **window**.

```text
batches:   | b1 | b2 | b3 | b4 | b5 | b6 | ...
window of 3 batches, sliding by 1:
           [b1 b2 b3]
                [b2 b3 b4]
                     [b3 b4 b5]
```

A window has two parameters (both multiples of the batch interval):

| Parameter | Meaning |
| --- | --- |
| **Window length** | How much history to look at (e.g. 6 s) |
| **Sliding interval** | How often to compute (e.g. every 2 s) |

```scala
val windowed = pairs.reduceByKeyAndWindow(
  (a: Int, b: Int) => a + b,   // how to combine counts
  Seconds(6),                  // window length
  Seconds(2)                   // slide interval
)
windowed.print()
```

**Behaviour observed in the test**

- While the first message is inside the last 6 seconds, the window shows `(hello,2) (kafka,1) (spark,1)`.
- Once it falls **out** of the 6-second window, those counts disappear and only the newer message remains: `(kafka,1) (spark,1) (streams,1)`.

> **Window vs state:** `updateStateByKey` = *everything since the start*. A **window** = *only the last N seconds*.

**Common DStream transformations**

| Operation | Meaning |
| --- | --- |
| `map`, `flatMap`, `filter` | Element-wise, like RDDs |
| `reduceByKey` | Combine values per key **per batch** |
| `updateStateByKey` | Running state across batches |
| `window`, `reduceByKeyAndWindow` | Compute over a sliding time window |
| `transform` | Apply any RDD operation to each batch RDD |
| `foreachRDD` | Run custom code on each batch (write to a database...) |
| `print`, `saveAsTextFiles` | Output |

---

### 7. DStream rules and limits

| Point | Detail |
| --- | --- |
| Status | **Legacy**. Still works in Spark 3.5, but gets no new features |
| Level | RDD level; no Catalyst optimizer |
| Time semantics | Based on **processing time** (when Spark received the data), not event time |
| Kafka package | `spark-streaming-kafka-0-10` (separate from the SQL one) |
| Recommendation | Use **Structured Streaming** for new work |

We learned DStreams because they are in the classic "Spark Streaming" syllabus and show the micro-batch idea clearly. **Now the modern API.**

---

## PART B — STRUCTURED STREAMING

### 8. The unbounded-table idea

Structured Streaming's big idea:

> **Treat a live stream as a table that keeps growing. A streaming query is a normal DataFrame/SQL query on that ever-growing table.**

```text
Stream:       event1   event2   event3   event4 ...
                 |        |        |        |
Input table:  ┌──────────────────────────────────┐
              │ row1                              │
              │ row2                              │   ← new rows are APPENDED
              │ row3                              │      as events arrive
              │ row4 ...                          │
              └──────────────────────────────────┘
                              |
                      your DataFrame query (select, filter, groupBy ...)
                              |
                       Result table  ──▶  written to a sink
```

**Consequence:** the code looks exactly like Spark SQL (Unit IV). The only differences are `readStream` instead of `read`, and `writeStream` instead of `write`.

| Batch Spark | Structured Streaming |
| --- | --- |
| `spark.read.format(...).load()` | `spark.readStream.format(...).load()` |
| `df.write.format(...).save()` | `df.writeStream.format(...).start()` |
| Runs once and ends | **Runs continuously** until stopped |

---

### 9. Your first streaming query: the `rate` source

Spark has a built-in test source called **`rate`** that generates numbered rows continuously. No files, sockets or Kafka needed.

```scala
import org.apache.spark.sql.functions._
import org.apache.spark.sql.streaming.Trigger

val rate = spark.readStream.format("rate").option("rowsPerSecond", 2).load()
```

**Is it a streaming DataFrame?**

```scala
rate.isStreaming
```

```text
res0: Boolean = true
```

**Its schema**

```scala
rate.printSchema()
```

```text
root
 |-- timestamp: timestamp (nullable = true)
 |-- value: long (nullable = true)
```

**Start a query: print each micro-batch to the console**

```scala
val q = rate.writeStream
  .format("console")
  .outputMode("append")
  .trigger(Trigger.ProcessingTime("2 seconds"))
  .start()
```

**Output** (after a few seconds; timestamps differ on your machine)

```text
-------------------------------------------
Batch: 0
-------------------------------------------
+---------+-----+
|timestamp|value|
+---------+-----+
+---------+-----+

-------------------------------------------
Batch: 1
-------------------------------------------
+--------------------+-----+
|           timestamp|value|
+--------------------+-----+
|2026-11-18 02:39:...|    0|
|2026-11-18 02:39:...|    2|
|2026-11-18 02:39:...|    1|
|2026-11-18 02:39:...|    3|
+--------------------+-----+

-------------------------------------------
Batch: 2
-------------------------------------------
+--------------------+-----+
|           timestamp|value|
+--------------------+-----+
|2026-11-18 02:39:...|    4|
...
```

**What we learned**

- Each **trigger** (every 2 s) produces a numbered **batch** (Batch 0, 1, 2...).
- 2 rows/second × 2 seconds = about **4 new rows per batch**.
- Batch 0 is empty (the stream had just started).

**Stop the query**

```scala
q.stop()
```

---

### 10. Reading and writing: sources and sinks

**Sources: where streaming data comes from**

| Source | Format name | Use |
| --- | --- | --- |
| Rate | `rate` | Testing: generates `timestamp` + `value` |
| Socket | `socket` | Testing: text lines from a port |
| Files | `csv`, `json`, `parquet`, `text` | New files appearing in a folder |
| **Kafka** | **`kafka`** | **Production streams (Phase 7)** |

**Sinks: where results go**

| Sink | Format name | Use |
| --- | --- | --- |
| Console | `console` | Debugging; prints batches |
| Memory | `memory` | Debugging; results queryable with SQL |
| Files | `csv`, `json`, `parquet` | Save results to disk |
| **Kafka** | **`kafka`** | **Send results to another topic (Phase 7)** |
| foreachBatch | custom code | Any destination (database...) |

**The memory sink: query a stream with SQL**

```scala
val q2 = rate.writeStream
  .format("memory")
  .queryName("ratetable")           // becomes a SQL table name
  .outputMode("append")
  .start()
```

After a few seconds:

```scala
spark.sql("SELECT COUNT(*) AS rows_so_far FROM ratetable").show()
```

```text
+-----------+
|rows_so_far|
+-----------+
|         10|
+-----------+
```

```scala
spark.sql("SELECT value FROM ratetable ORDER BY value LIMIT 3").show()
```

```text
+-----+
|value|
+-----+
|    0|
|    1|
|    2|
+-----+
```

The stream's results live in an in-memory table. We can run ordinary Unit IV SQL on it while the stream keeps growing.

```scala
q2.stop()
```

---

### 11. Streaming word count from a socket

Now the Structured Streaming version of the classic example. Start the Python socket server first (Section 2).

```scala
import org.apache.spark.sql.functions._
import org.apache.spark.sql.streaming.Trigger
import spark.implicits._

val lines = spark.readStream
  .format("socket")
  .option("host", "localhost")
  .option("port", 9999)
  .load()

lines.printSchema()
```

```text
root
 |-- value: string (nullable = true)
```

A socket source produces **one column, `value`**, holding each line.

```scala
val words  = lines.select(explode(split($"value", " ")).as("word"))
val counts = words.groupBy("word").count()

val q = counts.writeStream
  .format("console")
  .outputMode("complete")
  .trigger(Trigger.ProcessingTime("3 seconds"))
  .start()
```

**Typed on the server:** `hello spark hello kafka`, later `kafka streams spark`.

**Output**

```text
Batch: 0
+----+-----+
|word|count|
+----+-----+
+----+-----+

Batch: 1
+-----+-----+
| word|count|
+-----+-----+
|hello|    2|
|spark|    1|
|kafka|    1|
+-----+-----+

Batch: 2
+-------+-----+
|   word|count|
+-------+-----+
|  hello|    2|
|streams|    1|
|  spark|    2|
|  kafka|    2|
+-------+-----+
```

**Key observation:** unlike DStreams, the counts **accumulate automatically** (`kafka` went 1 → 2). Structured Streaming keeps the aggregation state for us, without `updateStateByKey`.

```scala
q.stop()
```

---

### 12. Output modes: append, update, complete

The **output mode** decides **what part of the result table is written** to the sink each batch.

| Mode | Writes | Works for |
| --- | --- | --- |
| **append** | Only **new rows** added since the last batch | Queries **without** aggregation (or aggregation with a watermark) |
| **update** | Only rows that **changed** since the last batch | Aggregations and non-aggregations |
| **complete** | The **entire** result table every batch | **Aggregations** only |

**Live demonstration using the file source** (sales data; we create the files in Section 14). We compute revenue per product.

```scala
val rev = sales.groupBy("product").agg(sum(col("quantity") * col("price")).alias("revenue"))
```

**Complete mode**: file 1 has Laptop (110,000) and Phone (100,000); file 2 adds a Laptop sale and a Tablet.

```text
Batch: 0
+-------+-------+
|product|revenue|
+-------+-------+
| Laptop| 110000|
|  Phone| 100000|
+-------+-------+

Batch: 1                       ← the WHOLE table again, updated
+-------+-------+
|product|revenue|
+-------+-------+
| Tablet|  45000|
| Laptop| 165000|
|  Phone| 100000|
+-------+-------+
```

**Update mode**: a new file adds a Phone sale. Only the **changed** row is printed:

```text
Batch: 1
+-------+-------+
|product|revenue|
+-------+-------+
|  Phone| 140000|
+-------+-------+
```

(Laptop and Tablet did not change, so they are not shown.)

**Append mode with an aggregation is an error**

```scala
rev.writeStream.format("console").outputMode("append").start()
```

```text
AnalysisException: Append output mode not supported when there are streaming aggregations
on streaming DataFrames/DataSets without watermark
```

**Why?** In append mode, a row is written once and **never changed**. But an aggregate such as a sum keeps changing as more data arrives, so Spark cannot append it. (With a **watermark**, Spark knows when a window is final; see Section 16.)

**Which mode when?**

```text
Plain select/filter/map (no aggregation) → append
Aggregation, want the full report        → complete
Aggregation, want only changes           → update
Aggregation with window + watermark      → append (final results) or update
```

---

### 13. Triggers

A **trigger** controls **when** a micro-batch runs.

| Trigger | Code | Behaviour |
| --- | --- | --- |
| Default | *(nothing)* | Start the next batch as soon as the previous finishes |
| Fixed interval | `Trigger.ProcessingTime("5 seconds")` | One batch every 5 s |
| Process everything once | `Trigger.AvailableNow()` | Process all available data (in several batches), then **stop** |
| Once (older) | `Trigger.Once()` | One batch then stop (deprecated in favour of `AvailableNow`) |

```scala
.trigger(Trigger.ProcessingTime("5 seconds"))
```

**Choosing**

| Need | Trigger |
| --- | --- |
| Quick interactive demo | `ProcessingTime("2 seconds")` |
| Lower CPU usage on a laptop | `ProcessingTime("10 seconds")` |
| Treat a stream like a scheduled batch job | `AvailableNow` |

---

### 14. Streaming from files

A file source watches a **folder**; every **new file** that appears is processed. This is the easiest realistic stream with no extra tools.

**Step 1 — Create the folder**

```powershell
mkdir C:\sparkdata\stream-in
```

**Step 2 — Define the schema (mandatory for file streams)**

```scala
import org.apache.spark.sql.types._

val schema = new StructType()
  .add("order_id", IntegerType)
  .add("product", StringType)
  .add("city", StringType)
  .add("quantity", IntegerType)
  .add("price", IntegerType)
```

> Streaming file sources **need an explicit schema**, because Spark cannot scan a stream in advance. (Inference can be switched on with `spark.sql.streaming.schemaInference=true`, but is not recommended.)

**Step 3 — Create the streaming DataFrame**

```scala
val sales = spark.readStream
  .schema(schema)
  .option("header", "false")
  .csv("C:/sparkdata/stream-in")

sales.isStreaming
```

```text
res0: Boolean = true
```

**Step 4 — Define the query and start it**

```scala
val rev = sales.groupBy("product")
  .agg(sum(col("quantity") * col("price")).alias("revenue"))

val q = rev.writeStream
  .format("console")
  .outputMode("complete")
  .trigger(Trigger.ProcessingTime("3 seconds"))
  .start()
```

**Step 5 — Drop files into the folder** (while the query runs)

`f1.csv`

```text
1,Laptop,Delhi,2,55000
2,Phone,Mumbai,5,20000
```

`f2.csv`

```text
3,Laptop,Mumbai,1,55000
4,Tablet,Delhi,3,15000
```

Copy them in one after another with Explorer (or `copy`). Spark picks each up in the next batch. The output is the complete-mode table shown in Section 12.

**Useful file-source options**

| Option | Meaning |
| --- | --- |
| `maxFilesPerTrigger` | Limit the number of files per batch |
| `header` | CSV has a header row |
| `latestFirst` | Process newest files first |

> **Important:** a file must appear in the folder **completely**. Copy/move finished files rather than writing them slowly in place. Files already in the folder when the query starts are also processed.

---

### 15. Managing queries

`start()` returns a **StreamingQuery** object. Use it to monitor and control the stream.

```scala
val q = rate.writeStream.format("console").start()
```

| Command | Meaning |
| --- | --- |
| `q.isActive` | Is it still running? |
| `q.status` | Current status message |
| `q.lastProgress` | Statistics of the last batch (rows, speed) |
| `q.explain()` | The execution plan |
| `q.stop()` | Stop the query |
| `q.awaitTermination()` | Block until stopped (use in scripts) |
| `q.awaitTermination(10000)` | Wait at most 10 s |
| `q.id`, `q.name`, `q.runId` | Identity |
| `spark.streams.active` | All running queries |

```scala
q.status
```

```text
{
  "message" : "Waiting for next trigger",
  "isDataAvailable" : true,
  "isTriggerActive" : false
}
```

```scala
q.isActive     // true
q.stop()
q.isActive     // false
```

```scala
spark.streams.active.map(_.name)      // names of all running queries
```

**You can run more than one query at a time.** Always stop test queries when you finish, otherwise they keep consuming resources and printing.

**Streaming DataFrame restrictions**

| Operation | Allowed? |
| --- | --- |
| `show()`, `collect()`, `count()` directly | ❌ Not allowed. Use `writeStream` |
| `select`, `filter`, `groupBy`, joins | ✅ Allowed |
| `orderBy` | ❌ Only after aggregation in complete mode |
| `writeStream` on a **non-streaming** DataFrame | ❌ Error |

```scala
val staticDF = spark.range(3).toDF("id")
staticDF.isStreaming                               // false
staticDF.writeStream.format("console").start()
```

```text
AnalysisException: [WRITE_STREAM_NOT_ALLOWED] `writeStream` can be called only on
streaming Dataset/DataFrame.
```

---

### 16. Event time, windows and watermarks

**Two kinds of time**

| Time | Meaning |
| --- | --- |
| **Processing time** | When Spark **receives** the event |
| **Event time** | When the event **actually happened** (a column in the data) |

Network delays can make events arrive late or out of order, so real analytics use **event time**.

**Window aggregation by event time**

Data file `a.csv` with columns `event_time,city,amount`:

```text
2026-11-18 10:01:00,Delhi,100
2026-11-18 10:04:00,Delhi,50
2026-11-18 10:12:00,Delhi,70
2026-11-18 10:03:00,Mumbai,30
```

```scala
val sch = new StructType()
  .add("event_time", StringType).add("city", StringType).add("amount", IntegerType)

val ev = spark.readStream.schema(sch).csv("C:/sparkdata/stream-in2")
  .withColumn("ts", to_timestamp($"event_time"))

val agg = ev.groupBy(window($"ts", "10 minutes"), $"city")
  .agg(sum("amount").as("total"))
  .select($"window.start".as("win_start"), $"window.end".as("win_end"), $"city", $"total")
  .orderBy("win_start", "city")

val q = agg.writeStream.format("console").outputMode("complete")
  .option("truncate", "false").start()
```

**Output**

```text
+-------------------+-------------------+------+-----+
|win_start          |win_end            |city  |total|
+-------------------+-------------------+------+-----+
|2026-11-18 10:00:00|2026-11-18 10:10:00|Delhi |150  |
|2026-11-18 10:00:00|2026-11-18 10:10:00|Mumbai|30   |
|2026-11-18 10:10:00|2026-11-18 10:20:00|Delhi |70   |
+-------------------+-------------------+------+-----+
```

**Reading it**

- `window($"ts", "10 minutes")` creates **10-minute buckets**: 10:00–10:10, 10:10–10:20.
- Delhi's 10:01 and 10:04 sales (100 + 50) fall in the first bucket → **150**.
- Delhi's 10:12 sale (70) falls in the second bucket.
- The `window` column is a struct with `start` and `end`.

**Sliding windows:** `window($"ts", "10 minutes", "5 minutes")` = 10-minute windows computed every 5 minutes (windows overlap).

**Watermark: how late is too late?**

Late events could update old windows forever, forcing Spark to keep state indefinitely. A **watermark** says: *"ignore events more than X older than the newest event seen"*, so old state can be dropped.

```scala
val win = rate
  .withWatermark("timestamp", "10 seconds")
  .groupBy(window($"timestamp", "5 seconds"))
  .count()

win.printSchema()
```

```text
root
 |-- window: struct (nullable = false)
 |    |-- start: timestamp (nullable = true)
 |    |-- end: timestamp (nullable = true)
 |-- count: long (nullable = false)
```

| Concept | Meaning |
| --- | --- |
| Watermark | Threshold = max event time seen − allowed lateness |
| Effect | State of windows older than the watermark is dropped; very late events are discarded |
| Benefit | Bounded memory; makes **append** mode possible for windowed aggregations |

In the test, a query with a 5-second window on the `rate` stream wrote window rows with width `INTERVAL '0 00:00:05' DAY TO SECOND` (5 seconds) into the memory sink, confirming that the windows were built correctly.

---

### 17. Checkpointing and fault tolerance

What if the query or the machine crashes? Without extra information, Spark would not know **what it had already processed**.

A **checkpoint location** is a folder where Spark saves the query's **progress and state**.

```scala
val q = win.writeStream
  .format("memory").queryName("w")
  .outputMode("update")
  .option("checkpointLocation", "C:/sparkdata/checkpoints/q1")
  .start()
```

After the query ran, the checkpoint folder contained:

```text
C:\sparkdata\checkpoints\q1\
   ├── metadata      ← query id
   ├── offsets       ← what each batch intended to read
   ├── commits       ← which batches finished
   ├── sources       ← source information
   └── state         ← aggregation state (counts, sums...)
```

**What a checkpoint gives us**

| Benefit | Explanation |
| --- | --- |
| **Recovery** | After a crash, the restarted query continues from the last completed batch |
| **No duplicates / no gaps** | Combined with a replayable source (Kafka) and an idempotent sink → exactly-once results |
| **State restore** | Aggregations resume with their previous totals |

**Rules**

1. Use a **separate checkpoint folder per query**.
2. Never reuse an old checkpoint folder for a **different** query (it may refuse to start or resume wrongly).
3. For a *fresh start*, delete the folder or choose a new one.
4. Without a checkpoint, the file-source demo in Section 12 reprocessed **all** files when we restarted it. With a checkpoint it would have continued from where it stopped.

> **Viva line:** Checkpointing stores offsets and state so that a streaming query can recover from failures and provide end-to-end fault tolerance.

---

### 18. `foreachBatch`

Sometimes the sink we need is not built in (a database, a REST API, two sinks at once). `foreachBatch` gives us **each micro-batch as an ordinary DataFrame**, so we can write any batch code.

```scala
val q = rate.writeStream
  .trigger(Trigger.ProcessingTime("2 seconds"))
  .foreachBatch { (batchDF: org.apache.spark.sql.DataFrame, batchId: Long) =>
    println(s"batch $batchId rows=" + batchDF.count())
  }
  .start()
```

**Output**

```text
batch 0 rows=0
batch 1 rows=10
batch 2 rows=10
```

**What the two parameters are**

| Parameter | Meaning |
| --- | --- |
| `batchDF` | The micro-batch as a normal **static** DataFrame |
| `batchId` | Batch number (use it to make writes idempotent) |

Inside the function you can use everything from Unit IV: `batchDF.write.csv(...)`, `batchDF.createOrReplaceTempView("t")` + `spark.sql(...)`, JDBC writes, and so on.

---

### 19. DStreams vs Structured Streaming

| | DStreams | Structured Streaming |
| --- | --- | --- |
| Abstraction | Sequence of RDDs | Unbounded DataFrame/table |
| API | `map`, `reduceByKey` | `select`, `groupBy`, SQL |
| Optimizer | None | Catalyst |
| State | Manual (`updateStateByKey`) | Automatic for aggregations |
| Time | Processing time | **Event time** + watermarks |
| Fault tolerance | Checkpoint + WAL | Checkpoint (offsets + state) |
| End-to-end exactly-once | Hard | Built in (replayable source + idempotent sink) |
| Entry point | `StreamingContext` | `SparkSession` |
| Status | Legacy | **Recommended** |

---

### 20. Common errors

| Symptom | Cause | Fix |
| --- | --- | --- |
| `Connection refused` on socket | Server not started yet | Start the Python server first |
| `Queries with streaming sources must be executed with writeStream.start()` | Called `show()`/`count()` on a stream | Use `writeStream` |
| `Append output mode not supported ... streaming aggregations` | Aggregation + append | Use `complete`/`update` or add a watermark |
| `Complete output mode not supported when there are no streaming aggregations` | `complete` without aggregation | Use `append` |
| File stream: `Schema must be specified` | No schema given | Add `.schema(schema)` |
| Only one `StreamingContext` can be active | Old `ssc` still running | `ssc.stop(false)` first |
| `ssc` cannot be restarted | Stopped contexts are final | Create a new `StreamingContext` |
| Shell `sc`/`spark` died after `ssc.stop()` | Stopped the SparkContext too | `ssc.stop(stopSparkContext = false)` |
| Nothing printed with no data | Normal; empty batches | Send data / check source |
| Old results replayed on restart | No checkpoint | Add `checkpointLocation` |
| Checkpoint error after changing the query | Reused folder for a different query | Use a new checkpoint folder |
| Output floods the console | Fast trigger, big batches | Use a larger trigger interval |
| Query keeps running after the test | Forgot `stop()` | `spark.streams.active.foreach(_.stop())` |

---

### 21. Practical scenarios — Phase 6 practice

1. **DStream word count** from the Python socket server (2-second batches).
2. Add `updateStateByKey` so counts **accumulate**.
3. Add a **window** of 10 seconds sliding every 4 seconds.
4. Start a `rate` source query to the console; read the batch numbers.
5. Write the `rate` stream to the **memory sink** and query it with `spark.sql`.
6. **Structured Streaming word count** from the socket in `complete` mode.
7. Run the same word count in `update` mode and **explain the difference** in output.
8. Show the **append-mode error** for an aggregation and explain it.
9. Create a **file-source** stream of sales CSV files and compute revenue per product.
10. Add a **checkpoint** and restart the query; observe that old files are not reprocessed.
11. Compute 10-minute **event-time windows** from a CSV stream.
12. Use **`foreachBatch`** to print the number of rows in each batch.
13. Use `q.status`, `q.lastProgress` and `spark.streams.active`.
14. Stop all queries with `spark.streams.active.foreach(_.stop())`.

---

### 22. Phase 6 — viva questions and answers

#### Q1. What is Spark Streaming?

Spark's extension for processing live data streams by dividing them into small micro-batches processed by the Spark engine.

#### Q2. What is micro-batch processing?

Cutting a stream into small time slices (batches) and processing each slice as a small batch job.

#### Q3. What is a DStream?

A Discretized Stream: a continuous sequence of RDDs, one for each batch interval.

#### Q4. What is a `StreamingContext` and what is the batch interval?

The entry point for DStream programs. The batch interval, for example `Seconds(2)`, defines how long data is collected for each batch.

#### Q5. How do you keep a running count across batches in DStreams?

With `updateStateByKey` (needs a checkpoint directory) or by using a window.

#### Q6. What is a window operation?

Computation over the data of the last N seconds, sliding every M seconds. Both are multiples of the batch interval.

#### Q7. What is Structured Streaming?

Spark's modern streaming engine in which a live stream is treated as an unbounded table that is queried with DataFrame/SQL operations.

#### Q8. How does Structured Streaming differ from DStreams?

It uses DataFrames and Catalyst instead of RDDs, supports event-time windows and watermarks, keeps state automatically and offers end-to-end exactly-once guarantees.

#### Q9. What are the main streaming sources?

Rate, socket, files and Kafka.

#### Q10. What are the main sinks?

Console, memory, files, Kafka and `foreachBatch`.

#### Q11. Name the output modes.

Append, update and complete.

#### Q12. Why is append mode not allowed for a plain aggregation?

Aggregates keep changing as new data arrives, but append mode writes a row only once and never changes it.

#### Q13. When do we use complete mode?

When we need the entire aggregated result written every batch, for example a running word-count table.

#### Q14. What is a trigger?

A setting that controls when micro-batches run: as fast as possible, at a fixed interval, or once with `AvailableNow`.

#### Q15. What is checkpointing in Structured Streaming?

Saving a query's progress (offsets) and state to a folder so that it can recover after a failure and continue without loss or duplication.

#### Q16. What is event time and how is it different from processing time?

Event time is when the event actually occurred (stored in the data); processing time is when Spark received it.

#### Q17. What is a watermark?

A threshold on how late data may arrive. Data older than the watermark is dropped, so Spark can discard old state and limit memory.

#### Q18. What does `foreachBatch` do?

It gives each micro-batch as a normal DataFrame, so any batch code or custom sink can be used.

#### Q19. Why does a file stream need an explicit schema?

Because Spark cannot scan an unbounded stream beforehand to infer the types.

#### Q20. Why can't we call `show()` on a streaming DataFrame?

A streaming DataFrame has no finite content to show. Results must be produced by a running query started with `writeStream.start()`.

#### Q21. How do you stop a streaming query?

`query.stop()`, or `spark.streams.active.foreach(_.stop())` for all of them.

#### Q22. What does the `rate` source produce?

A stream with columns `timestamp` and `value`, generating a configurable number of rows per second. It is used for testing.

---

### 23. Phase 6 — final master code revision

```scala
// ---------- DSTREAM ----------
import org.apache.spark.streaming.{StreamingContext, Seconds}
val ssc = new StreamingContext(sc, Seconds(2))
val lines = ssc.socketTextStream("localhost", 9999)
val counts = lines.flatMap(_.split(" ")).map((_, 1)).reduceByKey(_ + _)
counts.print()
ssc.start(); ssc.awaitTerminationOrTimeout(30000); ssc.stop(stopSparkContext = false)

// ---------- STRUCTURED STREAMING ----------
import org.apache.spark.sql.functions._
import org.apache.spark.sql.streaming.Trigger
import spark.implicits._

// 1. source
val lines2 = spark.readStream.format("socket").option("host","localhost").option("port",9999).load()

// 2. transform (ordinary DataFrame code)
val counts2 = lines2.select(explode(split($"value"," ")).as("word")).groupBy("word").count()

// 3. sink + start
val q = counts2.writeStream
  .format("console")
  .outputMode("complete")
  .trigger(Trigger.ProcessingTime("3 seconds"))
  .option("checkpointLocation", "C:/sparkdata/checkpoints/wc")
  .start()

// 4. manage
q.status
q.stop()

// file source
val sales = spark.readStream.schema(schema).csv("C:/sparkdata/stream-in")

// event-time window
df.withWatermark("ts", "10 minutes").groupBy(window($"ts", "10 minutes"), $"city").count()

// foreachBatch
df.writeStream.foreachBatch { (b: org.apache.spark.sql.DataFrame, id: Long) => b.show() }.start()
```

**Structured Streaming recipe: always the same 4 steps**

```text
readStream  →  transform (select/filter/groupBy/join)  →  writeStream  →  start()
(source)                                                   (sink+mode+trigger+checkpoint)
```

**Quick-recall card**

| Need | Syntax |
| --- | --- |
| Streaming source | `spark.readStream.format("...").load()` |
| Is it streaming? | `df.isStreaming` |
| Start a sink | `df.writeStream.format("console").outputMode("append").start()` |
| Output modes | `append`, `update`, `complete` |
| Trigger | `.trigger(Trigger.ProcessingTime("5 seconds"))` |
| Checkpoint | `.option("checkpointLocation", "path")` |
| Event-time window | `window($"ts", "10 minutes")` |
| Watermark | `.withWatermark("ts", "10 minutes")` |
| Custom sink | `.foreachBatch { (df, id) => ... }` |
| Stop | `q.stop()` |

**Phase 6 completed**

---

> **Next → Phase 7: Integration of Kafka with Spark** (the connector, reading Kafka with `readStream`, the Kafka DataFrame schema, parsing CSV/JSON messages, writing results back to Kafka, offsets, checkpoints and troubleshooting). File: `7_Unit5_Kafka_Spark_Integration.md`
