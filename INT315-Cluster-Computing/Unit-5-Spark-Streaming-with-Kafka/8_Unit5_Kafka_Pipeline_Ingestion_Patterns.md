# Unit V — Phase 8: Kafka Pipeline and Ingestion Patterns

> **Continuing from Phase 7.** We can read from and write to Kafka with Spark. Now we step back and design **whole pipelines**: how data enters the system (**ingestion**), how it flows through stages, and the standard **patterns** engineers use. This covers the syllabus topics **Kafka pipeline** and **ingestion patterns**.
>
> **Verification note:** The Spark transformation logic in this phase (dead-letter split, de-duplication, stream-static enrichment, windowed aggregation) was run in a real `spark-shell`; those outputs are genuine. Lines that talk to a Kafka broker follow the official syntax and are shown as ⚠️ **Typical output**.

> **Reminder:** enter multi-line chained code (lines starting with `.`) using **`:paste`** and finish with **Ctrl + D** (Phase 6, Section 2b).

---

## Phase 8 — Roadmap

| # | Topic |
| --- | --- |
| 1 | What is a data pipeline? |
| 2 | Stages of a Kafka-based pipeline |
| 3 | What is ingestion? Three styles |
| 4 | Reference architecture with multiple topics |
| 5 | Pattern 1 — Batch ingestion (file → Kafka) |
| 6 | Pattern 2 — Real-time event ingestion |
| 7 | Pattern 3 — Log aggregation (fan-in) |
| 8 | Pattern 4 — Change Data Capture (CDC) |
| 9 | Pattern 5 — Fan-out (one topic, many readers) |
| 10 | Pattern 6 — Filter and route |
| 11 | Pattern 7 — Enrichment (stream–static join) |
| 12 | Pattern 8 — Windowed aggregation |
| 13 | Pattern 9 — Dead-letter queue |
| 14 | Pattern 10 — De-duplication |
| 15 | Pattern 11 — Replay and reprocessing |
| 16 | Lambda vs Kappa architecture |
| 17 | The wider Kafka ecosystem |
| 18 | Reliability checklist |
| 19 | A complete pipeline design |
| 20 | Common pipeline mistakes |
| 21 | Practical scenarios |
| 22 | Viva questions |
| 23 | Master revision |

---

### 1. What is a data pipeline?

> **A data pipeline is a series of connected steps that move data from where it is produced to where it is used, transforming it on the way.**

```text
SOURCE ──▶ INGEST ──▶ TRANSPORT ──▶ PROCESS ──▶ STORE / SERVE
(apps,     (collect)   (Kafka)      (Spark)     (files, DB,
 sensors,                                         dashboards,
 files)                                           other topics)
```

**Analogy:** a water supply system. Rivers (sources) feed a pumping station (ingestion), pipes (Kafka) carry water, a treatment plant (Spark) cleans it, and taps (dashboards, databases) deliver it.

**Batch pipeline vs streaming pipeline**

| Batch pipeline | Streaming pipeline |
| --- | --- |
| Runs on a schedule (hourly, nightly) | Runs **continuously** |
| Data is complete when processed | Data is processed as it **arrives** |
| Delay: minutes to hours | Delay: seconds or less |
| Example: nightly sales report | Example: live fraud alert |

---

### 2. Stages of a Kafka-based pipeline

| Stage | Question it answers | Typical tools |
| --- | --- | --- |
| **1. Source** | Where does the data originate? | Web apps, mobile apps, sensors, databases, log files |
| **2. Ingestion** | How does it get **into** Kafka? | Producers, Kafka Connect, scripts |
| **3. Transport / buffer** | Where is it stored safely and delivered? | **Kafka topics** |
| **4. Processing** | How is it cleaned, joined, aggregated? | **Spark Structured Streaming** |
| **5. Storage / serving** | Where do results go? | Files, databases, dashboards, other topics |
| **6. Monitoring** | Is it healthy? | Consumer lag, Spark UI, logs |

**Why Kafka sits in the middle**

1. **Decouples** producers from processors: neither needs the other online at the same moment.
2. **Buffers** spikes so Spark is not overwhelmed.
3. **Stores** events, so a new job can replay history.
4. **Feeds many consumers** from one copy of the data.

---

### 3. What is ingestion? Three styles

> **Data ingestion is the process of collecting data from sources and bringing it into a system for storage or processing.**

| Style | How data arrives | Latency | Example |
| --- | --- | --- | --- |
| **Batch ingestion** | Large chunks at intervals | Minutes to hours | Load yesterday's sales file into Kafka |
| **Micro-batch ingestion** | Small chunks every few seconds | Seconds | Spark reading Kafka every 5 s |
| **Real-time (streaming) ingestion** | Each event immediately | Milliseconds to seconds | App publishes every click |

```text
Batch:        ████████████ ........ ████████████ ........
Micro-batch:  ██ ██ ██ ██ ██ ██ ██ ██ ██ ██
Real-time:    ▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪
```

Kafka handles **all three**: it accepts a big file load, steady trickles and bursts equally well.

---

### 4. Reference architecture with multiple topics

Real pipelines use **several topics, one per stage**. Each stage reads one topic and writes the next.

```text
                       ┌───────────────┐
 sources ──▶ sales-raw ─▶│ Spark: clean  │──▶ sales-clean ──▶┌─────────────────┐
 (apps,     (as received) │ + validate    │                   │ Spark: enrich + │──▶ sales-enriched
  files)                  └───────┬───────┘                   │ aggregate       │──▶ files / dashboard
                                  │ bad records               └─────────────────┘
                                  ▼
                              sales-dlq   (dead-letter queue)
```

**Topic design conventions**

| Topic | Contents |
| --- | --- |
| `sales-raw` | Data exactly as received (never modify; allows replay) |
| `sales-clean` | Validated and parsed records |
| `sales-enriched` | Records joined with reference data |
| `sales-alerts` | Business alerts (for example big orders) |
| `sales-dlq` | Records that failed validation |

**Why separate topics?**

- Each stage can be **scaled, fixed and replayed independently**.
- A bug in the cleaning step never destroys the original data.
- Different teams can subscribe to the stage they need.

---

### 5. Pattern 1 — Batch ingestion (file → Kafka)

**Problem:** Data exists in a file; we need it in Kafka so streaming jobs can use it.

```text
sales.txt ──▶ Python producer ──▶ topic sales ──▶ Spark
```

`C:\kafka-labs\file_to_kafka.py`

```python
from kafka import KafkaProducer

producer = KafkaProducer(
    bootstrap_servers="localhost:9092",
    value_serializer=lambda v: v.encode("utf-8"),
)

count = 0
with open(r"C:\sparkdata\sales.txt") as f:
    for line in f:
        line = line.strip()
        if line:                                   # skip blank lines
            producer.send("sales", value=line)
            count += 1

producer.flush()
producer.close()
print("sent", count, "records")
```

```powershell
python C:\kafka-labs\file_to_kafka.py
```

```text
sent 6 records
```

(Each line of Unit IV's `sales.txt` becomes one Kafka record.)

**When to use:** loading historical data, migrating files, testing a pipeline with known data.

---

### 6. Pattern 2 — Real-time event ingestion

**Problem:** Applications generate events continuously; publish each event as it happens.

```text
web app / sensor ──▶ producer ──▶ topic ──▶ consumers (real time)
```

This is our `sales_generator.py` from Phase 5: one event per second. In real systems, the application itself embeds a Kafka producer (or a lightweight agent sends events).

**Design points**

| Point | Guideline |
| --- | --- |
| **Key** | Choose a key that groups events that must stay in order (customer id, device id) |
| **Format** | Prefer **JSON** (readable, flexible) or Avro/Protobuf (compact, schema-checked) |
| **Timestamp** | Include an **event time** field in the data |
| **Reliability** | `acks=all`, idempotence on |

---

### 7. Pattern 3 — Log aggregation (fan-in)

**Problem:** Hundreds of servers write log files. Collect them in **one place**.

```text
Server 1 ─┐
Server 2 ─┼──▶ topic "app-logs" ──▶ Spark (search, count errors) ──▶ dashboard
Server 3 ─┘
```

**Fan-in** = *many producers → one topic*.

Practical example with our tools: three producer windows each send `ERROR`/`INFO` lines to `app-logs`; a Spark query counts errors:

```scala
val logs = raw.selectExpr("CAST(value AS STRING) AS line")

val errors = logs.filter($"line".contains("ERROR"))
  .groupBy().count()            // total ERROR lines so far
```

Use the **server name as the key** so each server's logs stay ordered within one partition.

---

### 8. Pattern 4 — Change Data Capture (CDC)

**Problem:** A business database (orders, customers) changes constantly. Other systems need those changes **without querying the database repeatedly**.

**Solution:** capture every insert, update and delete from the database's transaction log and publish each as an **event** to Kafka.

```text
Database ──▶ CDC tool (e.g. Debezium via Kafka Connect) ──▶ topic "db.orders"
                                                              │
                                              ┌───────────────┼──────────────┐
                                              ▼               ▼              ▼
                                         Spark job       Search index   Data warehouse
```

| Benefit | Explanation |
| --- | --- |
| Low load | No repeated "select everything" queries |
| Complete | Every change is captured, including deletes |
| Real time | Changes flow within seconds |

> **Exam point:** CDC turns database changes into a stream of events in Kafka. It is typically done with **Kafka Connect** connectors such as Debezium. (We only need the concept for the syllabus.)

---

### 9. Pattern 5 — Fan-out (one topic, many readers)

**Problem:** The same data is needed by several applications.

```text
                    ┌──▶ Spark query 1: revenue dashboard
topic "sales" ──────┼──▶ Spark query 2: fraud/big-order alerts
                    └──▶ Python consumer: archive to file
```

**Fan-out** = *one topic → many independent readers*. Each reader has its **own position**, so they never interfere (this is the "different groups" behaviour of Phase 2, Section 10).

**In Spark, fan-out means several queries, each with its own checkpoint:**

```scala
val q1 = revenue.writeStream.format("console").outputMode("complete")
  .option("checkpointLocation", "C:/sparkdata/checkpoints/fan1").start()

val q2 = alerts.writeStream.format("console").outputMode("append")
  .option("checkpointLocation", "C:/sparkdata/checkpoints/fan2").start()
```

Both queries read the topic independently. Spark does not use consumer groups for its offsets, but each checkpoint plays the role of the group's bookmark.

---

### 10. Pattern 6 — Filter and route

**Problem:** One incoming stream contains different kinds of records that must go to different places.

```text
                    ┌── quantity ≥ 3 ──▶ topic "sales-bulk"
topic "sales" ──▶ Spark ┤
                    └── quantity < 3 ──▶ topic "sales-single"
```

```scala
val bulk   = sales.filter($"quantity" >= 3)
val single = sales.filter($"quantity" <  3)

def toKafka(df: org.apache.spark.sql.DataFrame, topic: String, ckpt: String) =
  df.select(
      $"city".as("key"),
      to_json(struct($"order_id", $"product", $"city", $"quantity", $"price")).as("value"))
    .writeStream.format("kafka")
    .option("kafka.bootstrap.servers", "localhost:9092")
    .option("topic", topic)
    .option("checkpointLocation", ckpt)
    .start()

val qb = toKafka(bulk,   "sales-bulk",   "C:/sparkdata/checkpoints/bulk")
val qs = toKafka(single, "sales-single", "C:/sparkdata/checkpoints/single")
```

✅ Both `bulk` and `single` are valid streaming DataFrames (verified: `isStreaming = true`). One query per destination is the standard approach.

---

### 11. Pattern 7 — Enrichment (stream–static join)

**Problem:** Events carry only ids or short codes. We need to add information from a **small reference table** (product categories, department heads...).

```text
stream: sales events ──┐
                       ├── JOIN on product ──▶ enriched events
static: categories ────┘     (small lookup table)
```

```scala
val categories = Seq(
  ("Laptop", "Computing"),
  ("Tablet", "Computing"),
  ("Phone",  "Mobile")
).toDF("product", "category")                 // an ordinary static DataFrame

val enriched = sales.join(categories, "product")      // streaming JOIN static
```

✅ **Verified:** a streaming DataFrame can be joined with a static DataFrame using the same `join` syntax from Unit IV. In our test, a `rate` stream with a computed key was joined with a 3-row lookup table, and every key found its match.

⚠️ **Typical output**

```text
+-------+--------+------+--------+-----+---------+
|product|order_id|  city|quantity|price| category|
+-------+--------+------+--------+-----+---------+
| Laptop|       1| Delhi|       2|55000|Computing|
|  Phone|       2|Mumbai|       5|20000|   Mobile|
+-------+--------+------+--------+-----+---------+
```

**Join type matters**

| Join | Behaviour for an unknown product |
| --- | --- |
| `inner` (default) | The event is **dropped** |
| `left` (`join(categories, Seq("product"), "left")`) | The event is **kept** with a NULL category |

> Use a **left join** if losing events is worse than having missing enrichment.

**Rules**

- The static table is read when the query **starts**; it is not refreshed automatically.
- Stream–**stream** joins also exist (with watermarks) but are an advanced topic.

---

### 12. Pattern 8 — Windowed aggregation

**Problem:** Produce **live metrics** such as "revenue per city in the last 30 seconds", updated continuously.

```scala
val metrics = sales2
  .withWatermark("timestamp", "1 minute")
  .groupBy(window($"timestamp", "30 seconds"), $"city")
  .agg(sum($"quantity" * $"price").as("revenue"))
```

(`sales2` keeps the Kafka `timestamp` column, Phase 7, Section 10.)

✅ **Verified window result** (Phase 6, Section 16): grouping by `window($"ts","10 minutes")` and `city` produced one row per window and city:

```text
|win_start          |win_end            |city  |total|
|2026-11-18 10:00:00|2026-11-18 10:10:00|Delhi |150  |
|2026-11-18 10:00:00|2026-11-18 10:10:00|Mumbai|30   |
|2026-11-18 10:10:00|2026-11-18 10:20:00|Delhi |70   |
```

**Use cases:** live dashboards, rate monitoring ("errors per minute"), anomaly detection.

---

### 13. Pattern 9 — Dead-letter queue (DLQ)

**Problem:** Some messages are malformed. If the pipeline crashes on them, **one bad record blocks everything**. If it silently drops them, we **lose** them without knowing.

**Solution:** separate valid from invalid records, process the valid ones, and store the invalid ones in a **dead-letter topic** for inspection.

```text
                      ┌── valid ──▶ continue pipeline
topic "sales-raw" ──▶ Spark ┤
                      └── invalid ──▶ topic "sales-dlq"  (inspect and fix later)
```

**Code**

```scala
val jsonSchema = new StructType()
  .add("order_id", IntegerType).add("product", StringType).add("quantity", IntegerType)

val parsed = raw.select(
  $"value".cast("string").as("raw"),
  from_json($"value".cast("string"), jsonSchema).as("d"))

val good = parsed.filter($"d.order_id".isNotNull && $"d.quantity".isNotNull).select("d.*")
val bad  = parsed.filter($"d.order_id".isNull  || $"d.quantity".isNull).select($"raw".as("value"))
```

✅ **Verified on three test messages** (a good one, one with `"quantity":"five"`, and the text `not json at all`):

```text
good:
+--------+-------+--------+
|order_id|product|quantity|
+--------+-------+--------+
|       1| Laptop|       2|
+--------+-------+--------+

bad (kept for the DLQ):
+--------------------------------------------------+
|raw                                               |
+--------------------------------------------------+
|{"order_id":2,"product":"Phone","quantity":"five"}|
|not json at all                                   |
+--------------------------------------------------+
```

**Send the bad records to the DLQ topic**

```scala
val qBad = bad.writeStream.format("kafka")
  .option("kafka.bootstrap.servers", "localhost:9092")
  .option("topic", "sales-dlq")
  .option("checkpointLocation", "C:/sparkdata/checkpoints/dlq")
  .start()
```

> **Reminder (Phase 7, Section 9):** garbage text gives a struct of NULL *fields*, not a NULL struct, so test the **fields** (`d.order_id`), not `d` alone.

---

### 14. Pattern 10 — De-duplication

**Problem:** At-least-once delivery (retries, replays, Kafka sink) can produce **duplicate** events. Counting them twice gives wrong totals.

**Solution:** drop repeated events using a unique id.

```scala
val dedup = sales.dropDuplicates("order_id")
```

✅ **Verified with a file stream:** file `a.csv` had orders 1 and 2; later file `b.csv` repeated orders 2 and 1 and added order 3.

```text
Batch: 0
+--------+-------+--------+
|order_id|product|quantity|
+--------+-------+--------+
|       2|  Phone|       5|
|       1| Laptop|       2|
+--------+-------+--------+

Batch: 1                         ← duplicates 2 and 1 were dropped
+--------+-------+--------+
|order_id|product|quantity|
+--------+-------+--------+
|       3| Tablet|       1|
+--------+-------+--------+
```

Spark remembers the ids it has already seen (**state**) and passes only new ones.

> **Memory warning:** remembering every id forever grows without limit. In production combine it with a **watermark** on an event-time column so old ids are forgotten:
> `df.withWatermark("event_time", "10 minutes").dropDuplicates("order_id", "event_time")`.

---

### 15. Pattern 11 — Replay and reprocessing

**Problem:** We fixed a bug in our processing logic and want to recompute results for the last week. Or a new team needs historical data.

**Solution:** Kafka **kept the events** (Phase 2, Section 12), so simply read them again.

```scala
val replay = spark.readStream.format("kafka")
  .option("kafka.bootstrap.servers", "localhost:9092")
  .option("subscribe", "sales-raw")
  .option("startingOffsets", "earliest")           // from the beginning
  .load()
// ...use a NEW checkpoint folder...
```

**Steps**

1. Stop the old query.
2. Fix the code.
3. Start it with a **new checkpoint folder** and `startingOffsets=earliest`.
4. Write results to a **new** output location, then switch consumers over.

> **This is the superpower of an event log:** the raw events are the source of truth; any result can be recomputed from them. It only works if the raw topic's **retention** is long enough.

---

### 16. Lambda vs Kappa architecture

Two classic designs for combining historical and real-time processing.

**Lambda architecture: two paths**

```text
                  ┌──▶ BATCH layer (Spark batch, accurate, slow) ──┐
data ──▶ Kafka ───┤                                                ├──▶ serving layer (merged view)
                  └──▶ SPEED layer (Spark Streaming, fast) ────────┘
```

| Aspect | Detail |
| --- | --- |
| Idea | Accurate-but-slow batch results + fast-but-approximate streaming results |
| Drawback | **Two code bases** to build and keep consistent |

**Kappa architecture: one path**

```text
data ──▶ Kafka (long retention) ──▶ ONE streaming job ──▶ serving layer
              └─ reprocess by replaying the log
```

| Aspect | Detail |
| --- | --- |
| Idea | Use only the **streaming** engine. For reprocessing, replay Kafka from the start |
| Benefit | **One code base**; simpler |
| Requirement | Kafka must retain data long enough; engine must handle replay |

| | Lambda | Kappa |
| --- | --- | --- |
| Layers | Batch + speed + serving | Streaming + serving |
| Code bases | Two | One |
| Reprocessing | Batch job | Replay the log |
| Fits Spark + Kafka? | Possible | **Natural fit** (Structured Streaming handles both batch and stream code) |

---

### 17. The wider Kafka ecosystem

| Component | What it is | When to use |
| --- | --- | --- |
| **Kafka Connect** | Framework of ready-made **connectors** that move data between Kafka and other systems (databases, files, cloud storage) without writing code | Ingestion (source connectors) and export (sink connectors); CDC |
| **Kafka Streams** | A **Java/Scala library** (not a cluster) for stream processing inside your own application | Lightweight processing without Spark |
| **ksqlDB** | SQL on Kafka streams (a separate product) | SQL-style stream processing |
| **Schema Registry** | Stores message schemas (Avro/Protobuf/JSON) and checks compatibility (a separate product) | Controlled schema evolution |
| **Spark Structured Streaming** | Distributed engine for heavy analytics, ML, joins, windows | Large-scale processing (our choice) |

```text
Kafka Connect (in)  ──▶  KAFKA  ──▶  Spark / Kafka Streams  ──▶  Kafka Connect (out)
```

> **Exam point:** Kafka Connect = integration (move data in/out), Kafka Streams = processing library, Spark = distributed processing engine.

---

### 18. Reliability checklist

| Area | Good practice |
| --- | --- |
| **Producer** | `acks=all`, `enable.idempotence=true`, retries on |
| **Topic** | Replication factor 3 and `min.insync.replicas=2` (production) |
| **Keys** | Choose a key that spreads load and keeps related events ordered |
| **Partitions** | Plan for the parallelism you need; adding later changes key-to-partition mapping |
| **Retention** | Long enough for replay and recovery |
| **Spark** | `checkpointLocation` on every query; one folder per query |
| **Duplicates** | Make downstream processing idempotent; use `order_id` for de-duplication |
| **Bad data** | Dead-letter queue instead of crash or silent drop |
| **Backpressure** | `maxOffsetsPerTrigger` and sensible trigger intervals |
| **Schema** | Version your message format; add fields rather than change meanings |
| **Monitoring** | Watch consumer lag, batch duration (Spark UI), input rate vs processing rate |

**The key health signal in Spark**

```scala
q.lastProgress
```

Compare **`inputRowsPerSecond`** (arrival) with **`processedRowsPerSecond`** (processing). If processing is consistently slower, the pipeline falls behind and lag grows.

---

### 19. A complete pipeline design

**Business problem:** An online shop wants (1) live revenue per product, (2) alerts for orders of ₹1,00,000 or more, and (3) no lost or silently ignored bad records.

```text
Python sales_generator ─▶ [sales-raw]
                              │
                              ▼  Spark Query A: parse + validate
                 ┌────────────┴─────────────┐
           valid │                          │ invalid
                 ▼                          ▼
          [sales-clean]               [sales-dlq]
                 │
      ┌──────────┼────────────────────┐
      ▼          ▼                    ▼
 Query B:    Query C:             Query D:
 enrich +    filter revenue       save clean data
 revenue/    ≥ 100000             as CSV files
 product      │
 (console)    ▼
          [sales-alerts]
```

| Component | Pattern used |
| --- | --- |
| `sales-raw` | Real-time ingestion; raw copy for replay |
| Query A | Validation + **dead-letter queue** |
| Query B | **Enrichment** + aggregation |
| Query C | **Filter and route** |
| Query D | Archive to files |
| One topic, many queries | **Fan-out** |

We build this in **Phase 9**.

---

### 20. Common pipeline mistakes

| Mistake | Consequence | Better approach |
| --- | --- | --- |
| Processing and overwriting the raw topic | Original data lost, no replay | Keep `*-raw` untouched; write to new topics |
| No dead-letter handling | Crash or silent data loss | Validate and route bad records |
| Reusing a checkpoint for a changed query | Failure or wrong results | New checkpoint per query/version |
| Forgetting retention | Cannot replay or recover | Set retention to match recovery needs |
| One giant topic for everything | Hard to manage and scale | A topic per data type and stage |
| Hot key (one key with most traffic) | One partition overloaded | Choose a well-distributed key |
| Assuming exactly-once everywhere | Duplicates in the Kafka sink | De-duplicate downstream |
| Ignoring lag/processing rate | Pipeline silently falls behind | Monitor continuously |
| Parsing without validation | NULLs everywhere downstream | Validate fields, then route |
| Resetting Kafka but keeping old checkpoints | Offset/data-loss errors | Delete checkpoints together with Kafka data |

---

### 21. Practical scenarios — Phase 8 practice

1. **Batch ingestion:** load `sales.txt` into topic `sales` with `file_to_kafka.py`; verify with the console consumer.
2. **Real-time ingestion:** run `sales_generator.py` and confirm one event per second.
3. **Fan-in:** run two generator copies (different cities) into the same topic.
4. **Fan-out:** start two Spark queries (revenue and alerts) on the same topic with separate checkpoints.
5. **Filter and route:** write big and small orders to `sales-bulk` and `sales-single`.
6. **Enrichment:** join the sales stream with a product-category table (inner, then left join).
7. **Windowed metric:** revenue per city per 30 seconds with a 1-minute watermark.
8. **Dead-letter queue:** send a bad message (`not json`) and confirm it lands in `sales-dlq`.
9. **De-duplication:** send the same `order_id` twice; show it is counted once.
10. **Replay:** restart a query with a new checkpoint and `startingOffsets=earliest`.
11. **Draw** the pipeline for a ride-hailing app (ride requests → matching → alerts), naming the pattern at each step.
12. **Explain** Lambda vs Kappa for a 2-minute viva answer.

---

### 22. Phase 8 — viva questions and answers

#### Q1. What is a data pipeline?

A series of steps that move data from its sources to where it is used, transforming it along the way.

#### Q2. What are the stages of a Kafka-based pipeline?

Source, ingestion, transport/buffer (Kafka), processing (Spark), storage/serving, and monitoring.

#### Q3. What is data ingestion?

Collecting data from sources and bringing it into a system for storage or processing.

#### Q4. What are the styles of ingestion?

Batch, micro-batch and real-time (streaming).

#### Q5. Why use Kafka between producers and Spark?

It decouples them, buffers spikes, stores events for replay and lets many consumers read the same data.

#### Q6. What is fan-in and fan-out?

Fan-in: many producers write to one topic. Fan-out: one topic is read independently by many consumers or queries.

#### Q7. What is Change Data Capture?

Capturing every change (insert, update, delete) in a database from its log and publishing it as events to Kafka, usually via Kafka Connect and Debezium.

#### Q8. What is a dead-letter queue?

A separate topic where records that fail validation are stored, so the pipeline continues and bad data can be inspected later.

#### Q9. How do you detect bad JSON records in Spark?

Parse with `from_json` and check that required fields are not NULL, since garbage input produces a struct of NULL fields.

#### Q10. What is stream–static enrichment?

Joining a streaming DataFrame with a small static reference table to add information such as categories.

#### Q11. What is the difference between an inner and a left join in enrichment?

Inner drops events with no match; left keeps them with NULLs for the enriched columns.

#### Q12. How do you remove duplicates in a stream?

`dropDuplicates("id")`, ideally with a watermark so old state can be discarded.

#### Q13. Why can duplicates appear in a pipeline?

At-least-once delivery: producer retries, replays and the Kafka sink can deliver a record more than once.

#### Q14. How can you reprocess historical data?

Start a query with `startingOffsets=earliest` and a **new checkpoint**, because Kafka retains events.

#### Q15. What is Lambda architecture?

Two processing paths: a batch layer for accurate results and a speed layer for fast results, merged by a serving layer.

#### Q16. What is Kappa architecture?

A single streaming path; reprocessing is done by replaying the Kafka log, so only one code base is needed.

#### Q17. What is Kafka Connect?

A framework of ready-made connectors that move data between Kafka and external systems such as databases and files.

#### Q18. What is Kafka Streams?

A Java/Scala client library for stream processing inside an application, without a separate processing cluster.

#### Q19. Why keep a raw topic unchanged?

So the original data can be replayed and results recomputed after fixing bugs.

#### Q20. How do you know a Spark streaming pipeline is falling behind?

`inputRowsPerSecond` stays higher than `processedRowsPerSecond`, batch durations grow and consumer lag rises.

#### Q21. What are the components of a safe producer configuration?

`acks=all`, `enable.idempotence=true` and retries, together with replication factor 3 and `min.insync.replicas=2`.

#### Q22. Why do we use a separate topic for each pipeline stage?

Stages can be scaled, debugged and replayed independently, and consumers pick the stage they need.

---

### 23. Phase 8 — final master recap

```text
KAFKA PIPELINE
  source → ingestion → KAFKA (transport) → SPARK (process) → store/serve → monitor

INGESTION STYLES
  batch · micro-batch · real-time

PATTERNS
  1 Batch ingestion      file → producer → topic
  2 Real-time ingestion  app → producer → topic
  3 Log aggregation      many servers → one topic (fan-in)
  4 CDC                  database changes → events (Kafka Connect/Debezium)
  5 Fan-out              one topic → many independent readers
  6 Filter and route     one stream → many topics
  7 Enrichment           stream JOIN static lookup
  8 Windowed aggregation live metrics (window + watermark)
  9 Dead-letter queue    bad records → sales-dlq
 10 De-duplication       dropDuplicates("id")
 11 Replay               new checkpoint + earliest

ARCHITECTURES
  Lambda = batch + speed (two code bases)   |   Kappa = one stream path + replay
```

**Seven facts to memorise**

1. Keep a **raw** topic untouched so you can replay.
2. One topic per stage; one checkpoint per query.
3. Fan-in = many → one; fan-out = one → many.
4. Bad records go to a **dead-letter** topic, not the bin.
5. The Kafka sink is **at-least-once**; de-duplicate downstream.
6. Kappa replays the log; Lambda keeps separate batch and speed layers.
7. Kafka Connect moves data, Kafka Streams processes inside apps, Spark processes at scale.

### Phase 8 completed

---

> **Next → Phase 9: Complete Practical Integration** (end-to-end projects with all windows, step-by-step lab routines, ten exam-style scenarios, the *question → pattern* method, troubleshooting). File: `9_Unit5_Complete_Practical_Integration.md`
