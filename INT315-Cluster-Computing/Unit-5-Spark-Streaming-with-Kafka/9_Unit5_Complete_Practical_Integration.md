# Unit V — Phase 9: Complete Practical Integration

> **Continuing from Phase 8.** Every piece is now in your hands: Kafka basics, installation, producers and consumers, Structured Streaming, the Kafka connector, and pipeline patterns. In this phase we **combine everything** into complete, exam-ready projects, give you a lab routine you can follow under pressure, and train the *question → pattern* method for unseen practicals.
>
> **What is verified:** the parse / validate / aggregate / alert logic used in every project below was run in a real `spark-shell`. The final tables shown are the **exact results** after the sample events have been processed. The Kafka connection lines follow official Kafka 3.4.0 / Spark 3.5 syntax. In a streaming run you will see the same numbers appear **gradually**, batch by batch.
>
> **Pasting tip (Phase 6, Section 2b):** enter every multi-line block in `:paste` mode (type `:paste`, paste, press **Ctrl + D**).

---

## Phase 9 — Roadmap

| # | Topic |
| --- | --- |
| 1 | The lab layout: five windows |
| 2 | Pre-flight checklist |
| 3 | Project A — Live Sales Dashboard with alerts |
| 4 | Project B — Student Marks Stream |
| 5 | Project C — Word Count over Kafka |
| 6 | Project D — File → Kafka → Spark batch analysis |
| 7 | Practical exam routine |
| 8 | The "question → pattern" method |
| 9 | Ten exam-style scenarios |
| 10 | Troubleshooting master table |
| 11 | Shutdown and cleanup |
| 12 | Master template |

---

### 1. The lab layout: five windows

Kafka + Spark practicals need several windows open at once. Always use the same layout:

```text
┌────────────────────┬────────────────────┬────────────────────┐
│ W1  ZooKeeper      │ W2  Kafka broker   │ W3  Commands       │
│ (leave running)    │ (leave running)    │ topics, producers  │
├────────────────────┴───────┬────────────┴────────────────────┤
│ W4  spark-shell            │ W5  Console consumer            │
│ (--packages connector)     │ (watch an output topic)         │
└────────────────────────────┴─────────────────────────────────┘
```

| Window | Command |
| --- | --- |
| **W1** | `cd C:\kafka` → `bin\windows\zookeeper-server-start.bat config\zookeeper.properties` |
| **W2** | `cd C:\kafka` → `bin\windows\kafka-server-start.bat config\server.properties` |
| **W3** | topic commands, console producer or Python scripts |
| **W4** | `spark-shell --packages org.apache.spark:spark-sql-kafka-0-10_2.13:3.5.9 --conf spark.sql.shuffle.partitions=4` |
| **W5** | `kafka-console-consumer.bat ...` (to show results in an output topic) |

---

### 2. Pre-flight checklist

Do this **every time**, before any project:

| ✔ | Check | Command / sign |
| --- | --- | --- |
| ☐ | ZooKeeper up | W1: `binding to port ... 2181` |
| ☐ | Kafka up | W2: `[KafkaServer id=0] started` |
| ☐ | Both processes | `jps` → `QuorumPeerMain`, `Kafka` |
| ☐ | Topics exist | `kafka-topics.bat --list --bootstrap-server localhost:9092` |
| ☐ | Spark has connector | `spark.readStream.format("kafka")` works with no error |
| ☐ | Fresh checkpoints | Use **new** folder names (or delete `C:\sparkdata\checkpoints`) |
| ☐ | Folder for scripts | `C:\kafka-labs` has `sales_generator.py`, etc. |

> **After resetting Kafka's data folders, also delete `C:\sparkdata\checkpoints`.** Old checkpoints remember offsets that no longer exist.

---

### 3. Project A — Live Sales Dashboard with alerts

**Goal.** Sales events stream into Kafka. Spark must:

1. **Validate** each message (bad ones go to a dead-letter topic).
2. Show **revenue per product** continuously.
3. Send **big orders (revenue ≥ 100,000)** to a topic `sales-alerts`.

**Patterns used:** real-time ingestion, dead-letter queue, aggregation, filter and route, fan-out.

```text
 producer ─▶ [sales] ─▶ Spark ──┬─ valid ──▶ revenue per product (console)
                                │          └▶ revenue ≥ 100000 ──▶ [sales-alerts] ─▶ W5
                                └─ invalid ──▶ [sales-dlq]
```

#### Step 1 — Create the topics (W3)

```powershell
kafka-topics.bat --create --topic sales --bootstrap-server localhost:9092 --partitions 3 --replication-factor 1
kafka-topics.bat --create --topic sales-alerts --bootstrap-server localhost:9092 --partitions 1 --replication-factor 1
kafka-topics.bat --create --topic sales-dlq --bootstrap-server localhost:9092 --partitions 1 --replication-factor 1
```

#### Step 2 — Start Spark (W4)

```powershell
spark-shell --packages org.apache.spark:spark-sql-kafka-0-10_2.13:3.5.9 --conf spark.sql.shuffle.partitions=4
```

#### Step 3 — Read, parse and validate (W4)

```scala
import org.apache.spark.sql.functions._
import org.apache.spark.sql.streaming.Trigger
import spark.implicits._
```

`:paste` this block:

```scala
val raw = spark.readStream
  .format("kafka")
  .option("kafka.bootstrap.servers", "localhost:9092")
  .option("subscribe", "sales")
  .option("startingOffsets", "earliest")
  .load()

val parts = split($"value".cast("string"), ",")

val all = raw.select(
  $"value".cast("string").as("raw"),
  parts.getItem(0).cast("int").as("order_id"),
  parts.getItem(1).as("product"),
  parts.getItem(2).as("city"),
  parts.getItem(3).cast("int").as("quantity"),
  parts.getItem(4).cast("int").as("price"))

val good = all
  .filter($"order_id".isNotNull && $"quantity".isNotNull && $"price".isNotNull)
  .drop("raw")

val bad = all
  .filter($"order_id".isNull || $"quantity".isNull || $"price".isNull)
  .select($"raw".as("value"))
```

**How it works.** The CSV is split into fields and cast to numbers. If a field cannot be converted (`"bad,record"`), the cast gives NULL. Records with a NULL in a required field are **bad**.

#### Step 4 — Query 1: revenue per product (console)

`:paste`:

```scala
val revenue = good
  .groupBy("product")
  .agg(sum($"quantity" * $"price").as("revenue"))

val q1 = revenue.writeStream
  .format("console")
  .outputMode("complete")
  .trigger(Trigger.ProcessingTime("5 seconds"))
  .option("checkpointLocation", "C:/sparkdata/checkpoints/a_revenue")
  .start()
```

#### Step 5 — Query 2: alerts to Kafka

`:paste`:

```scala
val alerts = good
  .withColumn("revenue", $"quantity" * $"price")
  .filter($"revenue" >= 100000)
  .select(
    $"city".as("key"),
    to_json(struct($"order_id", $"product", $"city", $"revenue")).as("value"))

val q2 = alerts.writeStream
  .format("kafka")
  .option("kafka.bootstrap.servers", "localhost:9092")
  .option("topic", "sales-alerts")
  .option("checkpointLocation", "C:/sparkdata/checkpoints/a_alerts")
  .start()
```

#### Step 6 — Query 3: dead-letter queue

`:paste`:

```scala
val q3 = bad.writeStream
  .format("kafka")
  .option("kafka.bootstrap.servers", "localhost:9092")
  .option("topic", "sales-dlq")
  .option("checkpointLocation", "C:/sparkdata/checkpoints/a_dlq")
  .start()
```

#### Step 7 — Watch the alert topic (W5)

```powershell
kafka-console-consumer.bat --topic sales-alerts --from-beginning --bootstrap-server localhost:9092 --property print.key=true
```

#### Step 8 — Send events (W3)

```powershell
kafka-console-producer.bat --topic sales --bootstrap-server localhost:9092
```

Type these eight lines (the last one is deliberately bad):

```text
1,Laptop,Delhi,2,55000
2,Phone,Mumbai,5,20000
3,Tablet,Pune,3,15000
4,Laptop,Mumbai,1,55000
5,Phone,Delhi,2,20000
6,Tablet,Delhi,4,15000
7,Laptop,Pune,3,55000
bad,record
```

(Or run `python C:\kafka-labs\sales_generator.py` for an endless random stream.)

#### Step 9 — Expected results ✅

**Revenue per product** (W4, after all events are processed):

```text
+-------+-------+
|product|revenue|
+-------+-------+
| Laptop| 330000|
|  Phone| 140000|
| Tablet| 105000|
+-------+-------+
```

Check: Laptop = 2×55000 + 1×55000 + 3×55000 = 330,000 ✔

**Alerts topic** (W5): three orders have revenue ≥ 100,000:

```text
+--------+-------+------+-------+
|order_id|product|  city|revenue|
+--------+-------+------+-------+
|       1| Laptop| Delhi| 110000|
|       2|  Phone|Mumbai| 100000|
|       7| Laptop|  Pune| 165000|
+--------+-------+------+-------+
```

In W5 you will see them as keyed JSON messages, for example:

```text
Delhi   {"order_id":1,"product":"Laptop","city":"Delhi","revenue":110000}
Mumbai  {"order_id":2,"product":"Phone","city":"Mumbai","revenue":100000}
Pune    {"order_id":7,"product":"Laptop","city":"Pune","revenue":165000}
```

**Dead-letter topic:** read it to confirm the bad record was captured:

```powershell
kafka-console-consumer.bat --topic sales-dlq --from-beginning --bootstrap-server localhost:9092
```

```text
bad,record
```

**What to say to the examiner:** *"Valid events are aggregated; large orders are routed to an alerts topic; malformed events are preserved in a dead-letter topic. All three queries read the same topic independently, each with its own checkpoint."*

#### Step 10 — Stop (W4)

```scala
spark.streams.active.foreach(_.stop())
```

---

### 4. Project B — Student Marks Stream

**Goal.** Student results arrive as CSV lines (same format as Unit IV): `student_id,name,department,age,marks`. Spark must show a live **department summary**, flag **failures** (marks < 40) and count **grades**.

**Patterns used:** real-time ingestion, parsing, aggregation, filter.

#### Step 1 — Topic (W3)

```powershell
kafka-topics.bat --create --topic marks --bootstrap-server localhost:9092 --partitions 2 --replication-factor 1
```

#### Step 2 — Parse the stream (W4, `:paste`)

```scala
val raw = spark.readStream
  .format("kafka")
  .option("kafka.bootstrap.servers", "localhost:9092")
  .option("subscribe", "marks")
  .option("startingOffsets", "earliest")
  .load()

val p = split($"value".cast("string"), ",")

val students = raw.select(
  p.getItem(0).cast("int").as("student_id"),
  p.getItem(1).as("name"),
  p.getItem(2).as("department"),
  p.getItem(3).cast("int").as("age"),
  p.getItem(4).cast("int").as("marks"))
```

#### Step 3 — Query 1: department summary (`:paste`)

```scala
val summary = students
  .groupBy("department")
  .agg(count("*").as("students"), round(avg("marks"), 1).as("avg_marks"), max("marks").as("top"))

val q1 = summary.writeStream
  .format("console")
  .outputMode("complete")
  .trigger(Trigger.ProcessingTime("5 seconds"))
  .option("checkpointLocation", "C:/sparkdata/checkpoints/b_summary")
  .start()
```

#### Step 4 — Query 2: failures (`:paste`)

```scala
val fails = students.filter($"marks" < 40).select("student_id", "name", "department", "marks")

val q2 = fails.writeStream
  .format("console")
  .outputMode("append")
  .option("checkpointLocation", "C:/sparkdata/checkpoints/b_fails")
  .start()
```

#### Step 5 — Query 3: grade distribution (`:paste`)

```scala
val graded = students.withColumn("grade",
  when($"marks" >= 90, "A")
    .when($"marks" >= 80, "B")
    .when($"marks" >= 40, "C")
    .otherwise("FAIL"))

val q3 = graded.groupBy("grade").count().writeStream
  .format("console")
  .outputMode("complete")
  .trigger(Trigger.ProcessingTime("5 seconds"))
  .option("checkpointLocation", "C:/sparkdata/checkpoints/b_grades")
  .start()
```

#### Step 6 — Send data (W3)

```powershell
kafka-console-producer.bat --topic marks --bootstrap-server localhost:9092
```

```text
101,Alice,CSE,21,85
102,Bob,ECE,22,72
103,Charlie,CSE,21,91
104,David,IT,23,35
105,Emma,ECE,22,88
106,Farah,IT,22,48
```

#### Step 7 — Expected results ✅

**Department summary**

```text
+----------+--------+---------+---+
|department|students|avg_marks|top|
+----------+--------+---------+---+
|       CSE|       2|     88.0| 91|
|       ECE|       2|     80.0| 88|
|        IT|       2|     41.5| 48|
+----------+--------+---------+---+
```

**Failures** (printed once, in append mode)

```text
+----------+-----+----------+-----+
|student_id| name|department|marks|
+----------+-----+----------+-----+
|       104|David|        IT|   35|
+----------+-----+----------+-----+
```

**Grade distribution**

```text
+-----+-----+
|grade|count|
+-----+-----+
|    A|    1|
|    B|    2|
|    C|    2|
| FAIL|    1|
+-----+-----+
```

**Connection to Unit IV:** these are exactly the `GROUP BY department`, `WHERE marks < 40` and `CASE WHEN` grading queries you wrote in Unit IV, now running on a **live stream**. The streaming SQL version also works: register `students.createOrReplaceTempView("students_stream")` and run `spark.sql("SELECT department, AVG(marks) ... GROUP BY department")`.

---

### 5. Project C — Word Count over Kafka

The classic "hello world" of streaming, with Kafka as the source.

#### Step 1 — Topic (W3)

```powershell
kafka-topics.bat --create --topic lines --bootstrap-server localhost:9092 --partitions 1 --replication-factor 1
```

#### Step 2 — Streaming word count (W4, `:paste`)

```scala
val raw = spark.readStream
  .format("kafka")
  .option("kafka.bootstrap.servers", "localhost:9092")
  .option("subscribe", "lines")
  .load()

val words = raw.select(explode(split($"value".cast("string"), " ")).as("word"))
val counts = words.groupBy("word").count()

val q = counts.writeStream
  .format("console")
  .outputMode("complete")
  .trigger(Trigger.ProcessingTime("3 seconds"))
  .option("checkpointLocation", "C:/sparkdata/checkpoints/c_wc")
  .start()
```

(No `startingOffsets` here, so the default `latest` applies: only lines sent **after** the query starts are counted.)

#### Step 3 — Send lines (W3)

```powershell
kafka-console-producer.bat --topic lines --bootstrap-server localhost:9092
```

```text
>hello spark hello kafka
>kafka streams spark
```

#### Step 4 — Expected result ✅

```text
+-------+-----+
|   word|count|
+-------+-----+
|  hello|    2|
|  kafka|    2|
|  spark|    2|
|streams|    1|
+-------+-----+
```

(Row order in the console may differ. Counts accumulate across batches, as in Phase 6.)

---

### 6. Project D — File → Kafka → Spark batch analysis

**Goal.** Load an existing file into Kafka (batch ingestion), then analyse the topic with a **batch** Spark read. This shows that one Kafka topic serves both styles.

```text
sales.txt ──▶ file_to_kafka.py ──▶ [sales-history] ──▶ spark.read (batch) ──▶ SQL report
```

#### Step 1 — Topic and load (W3)

```powershell
kafka-topics.bat --create --topic sales-history --bootstrap-server localhost:9092 --partitions 2 --replication-factor 1
python C:\kafka-labs\file_to_kafka.py
```

(Edit the script's topic name to `sales-history`.)

#### Step 2 — Batch read (W4, `:paste`)

```scala
val hist = spark.read
  .format("kafka")
  .option("kafka.bootstrap.servers", "localhost:9092")
  .option("subscribe", "sales-history")
  .option("startingOffsets", "earliest")
  .option("endingOffsets", "latest")
  .load()

println(hist.count())
```

```text
6
```

#### Step 3 — Parse and use Unit IV SQL (W4)

```scala
val p = split($"value".cast("string"), ",")

val sales = hist.select(
  p.getItem(0).cast("int").as("order_id"), p.getItem(1).as("product"),
  p.getItem(2).as("city"), p.getItem(3).cast("int").as("quantity"), p.getItem(4).cast("int").as("price"))

sales.createOrReplaceTempView("sales")

spark.sql("""
  SELECT product, SUM(quantity * price) AS revenue
  FROM sales GROUP BY product ORDER BY revenue DESC
""").show()
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

(These are the same totals as Unit IV, Phase 6, because we loaded the same six rows.)

**Key learning:** the *same data* in Kafka can be processed as a **batch** (`spark.read`, finite) or as a **stream** (`spark.readStream`, endless). The Spark code after the read is identical.

---

### 7. Practical exam routine

A plan you can follow when the clock is running.

| Minute | Action |
| --- | --- |
| 0–3 | Start ZooKeeper (W1) → Kafka (W2) → `jps` check |
| 3–5 | Create the needed topics (W3) |
| 5–6 | Open W4: `spark-shell --packages ...` |
| 6–8 | Define the DataFrame code (`:paste`) and start the query |
| 8–10 | Start the producer (W3), type 5–6 sample records |
| 10–12 | Show the output and explain it |
| 12+ | Answer follow-ups: stop the query, change the output mode, add a filter |

**What to demonstrate to the examiner**

1. **Kafka running:** `jps` output.
2. **Topic:** `--describe` (partitions, leader).
3. **Producer → consumer:** a message typed in W3 appears in W5.
4. **Spark reading Kafka:** the schema and the parsed table.
5. **Processing:** aggregation or filter output.
6. **A fault-tolerance point:** mention checkpoints and offsets.

**Sentences to have ready**

| Topic | One-line answer |
| --- | --- |
| Why Kafka? | Decouples producers and consumers, buffers, stores and lets many consumers read |
| Why Spark? | Distributed processing with SQL, windows and exactly-once with checkpoints |
| Why cast the value? | Kafka stores bytes; the connector returns binary columns |
| Why a checkpoint? | To remember offsets and state, so a restart continues without loss or duplication |

---

### 8. The "question → pattern" method

Underline the keywords in the question; each maps to a component.

| Keyword in the question | Use |
| --- | --- |
| "publish", "send messages" | Producer (`kafka-console-producer`, Python, Scala) |
| "read messages", "subscribe" | Consumer (`kafka-console-consumer`) |
| "from the beginning", "old messages" | `--from-beginning` / `startingOffsets=earliest` |
| "partitions", "parallel" | `--partitions N`; one Spark task per partition |
| "ordering of related messages" | Use a **key** |
| "same group", "share the work" | Same `group.id` |
| "all subscribers get the message" | Different groups |
| "live count", "running total" | `groupBy` + `complete` mode |
| "only new/changed rows" | `update` / `append` mode |
| "last N minutes", "per 30 seconds" | `window(...)` + watermark |
| "late data" | Watermark |
| "bad records", "invalid" | Dead-letter queue |
| "duplicates" | `dropDuplicates("id")` |
| "add category/department name" | Stream–static join |
| "alert", "threshold" | `filter` + write to alerts topic |
| "save to a file" | `writeStream.format("csv")` + checkpoint |
| "recover after failure" | `checkpointLocation` |
| "historical data", "reprocess" | `startingOffsets=earliest` + new checkpoint |
| "load a file into Kafka" | Batch-ingestion producer script |
| "legacy API" | DStreams (`StreamingContext`) |

---

### 9. Ten exam-style scenarios

#### Scenario 1 — Create a topic and exchange messages

**Question.** Create a topic `exam1` with 2 partitions. Send three messages from a producer and receive them in a consumer.

```powershell
kafka-topics.bat --create --topic exam1 --bootstrap-server localhost:9092 --partitions 2 --replication-factor 1
kafka-console-producer.bat --topic exam1 --bootstrap-server localhost:9092
kafka-console-consumer.bat --topic exam1 --from-beginning --bootstrap-server localhost:9092
```

#### Scenario 2 — Keys and partitions

**Question.** Show that messages with the same key go to the same partition.

```powershell
kafka-console-producer.bat --topic exam1 --bootstrap-server localhost:9092 --property parse.key=true --property key.separator=:
kafka-console-consumer.bat --topic exam1 --from-beginning --bootstrap-server localhost:9092 --property print.key=true --property print.partition=true
```

Send `A:1`, `B:2`, `A:3`; show that both `A` records print the same partition.

#### Scenario 3 — Consumer groups

**Question.** Demonstrate that consumers in the same group share the messages and different groups each get all.

```powershell
REM Run this in TWO windows (same group g1) → they split the messages
kafka-console-consumer.bat --topic exam1 --group g1 --bootstrap-server localhost:9092

REM Run this in a THIRD window (different group g2) → it gets everything
kafka-console-consumer.bat --topic exam1 --group g2 --bootstrap-server localhost:9092
```

#### Scenario 4 — Lag and offsets

**Question.** Show the offset position and lag of a group.

```powershell
kafka-consumer-groups.bat --bootstrap-server localhost:9092 --describe --group g1
```

Explain `CURRENT-OFFSET`, `LOG-END-OFFSET` and `LAG`.

#### Scenario 5 — Spark reads Kafka

**Question.** Read topic `exam1` in Spark and print the message text with its partition and offset.

```scala
val raw = spark.readStream.format("kafka")
  .option("kafka.bootstrap.servers", "localhost:9092").option("subscribe", "exam1")
  .option("startingOffsets", "earliest").load()

raw.selectExpr("CAST(value AS STRING) AS msg", "partition", "offset")
  .writeStream.format("console").outputMode("append").start()
```

(Use `:paste` for the chain.)

#### Scenario 6 — Streaming word count

**Question.** Count words arriving on a Kafka topic. → Project C.

#### Scenario 7 — Parse and aggregate

**Question.** Messages are `product,quantity` (for example `Laptop,2`). Compute the total quantity per product.

```scala
val p = split($"value".cast("string"), ",")
val items = raw.select(p.getItem(0).as("product"), p.getItem(1).cast("int").as("quantity"))
items.groupBy("product").agg(sum("quantity").as("total_quantity"))
  .writeStream.format("console").outputMode("complete").start()
```

#### Scenario 8 — Filter and write back

**Question.** Read marks messages, keep students with marks ≥ 80, and write them to another topic.

```scala
val top = students.filter($"marks" >= 80)
  .select($"student_id".cast("string").as("key"),
          to_json(struct($"student_id", $"name", $"department", $"marks")).as("value"))

top.writeStream.format("kafka")
  .option("kafka.bootstrap.servers", "localhost:9092").option("topic", "toppers")
  .option("checkpointLocation", "C:/sparkdata/checkpoints/toppers").start()
```

#### Scenario 9 — Windowed count

**Question.** Count orders per city for every 30-second window.

```scala
val w = sales2.withWatermark("timestamp", "1 minute")
  .groupBy(window($"timestamp", "30 seconds"), $"city").count()
```

(`sales2` keeps the Kafka `timestamp` column.) Write it with `update` mode to the console.

#### Scenario 10 — Explain the whole pipeline

**Question.** Draw and explain a pipeline for live sales monitoring.

```text
App/generator ─▶ Kafka (sales) ─▶ Spark Structured Streaming ─▶ console/alerts topic/files
                       ▲                      ▲
                  ZooKeeper               checkpoint (offsets + state)
```

Explain: producers publish events; Kafka stores them in partitions replicated for fault tolerance; Spark reads them as a streaming DataFrame, parses and aggregates them; checkpoints give exactly-once recovery; results go to a console, files or another topic.

---

### 10. Troubleshooting master table

| Symptom | First thing to check | Fix |
| --- | --- | --- |
| Kafka will not start | Is ZooKeeper running? Port in use? | Start ZooKeeper first; free ports 2181/9092 |
| `InconsistentClusterIdException` | Only one data folder was deleted | Delete **both** data folders |
| Producer/consumer: connection refused | Broker running? | `jps`; `localhost:9092` |
| Consumer shows nothing | Missing `--from-beginning` or new group | Add `--from-beginning` |
| `Failed to find data source: kafka` | Connector missing | Start Spark with `--packages ...spark-sql-kafka-0-10_2.13:3.5.9` |
| `NoSuchMethodError` at start-up | Version/Scala mismatch | Use `_2.13` and `3.5.9` |
| Streaming query prints nothing | `startingOffsets=latest` and no new data | Send data, or use `earliest` |
| First line works, next lines fail | Typed a multi-line chain line by line | Use `:paste` + Ctrl + D |
| Columns show NULL | Wrong delimiter/schema | Print `CAST(value AS STRING)` and compare |
| Hex bytes on screen | Forgot the cast | `CAST(value AS STRING)` |
| Offset/data-loss error on start | Kafka data reset, old checkpoint kept | Delete the checkpoint folder |
| `Append output mode not supported` | Aggregation in append mode | `complete` or `update` |
| Query stops after an error | Exception in the stream | Read `q.exception` / scroll to the first error |
| Spark very slow | 200 shuffle partitions | `--conf spark.sql.shuffle.partitions=4` |
| Kafka crashes after topic delete | Windows file locking | Never delete topics; reset data folders |

---

### 11. Shutdown and cleanup

```text
1. Spark:    spark.streams.active.foreach(_.stop())  → :quit
2. W5, W3:   Ctrl+C on consumers and producers
3. W2:       Ctrl+C on Kafka (wait for the prompt)
4. W1:       Ctrl+C on ZooKeeper
```

**Fresh start for the next session (optional)**

```powershell
rmdir /S /Q C:\kafka\data\kafka-logs
rmdir /S /Q C:\kafka\data\zookeeper
rmdir /S /Q C:\sparkdata\checkpoints
```

Delete **all three** together: Kafka data, ZooKeeper data and Spark checkpoints.

---

### 12. Master template

Everything for a standard Kafka → Spark → result practical:

```powershell
REM -------- WINDOWS 1 & 2 : start --------
bin\windows\zookeeper-server-start.bat config\zookeeper.properties
bin\windows\kafka-server-start.bat config\server.properties

REM -------- WINDOW 3 : topic + producer --------
kafka-topics.bat --create --topic T --bootstrap-server localhost:9092 --partitions 3 --replication-factor 1
kafka-console-producer.bat --topic T --bootstrap-server localhost:9092

REM -------- WINDOW 4 : Spark --------
spark-shell --packages org.apache.spark:spark-sql-kafka-0-10_2.13:3.5.9 --conf spark.sql.shuffle.partitions=4
```

```scala
// (inside spark-shell, use :paste for multi-line blocks)
import org.apache.spark.sql.functions._
import org.apache.spark.sql.streaming.Trigger
import spark.implicits._

val raw = spark.readStream.format("kafka")
  .option("kafka.bootstrap.servers", "localhost:9092")
  .option("subscribe", "T")
  .option("startingOffsets", "earliest")
  .load()

val p = split($"value".cast("string"), ",")
val df = raw.select(p.getItem(0).as("a"), p.getItem(1).cast("int").as("b"))     // parse

val result = df.groupBy("a").agg(sum("b").as("total"))                          // process

val q = result.writeStream.format("console").outputMode("complete")             // sink
  .trigger(Trigger.ProcessingTime("5 seconds"))
  .option("checkpointLocation", "C:/sparkdata/checkpoints/t1")
  .start()

// later
spark.streams.active.foreach(_.stop())
```

**The whole unit in five lines**

```text
1. Kafka stores events in partitioned, replicated topics (ZooKeeper coordinates).
2. Producers write, consumers read; groups share work; offsets are bookmarks.
3. Spark reads a topic with readStream.format("kafka"); key/value are binary → cast.
4. Process with DataFrame/SQL; choose output mode, trigger and checkpoint.
5. Write to console, files or another topic; patterns: DLQ, enrichment, dedupe, replay.
```

**Phase 9 completed**

---

> **Next → Phase 10: Final Revision** (syllabus coverage map, cheat sheets, error table, **exactly 30 viva questions**, last-10-minute sheet). File: `10_Unit5_Final_Revision_30_Viva.md`
