# Unit V — Phase 7: Integration of Apache Kafka with Spark

> **Continuing from Phase 6.** We know how Kafka stores events (Phases 1–5) and how Structured Streaming processes streams (Phase 6). Now we connect them: **Spark reads a Kafka topic as a streaming DataFrame, processes it, and writes results to the console, to files, or back into Kafka.** This is the syllabus topic *Integration of Apache Kafka with Spark*.
>
> **Verification note (honest):**
> - ✅ **Verified in a real `spark-shell`:** all the *transformation logic* in this phase: casting binary `key`/`value`, splitting CSV messages, `from_json`, `to_json`, aggregations, handling bad records. We tested it on a DataFrame with **exactly the same schema the Kafka source produces**.
> - ⚠️ **Not run here:** the lines that physically connect to a Kafka broker (`format("kafka")`, the broker options). My environment cannot run a Kafka broker or download the connector. Those lines follow the official Spark 3.5 + Kafka 3.4 syntax, and their outputs are marked ⚠️ **Typical output**.

> **Reminder (Phase 6, Section 2b):** code blocks in this phase with lines starting with `.` (`.format(...)`, `.option(...)`) must be entered with **`:paste`** (finish with **Ctrl + D**), or the shell runs only the first line.

---

## Phase 7 — Roadmap

| # | Topic |
| --- | --- |
| 1 | The big picture |
| 2 | Starting Spark with the Kafka connector |
| 3 | Reading Kafka: `readStream` |
| 4 | Required and common options |
| 5 | The schema of the Kafka DataFrame |
| 6 | First stream: Kafka → console |
| 7 | Casting binary to string |
| 8 | Parsing CSV messages |
| 9 | Parsing JSON messages |
| 10 | Aggregating a Kafka stream |
| 11 | Starting offsets and batch reads |
| 12 | Writing results to Kafka |
| 13 | Other sinks: files and `foreachBatch` |
| 14 | How Spark tracks offsets and checkpoints |
| 15 | Parallelism and rate limiting |
| 16 | Delivery guarantees |
| 17 | The legacy DStream integration |
| 18 | PySpark version |
| 19 | Troubleshooting |
| 20 | Practical scenarios |
| 21 | Viva questions |
| 22 | Master revision |

---

### 1. The big picture

```text
 Python / console producer                Kafka                       Spark
┌──────────────────────────┐      ┌──────────────────┐      ┌──────────────────────────┐
│ 1,Laptop,Delhi,2,55000   │─────▶│  topic: sales    │─────▶│ readStream.format("kafka")│
│ 2,Phone,Mumbai,5,20000   │      │  (bytes)         │      │   cast → parse → groupBy │
└──────────────────────────┘      └──────────────────┘      │ writeStream → console /  │
                                           ▲                │   files / another topic  │
                                           │                └──────────────────────────┘
                                   results can go back ──────────────┘
```

**The recipe (same four steps as Phase 6)**

```text
1. readStream  format("kafka")     ← only the SOURCE changes from Phase 6
2. cast bytes → string, parse columns
3. transform (select / filter / groupBy / window)
4. writeStream  (console / files / kafka)  + checkpoint + start()
```

> **Key idea:** everything you learned in Unit IV (SQL, DataFrames) and Phase 6 (streaming) works unchanged. The Kafka connector simply provides a new **source** and a new **sink**.

---

### 2. Starting Spark with the Kafka connector

Spark does not ship the Kafka connector. Attach it with `--packages` (Phase 4, Section 16):

```powershell
spark-shell --packages org.apache.spark:spark-sql-kafka-0-10_2.13:3.5.9 --conf spark.sql.shuffle.partitions=4
```

| Part | Why |
| --- | --- |
| `spark-sql-kafka-0-10` | Kafka source/sink for Structured Streaming |
| `_2.13` | Matches **your** Scala 2.13.8 |
| `3.5.9` | Matches **your** Spark 3.5.9 |
| `shuffle.partitions=4` | Small value = much faster on a laptop |

**Before running anything, check:**

```powershell
jps
```

You should see `QuorumPeerMain` and `Kafka`. And make sure the topic exists:

```powershell
kafka-topics.bat --create --topic sales --bootstrap-server localhost:9092 --partitions 3 --replication-factor 1
```

(If it already exists you get a harmless *already exists* message.)

---

### 3. Reading Kafka: `readStream`

```scala
import org.apache.spark.sql.functions._
import org.apache.spark.sql.types._
import spark.implicits._

val raw = spark.readStream
  .format("kafka")
  .option("kafka.bootstrap.servers", "localhost:9092")
  .option("subscribe", "sales")
  .option("startingOffsets", "earliest")
  .load()
```

**Reading it piece by piece**

| Line | Meaning |
| --- | --- |
| `spark.readStream` | Create a **streaming** DataFrame |
| `.format("kafka")` | Use the Kafka source |
| `kafka.bootstrap.servers` | The broker address (Phase 3, Section 3) |
| `subscribe` | The topic to read |
| `startingOffsets` | Where to begin the **first** time |
| `.load()` | Build the DataFrame (nothing runs yet; it is lazy) |

```scala
raw.isStreaming
```

```text
res0: Boolean = true
```

**Batch version:** use `spark.read` instead of `spark.readStream`. It reads a *fixed range* of records once and finishes (Section 11).

---

### 4. Required and common options

**Required**

| Option | Meaning |
| --- | --- |
| `kafka.bootstrap.servers` | `host:port` list of brokers |
| **one of** `subscribe`, `subscribePattern`, `assign` | Which topic(s) |

**Choosing topics**

| Option | Example | Meaning |
| --- | --- | --- |
| `subscribe` | `"sales"` or `"sales,orders"` | One or more topics by name |
| `subscribePattern` | `"sales.*"` | All topics matching a regex |
| `assign` | `{"sales":[0,1]}` | Specific partitions (JSON) |

**Common optional settings**

| Option | Meaning | Default |
| --- | --- | --- |
| `startingOffsets` | `earliest`, `latest`, or JSON | `latest` (streaming) |
| `endingOffsets` | `latest` or JSON (batch reads only) | `latest` |
| `maxOffsetsPerTrigger` | Limit records per micro-batch | no limit |
| `failOnDataLoss` | Fail if data was lost (for example expired by retention) | `true` |
| `minPartitions` | Minimum Spark partitions to read with | = Kafka partitions |
| `includeHeaders` | Add a `headers` column | `false` |
| `kafka.<anything>` | Pass any Kafka client setting | – |

> **Remember:** any option starting with `kafka.` is handed to the Kafka consumer. Plain names like `subscribe` and `startingOffsets` belong to Spark.

---

### 5. The schema of the Kafka DataFrame

Whatever you send, the Kafka source always gives the **same fixed columns**:

```scala
raw.printSchema()
```

```text
root
 |-- key: binary (nullable = true)
 |-- value: binary (nullable = true)
 |-- topic: string (nullable = true)
 |-- partition: integer (nullable = true)
 |-- offset: long (nullable = true)
 |-- timestamp: timestamp (nullable = true)
 |-- timestampType: integer (nullable = true)
```

| Column | Meaning |
| --- | --- |
| `key` | Record key as **bytes** (null if none) |
| `value` | Record value as **bytes** (the actual data) |
| `topic` | Topic name |
| `partition` | Partition number |
| `offset` | Offset within the partition |
| `timestamp` | Record timestamp |
| `timestampType` | 0 = CreateTime, 1 = LogAppendTime |

**The most important point of this phase:**

> `key` and `value` are **binary**, not text. Before using them we must **cast** them to string (and then parse). This is the serialization lesson from Phase 2, Section 13.

---

### 6. First stream: Kafka → console

```scala
import org.apache.spark.sql.streaming.Trigger

val q = raw
  .selectExpr("CAST(key AS STRING) AS key", "CAST(value AS STRING) AS value", "partition", "offset")
  .writeStream
  .format("console")
  .outputMode("append")
  .option("truncate", "false")
  .trigger(Trigger.ProcessingTime("3 seconds"))
  .start()
```

Run the Python generator from Phase 5 in another window (`python C:\kafka-labs\sales_generator.py`).

⚠️ **Typical output**

```text
-------------------------------------------
Batch: 1
-------------------------------------------
+------+-------------------------+---------+------+
|key   |value                    |partition|offset|
+------+-------------------------+---------+------+
|Delhi |1,Laptop,Delhi,2,55000   |0        |0     |
|Mumbai|2,Phone,Mumbai,5,20000   |1        |0     |
|Pune  |3,Tablet,Pune,3,15000    |2        |0     |
+------+-------------------------+---------+------+
```

**This is the moment everything connects:** the same message typed in the producer is now a DataFrame row in Spark, together with its partition and offset.

```scala
q.stop()
```

---

### 7. Casting binary to string

✅ **Verified.** We built a static DataFrame with the exact Kafka schema (`key` and `value` as binary) and ran the same transformations that you will run on the stream.

**What binary looks like if you forget to cast**

```scala
raw.show(false)    // (for a static/batch Kafka DataFrame)
```

```text
+-------------------+-------------------------------------------------------------+
|key                |value                                                        |
+-------------------+-------------------------------------------------------------+
|[44 65 6C 68 69]   |[7B 22 6F 72 64 65 72 5F 69 64 22 3A 31 2C 22 70 72 6F ...   |
|[4D 75 6D 62 61 69]|[7B 22 6F 72 64 65 72 5F 69 64 22 3A 32 2C 22 70 72 6F ...   |
+-------------------+-------------------------------------------------------------+
```

Those are **hexadecimal bytes**. `44 65 6C 68 69` is the text "Delhi".

**Two equivalent ways to cast**

```scala
// 1. selectExpr (SQL expressions)
raw.selectExpr("CAST(key AS STRING) AS key", "CAST(value AS STRING) AS value", "partition", "offset")

// 2. select with Column API
raw.select($"key".cast("string"), $"value".cast("string"))
```

**Result** (✅ verified):

```text
+------+---------------------------------------------------------------------------+---------+------+
|key   |value                                                                      |partition|offset|
+------+---------------------------------------------------------------------------+---------+------+
|Delhi |{"order_id":1,"product":"Laptop","city":"Delhi","quantity":2,"price":55000}|0        |0     |
|Mumbai|{"order_id":2,"product":"Phone","city":"Mumbai","quantity":5,"price":20000}|1        |0     |
|Delhi |{"order_id":3,"product":"Tablet","city":"Delhi","quantity":3,"price":15000}|0        |1     |
+------+---------------------------------------------------------------------------+---------+------+
```

> **Rule:** the first transformation on every Kafka stream is `CAST(value AS STRING)`.

---

### 8. Parsing CSV messages

Our generator sends CSV lines: `order_id,product,city,quantity,price`. We turn them into proper typed columns, exactly like the file-to-DataFrame parsing of Unit IV, Phase 3.

```scala
val text = raw.selectExpr("CAST(value AS STRING) AS value")

val parts = split($"value", ",")

val sales = text.select(
  parts.getItem(0).cast("int").as("order_id"),
  parts.getItem(1).as("product"),
  parts.getItem(2).as("city"),
  parts.getItem(3).cast("int").as("quantity"),
  parts.getItem(4).cast("int").as("price")
)
```

**Step by step**

```text
"1,Laptop,Delhi,2,55000"
        |  split($"value", ",")
        ▼
["1","Laptop","Delhi","2","55000"]   ← all strings
        |  getItem(i) + cast
        ▼
order_id=1  product=Laptop  city=Delhi  quantity=2  price=55000
```

✅ **Verified on a Kafka-shaped DataFrame** with student records (`101,Alice,CSE,21,85`):

```text
+----------+-----+----------+---+-----+
|student_id| name|department|age|marks|
+----------+-----+----------+---+-----+
|       101|Alice|       CSE| 21|   85|
|       102|  Bob|       ECE| 22|   72|
+----------+-----+----------+---+-----+
```

Now `sales` is an ordinary streaming DataFrame with typed columns. Run it with the console sink:

```scala
val q = sales.writeStream.format("console").outputMode("append").start()
```

⚠️ **Typical output**

```text
+--------+-------+------+--------+-----+
|order_id|product|  city|quantity|price|
+--------+-------+------+--------+-----+
|       1| Laptop| Delhi|       2|55000|
|       2|  Phone|Mumbai|       5|20000|
+--------+-------+------+--------+-----+
```

---

### 9. Parsing JSON messages

Real systems usually send **JSON**. Spark parses it with `from_json` and a schema.

**Sample message**

```text
{"order_id":1,"product":"Laptop","city":"Delhi","quantity":2,"price":55000}
```

**Step 1 — Describe the structure**

```scala
val jsonSchema = new StructType()
  .add("order_id", IntegerType)
  .add("product", StringType)
  .add("city", StringType)
  .add("quantity", IntegerType)
  .add("price", IntegerType)
```

**Step 2 — Parse**

```scala
val parsed = raw
  .select(from_json($"value".cast("string"), jsonSchema).as("d"))
  .select("d.*")
```

`from_json(...)` returns a **struct** column named `d`; `select("d.*")` expands it into separate columns.

✅ **Verified result:**

```text
+--------+-------+------+--------+-----+
|order_id|product|  city|quantity|price|
+--------+-------+------+--------+-----+
|       1| Laptop| Delhi|       2|55000|
|       2|  Phone|Mumbai|       5|20000|
|       3| Tablet| Delhi|       3|15000|
+--------+-------+------+--------+-----+
```

```text
root
 |-- order_id: integer (nullable = true)
 |-- product: string (nullable = true)
 |-- city: string (nullable = true)
 |-- quantity: integer (nullable = true)
 |-- price: integer (nullable = true)
```

**Bad messages: what really happens** ✅ (verified)

```scala
// three messages: good JSON, JSON with a wrong type, and garbage text
```

```text
+--------------------------------------------------+------------------+
|raw                                               |d                 |
+--------------------------------------------------+------------------+
|{"order_id":1,"product":"Laptop","quantity":2}    |{1, Laptop, 2}    |
|{"order_id":2,"product":"Phone","quantity":"five"}|{2, Phone, NULL}  |
|not json at all                                   |{NULL, NULL, NULL}|
+--------------------------------------------------+------------------+
```

**Lessons**

1. A **wrong type** (`"five"` for an integer) gives a `NULL` for **that field only**.
2. **Garbage text** gives a struct whose fields are **all NULL**. The struct `d` itself is *not* NULL.
3. So to detect bad records, check the **fields**, not `d IS NULL`:

```scala
val good = parsed.filter($"order_id".isNotNull && $"quantity".isNotNull)
```

(Phase 8 builds a *dead-letter pattern* on this.)

---

### 10. Aggregating a Kafka stream

Revenue per product, updated continuously. This is the Phase 6 query on a Kafka source.

```scala
val revenue = sales
  .groupBy("product")
  .agg(sum($"quantity" * $"price").alias("revenue"))

val q = revenue.writeStream
  .format("console")
  .outputMode("complete")
  .trigger(Trigger.ProcessingTime("5 seconds"))
  .option("checkpointLocation", "C:/sparkdata/checkpoints/revenue")
  .start()
```

⚠️ **Typical output** (grows as the generator sends more events):

```text
-------------------------------------------
Batch: 3
-------------------------------------------
+-------+-------+
|product|revenue|
+-------+-------+
| Laptop| 495000|
|  Phone| 260000|
| Tablet| 135000|
+-------+-------+
```

The same query as Unit IV's `GROUP BY product` + `SUM(quantity*price)`, now running forever on live data.

**Using SQL instead of the DataFrame API**

```scala
sales.createOrReplaceTempView("sales_stream")

val revenueSql = spark.sql("""
  SELECT product, SUM(quantity * price) AS revenue
  FROM sales_stream
  GROUP BY product
""")
```

`revenueSql` is also a streaming DataFrame. **Everything from Unit IV works on streams.**

**Event-time window on the Kafka timestamp**

```scala
val perWindow = sales2   // sales2 keeps the Kafka "timestamp" column as event time
  .withWatermark("timestamp", "1 minute")
  .groupBy(window($"timestamp", "30 seconds"), $"city")
  .agg(sum($"quantity" * $"price").alias("revenue"))
```

To keep the Kafka timestamp, include it when parsing:

```scala
val sales2 = raw
  .select($"timestamp", $"value".cast("string").as("value"))
  .select($"timestamp",
          split($"value", ",").getItem(2).as("city"),
          split($"value", ",").getItem(3).cast("int").as("quantity"),
          split($"value", ",").getItem(4).cast("int").as("price"))
```

---

### 11. Starting offsets and batch reads

#### 11.1 `startingOffsets` (streaming)

| Value | Meaning |
| --- | --- |
| `"latest"` | Only records that arrive **after** the query starts (default) |
| `"earliest"` | Everything still stored in Kafka |
| JSON | Exact positions, e.g. `{"sales":{"0":5,"1":10,"2":0}}` |

```scala
.option("startingOffsets", "earliest")
```

> **Critical rule:** `startingOffsets` is used **only the first time** a query runs. When the query restarts with the **same checkpoint folder**, Spark continues from the checkpointed offsets and **ignores** `startingOffsets` (Section 14).

#### 11.2 Batch read: a fixed range, read once

```scala
val batch = spark.read
  .format("kafka")
  .option("kafka.bootstrap.servers", "localhost:9092")
  .option("subscribe", "sales")
  .option("startingOffsets", "earliest")
  .option("endingOffsets", "latest")
  .load()

batch.selectExpr("CAST(value AS STRING) AS value").show(5, false)
println(batch.count())
```

This is a normal static DataFrame: `show()`, `count()` and SQL work. It is perfect for **debugging**: *"what is actually inside my topic?"*

⚠️ **Typical output**

```text
+-----------------------+
|value                  |
+-----------------------+
|1,Laptop,Delhi,2,55000 |
|2,Phone,Mumbai,5,20000 |
...
```

| Streaming read | Batch read |
| --- | --- |
| `spark.readStream` | `spark.read` |
| Runs continuously | Runs once and finishes |
| `startingOffsets` default `latest` | `startingOffsets` default `earliest` |
| Results via `writeStream` | Results via `show()` / `write` |

---

### 12. Writing results to Kafka

Spark can **publish** a DataFrame to a Kafka topic. This is how pipelines are chained (Phase 8).

**Rules of the Kafka sink**

| Rule | Detail |
| --- | --- |
| Required column | `value` (string or binary) |
| Optional column | `key` (string or binary) |
| Optional column | `topic` (if you do not use the `topic` option) |
| Required option | `kafka.bootstrap.servers` |
| Required for streams | `checkpointLocation` |

**Example: send "big orders" to a topic `sales-alerts` as JSON**

First create the topic:

```powershell
kafka-topics.bat --create --topic sales-alerts --bootstrap-server localhost:9092 --partitions 1 --replication-factor 1
```

Then in Spark:

```scala
val alerts = sales
  .withColumn("revenue", $"quantity" * $"price")
  .filter($"revenue" >= 100000)
  .select(
    $"city".as("key"),
    to_json(struct($"order_id", $"product", $"city", $"quantity", $"price", $"revenue")).as("value")
  )

val q = alerts.writeStream
  .format("kafka")
  .option("kafka.bootstrap.servers", "localhost:9092")
  .option("topic", "sales-alerts")
  .option("checkpointLocation", "C:/sparkdata/checkpoints/alerts")
  .start()
```

**What `to_json(struct(...))` does** ✅ (verified)

```text
+------+---------------------------------------------------------------------------+
|key   |value                                                                      |
+------+---------------------------------------------------------------------------+
|Delhi |{"order_id":1,"product":"Laptop","city":"Delhi","quantity":2,"price":55000}|
|Mumbai|{"order_id":2,"product":"Phone","city":"Mumbai","quantity":5,"price":20000}|
|Delhi |{"order_id":3,"product":"Tablet","city":"Delhi","quantity":3,"price":15000}|
+------+---------------------------------------------------------------------------+
```

`struct(...)` packs columns into one object; `to_json(...)` converts it to a JSON string. Exactly the `key` + `value` shape the Kafka sink needs.

**Check the output topic in another window**

```powershell
kafka-console-consumer.bat --topic sales-alerts --from-beginning --bootstrap-server localhost:9092 --property print.key=true
```

⚠️ **Typical output**

```text
Delhi    {"order_id":14,"product":"Laptop","city":"Delhi","quantity":2,"price":55000,"revenue":110000}
```

```text
 Kafka topic "sales"  ──▶  Spark (filter + enrich)  ──▶  Kafka topic "sales-alerts"
```

> **Remember:** the Kafka sink needs a column named exactly `value` (and `key` if you want keys). Forgetting it gives *"Required attribute 'value' not found"*.

---

### 13. Other sinks: files and `foreachBatch`

**Write the parsed stream to CSV files**

```scala
val q = sales.writeStream
  .format("csv")
  .option("path", "C:/sparkdata/out/sales-csv")
  .option("checkpointLocation", "C:/sparkdata/checkpoints/sales-csv")
  .outputMode("append")
  .start()
```

File sinks support **append** mode only (and need a checkpoint). Each micro-batch adds new part files to the folder.

**`foreachBatch`: do anything with each micro-batch**

```scala
val q = revenue.writeStream
  .outputMode("complete")
  .option("checkpointLocation", "C:/sparkdata/checkpoints/rev-fb")
  .foreachBatch { (batchDF: org.apache.spark.sql.DataFrame, batchId: Long) =>
    println(s"--- batch $batchId ---")
    batchDF.orderBy(desc("revenue")).show()
    batchDF.write.mode("overwrite").option("header", "true").csv("C:/sparkdata/out/revenue-latest")
  }
  .start()
```

Here one query shows the table **and** saves it. Inside `foreachBatch` the DataFrame is *static*, so `orderBy`, `show()` and `write` are all allowed.

---

### 14. How Spark tracks offsets and checkpoints

**Spark does not use Kafka consumer groups to remember its position.** Instead:

1. Spark reads a range of offsets for each partition.
2. It records those offsets in the **checkpoint folder** (`offsets` files).
3. When the batch finishes, it writes a **commit** file.

⚠️ **Typical content of an `offsets` file**

```text
{"sales":{"0":42,"1":37,"2":40}}
```

(`topic → partition → next offset to read`.)

**Consequences**

| Fact | What it means |
| --- | --- |
| Spark disables auto-commit and ignores group offsets | A Spark job **does not appear** with its lag in `kafka-consumer-groups.bat` |
| Position lives in the checkpoint | Delete the checkpoint → Spark forgets where it was |
| Restart with the same checkpoint | Continues exactly where it stopped, no loss, no duplicate reads |
| `startingOffsets` | Used only when **no checkpoint exists** |

```text
FIRST RUN  (no checkpoint)  → uses startingOffsets
RESTART    (checkpoint)     → uses checkpoint offsets, ignores startingOffsets
```

**Practical rules**

1. **One checkpoint folder per query.**
2. If you **reset Kafka's data** (Phase 4, Section 12), also **delete the old checkpoint folders**. Otherwise Spark asks for offsets that no longer exist and fails (Section 19).
3. Want to re-read everything? Use a **new checkpoint folder** with `startingOffsets=earliest`.

---

### 15. Parallelism and rate limiting

**One Spark task per Kafka partition.**

```text
Topic sales: P0, P1, P2   ──▶   Spark reads with 3 parallel tasks
```

| Setting | Effect |
| --- | --- |
| More Kafka partitions | More parallel Spark read tasks |
| `minPartitions` | Splits large partitions into more Spark tasks |
| `maxOffsetsPerTrigger` | Caps how many records one batch may read |

```scala
.option("maxOffsetsPerTrigger", 1000)
```

**Why rate limiting?** If Spark was stopped for an hour, millions of records pile up. Reading them all in one giant batch could exhaust memory. `maxOffsetsPerTrigger` makes Spark catch up in controlled steps. It also keeps laptop demos smooth.

---

### 16. Delivery guarantees

| Stage | Guarantee | Why |
| --- | --- | --- |
| **Kafka source → Spark** | Processing is **exactly-once** | Replayable source + checkpointed offsets |
| **Spark → file sink** | Exactly-once | The file sink tracks committed batches |
| **Spark → Kafka sink** | **At-least-once** | A retry can republish a batch → possible duplicates |
| **foreachBatch** | You control it | Use `batchId` to make writes idempotent |

**Making results effectively exactly-once**

1. Use `checkpointLocation` always.
2. Keep keys/ids in the data (`order_id`) so downstream consumers can **de-duplicate**.
3. With `foreachBatch`, write with `batchId` or upsert logic so a replayed batch overwrites rather than duplicates.

> **Viva line:** Structured Streaming with Kafka source and checkpointing gives end-to-end exactly-once for supported sinks; with the Kafka sink it is at-least-once, so downstream de-duplication is advised.

---

### 17. The legacy DStream integration

For completeness. The old DStream API has its own Kafka package:

```powershell
spark-shell --packages org.apache.spark:spark-streaming-kafka-0-10_2.13:3.5.9
```

```scala
import org.apache.spark.streaming.{StreamingContext, Seconds}
import org.apache.spark.streaming.kafka010._
import org.apache.kafka.common.serialization.StringDeserializer

val ssc = new StreamingContext(sc, Seconds(5))

val kafkaParams = Map[String, Object](
  "bootstrap.servers"  -> "localhost:9092",
  "key.deserializer"   -> classOf[StringDeserializer],
  "value.deserializer" -> classOf[StringDeserializer],
  "group.id"           -> "dstream-group",
  "auto.offset.reset"  -> "latest"
)

val stream = KafkaUtils.createDirectStream[String, String](
  ssc,
  LocationStrategies.PreferConsistent,
  ConsumerStrategies.Subscribe[String, String](Array("sales"), kafkaParams)
)

stream.map(record => record.value).print()

ssc.start()
ssc.awaitTerminationOrTimeout(30000)
ssc.stop(stopSparkContext = false)
```

| Piece | Meaning |
| --- | --- |
| `createDirectStream` | "Direct" approach: Spark reads from Kafka partitions itself, with no receiver |
| `PreferConsistent` | Distribute partitions evenly over executors |
| `Subscribe` | The topics to read |
| `record.value` | Each record is a Kafka `ConsumerRecord` object |

> **Use Structured Streaming for all new work.** DStreams are legacy. Know this snippet only for theory questions.

---

### 18. PySpark version

Your Python 3.10 and Unit VI (PySpark) make this a useful preview. The logic is identical:

```powershell
pyspark --packages org.apache.spark:spark-sql-kafka-0-10_2.13:3.5.9
```

```python
from pyspark.sql.functions import col, split, sum as _sum

raw = (spark.readStream
       .format("kafka")
       .option("kafka.bootstrap.servers", "localhost:9092")
       .option("subscribe", "sales")
       .option("startingOffsets", "earliest")
       .load())

parts = split(col("value").cast("string"), ",")

sales = raw.select(
    parts.getItem(0).cast("int").alias("order_id"),
    parts.getItem(1).alias("product"),
    parts.getItem(2).alias("city"),
    parts.getItem(3).cast("int").alias("quantity"),
    parts.getItem(4).cast("int").alias("price"),
)

revenue = sales.groupBy("product").agg(_sum(col("quantity") * col("price")).alias("revenue"))

q = (revenue.writeStream.format("console").outputMode("complete").start())
```

Only the syntax changes (`.alias`, `col(...)`, no `$`); the concepts are the same.

---

### 19. Troubleshooting

| Symptom / error | Cause | Fix |
| --- | --- | --- |
| `Failed to find data source: kafka` | Connector not loaded | Start with `--packages org.apache.spark:spark-sql-kafka-0-10_2.13:3.5.9` |
| `NoSuchMethodError` / `NoClassDefFoundError` / `ClassNotFoundException` at start | Wrong connector version or Scala suffix (`_2.12` vs `_2.13`, or version ≠ Spark) | Match **Scala (2.13)** and **Spark (3.5.9)** exactly |
| Dependency download fails / `unresolved dependency` | No internet for the first run | Connect once, or use `--jars` (Phase 4, Section 16) |
| `Option 'kafka.bootstrap.servers' must be specified` | Missing required option | Add it |
| `Subscribe/assign/subscribePattern` error | None or several of the three given | Provide **exactly one** |
| Hangs / `TimeoutException` / *Connection refused* to localhost:9092 | Broker not running | Start ZooKeeper and Kafka; check `jps` |
| Query starts, nothing prints | No new data and `startingOffsets=latest` | Send data, or use `earliest` |
| `Partition sales-0's offset was changed from X to Y` / data loss message | Topic recreated, retention deleted data, or **Kafka data reset but old checkpoint kept** | Delete the old checkpoint; or set `failOnDataLoss=false` for demos |
| Console shows hex like `[44 65 6C ...]` | Forgot to cast | `CAST(value AS STRING)` |
| All parsed columns are NULL | Wrong delimiter/schema | Print the raw string first and compare |
| `Required attribute 'value' not found` | Writing to Kafka without a `value` column | Select a column named `value` |
| `Checkpoint location must be specified` / similar | Kafka/file sink without checkpoint | Add `checkpointLocation` |
| Restarted query does not re-read old data | The checkpoint remembers progress | New checkpoint folder |
| Spark job missing in `kafka-consumer-groups` | Spark tracks offsets in its checkpoint | Expected behaviour |
| Very slow on laptop | 200 shuffle partitions | `--conf spark.sql.shuffle.partitions=4` |
| Duplicate records in the output topic | Kafka sink is at-least-once | De-duplicate downstream by `order_id` |

---

### 20. Practical scenarios — Phase 7 practice

1. **Start** `spark-shell` with the connector and run `spark.readStream.format("kafka")`.
2. **Print the schema** of a Kafka stream and name each column.
3. **Show** raw (uncast) data, then cast `key` and `value` to strings.
4. **Stream to the console** with partition and offset columns.
5. **Parse CSV** order messages into typed columns.
6. **Parse JSON** messages with `from_json` and a schema.
7. **Filter** orders with revenue above 100,000.
8. **Revenue per product** (complete mode) with a checkpoint.
9. **Revenue per city per 30-second window** with a watermark.
10. **Batch-read** the topic once with `startingOffsets=earliest` and `endingOffsets=latest`; print `count()`.
11. **Write alerts to a second topic** and read it with the console consumer.
12. **Save** the parsed stream as CSV files.
13. Use **`foreachBatch`** to show and save each batch's revenue table.
14. **Stop** the query, restart with the **same checkpoint**, and confirm Spark continues, not restarts.
15. **Reset** Kafka data, forget to delete the checkpoint, and observe the data-loss error (then fix it).
16. Limit batches with **`maxOffsetsPerTrigger`**.

---

### 21. Phase 7 — viva questions and answers

#### Q1. How does Spark read data from Kafka?

Through the Kafka data source of Structured Streaming: `spark.readStream.format("kafka")` with options such as `kafka.bootstrap.servers` and `subscribe`.

#### Q2. Which package is needed for Spark 3.5.9 with Scala 2.13?

`org.apache.spark:spark-sql-kafka-0-10_2.13:3.5.9`.

#### Q3. How do we add the connector to the shell?

`spark-shell --packages org.apache.spark:spark-sql-kafka-0-10_2.13:3.5.9`.

#### Q4. What are the required options for the Kafka source?

`kafka.bootstrap.servers` and exactly one of `subscribe`, `subscribePattern` or `assign`.

#### Q5. What columns does a Kafka DataFrame have?

`key`, `value`, `topic`, `partition`, `offset`, `timestamp` and `timestampType`.

#### Q6. What are the data types of `key` and `value`?

Binary. They must be cast to string and parsed.

#### Q7. How do you convert the Kafka value to text?

`CAST(value AS STRING)` or `$"value".cast("string")`.

#### Q8. How do you parse JSON messages?

With `from_json(col, schema)` and then `select("d.*")` to flatten the struct.

#### Q9. How do you parse CSV messages?

With `split(value, ",")` and `getItem(i)` plus casts to the proper types.

#### Q10. What does `startingOffsets` do?

It sets where the first run begins: `earliest`, `latest` or specific offsets. After a restart with a checkpoint it is ignored.

#### Q11. What is the difference between `spark.read` and `spark.readStream` for Kafka?

`read` is a one-time batch over an offset range; `readStream` is a continuous streaming query.

#### Q12. How do you write a DataFrame to Kafka?

`writeStream.format("kafka")` with `kafka.bootstrap.servers`, `topic` and `checkpointLocation`; the DataFrame needs a `value` column and optionally `key`.

#### Q13. Why is `to_json(struct(...))` used before writing to Kafka?

To pack several columns into a single JSON string for the `value` column.

#### Q14. Where does Spark store the Kafka offsets it has processed?

In the query's checkpoint folder, not in Kafka's consumer-group offsets.

#### Q15. Why does a Spark streaming job not appear in `kafka-consumer-groups`?

Because Spark manages offsets itself in its checkpoint rather than committing them as consumer-group offsets.

#### Q16. What is the relation between Kafka partitions and Spark tasks?

Spark creates one read task per Kafka partition (or more with `minPartitions`).

#### Q17. What is `maxOffsetsPerTrigger`?

A limit on how many records a single micro-batch may read, used for rate limiting and controlled catch-up.

#### Q18. What delivery guarantee does Kafka-to-Spark processing give?

Exactly-once processing with a checkpoint, thanks to the replayable Kafka source; the Kafka sink is at-least-once.

#### Q19. What error do you get without the connector and why?

`Failed to find data source: kafka`, because the package containing the Kafka source was not loaded.

#### Q20. What happens if the Kafka data is reset but the old checkpoint is kept?

Spark asks for offsets that no longer exist and fails with a data-loss/offset-changed error; delete the old checkpoint.

#### Q21. What is `failOnDataLoss`?

An option that makes the query fail when Kafka data it expected is gone (for example expired by retention); `false` makes Spark continue.

#### Q22. Which API is recommended for new Kafka-Spark pipelines?

Structured Streaming (`spark-sql-kafka-0-10`), not DStreams.

---

### 22. Phase 7 — final master code revision

```powershell
spark-shell --packages org.apache.spark:spark-sql-kafka-0-10_2.13:3.5.9 --conf spark.sql.shuffle.partitions=4
```

```scala
import org.apache.spark.sql.functions._
import org.apache.spark.sql.types._
import org.apache.spark.sql.streaming.Trigger
import spark.implicits._

// 1. READ from Kafka
val raw = spark.readStream
  .format("kafka")
  .option("kafka.bootstrap.servers", "localhost:9092")
  .option("subscribe", "sales")
  .option("startingOffsets", "earliest")
  .load()

// 2. CAST + PARSE (CSV messages: order_id,product,city,quantity,price)
val parts = split($"value".cast("string"), ",")
val sales = raw.select(
  parts.getItem(0).cast("int").as("order_id"),
  parts.getItem(1).as("product"),
  parts.getItem(2).as("city"),
  parts.getItem(3).cast("int").as("quantity"),
  parts.getItem(4).cast("int").as("price"))

// 3. TRANSFORM
val revenue = sales.groupBy("product").agg(sum($"quantity" * $"price").as("revenue"))

// 4a. WRITE to console
val q1 = revenue.writeStream.format("console").outputMode("complete")
  .trigger(Trigger.ProcessingTime("5 seconds"))
  .option("checkpointLocation", "C:/sparkdata/checkpoints/rev").start()

// 4b. WRITE back to Kafka
val alerts = sales.filter($"quantity" * $"price" >= 100000)
  .select($"city".as("key"), to_json(struct($"order_id", $"product", $"city", $"quantity", $"price")).as("value"))
val q2 = alerts.writeStream.format("kafka")
  .option("kafka.bootstrap.servers", "localhost:9092")
  .option("topic", "sales-alerts")
  .option("checkpointLocation", "C:/sparkdata/checkpoints/alerts").start()

// 5. MANAGE
spark.streams.active.foreach(q => println(q.id + " " + q.status))
spark.streams.active.foreach(_.stop())
```

**Quick-recall card**

| Need | Syntax |
| --- | --- |
| Connector | `--packages org.apache.spark:spark-sql-kafka-0-10_2.13:3.5.9` |
| Streaming read | `spark.readStream.format("kafka").option("kafka.bootstrap.servers","localhost:9092").option("subscribe","t").load()` |
| Batch read | `spark.read.format("kafka")...` + `endingOffsets` |
| Kafka columns | `key value topic partition offset timestamp timestampType` |
| Text | `CAST(value AS STRING)` |
| CSV parse | `split(value, ",")` + `getItem` |
| JSON parse | `from_json(value, schema)` |
| Write to Kafka | `.format("kafka").option("topic","t")` + `value` column |
| JSON for Kafka | `to_json(struct(...)).as("value")` |
| Start position | `startingOffsets = earliest/latest` (first run only) |
| Safety | `checkpointLocation` per query |

**Phase 7 completed**

---

> **Next → Phase 8: Kafka Pipeline and Ingestion Patterns** (pipeline stages, reference architecture, ingestion patterns, dead-letter queue, enrichment joins, fan-out/fan-in, lambda vs kappa, Kafka Connect, delivery guarantees). File: `8_Unit5_Kafka_Pipeline_Ingestion_Patterns.md`
