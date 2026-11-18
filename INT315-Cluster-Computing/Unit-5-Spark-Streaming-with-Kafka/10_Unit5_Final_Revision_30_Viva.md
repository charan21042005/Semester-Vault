# Unit V — Phase 10: Final Revision

> **Final phase of Unit V.** This is exam mode, not another teaching pass. It contains the syllabus coverage map, the one-page mental model, cheat sheets for Kafka and for Spark Streaming, a combined error table, **exactly 30 viva questions with answers**, and a last-10-minute sheet.
>
> **How the notes were verified:** all Spark-side code (DStreams, Structured Streaming, output modes, windows, parsing and validating Kafka-style messages, de-duplication, stream–static joins) was run in a real `spark-shell`. Kafka broker commands follow the official Kafka 3.4.0 syntax (a broker could not be run in my environment), so Kafka console outputs are *typical outputs*.

---

## 1. Syllabus coverage map

Your Unit V syllabus line:

> *Spark Streaming with Apache Kafka: Fundamentals of Apache Kafka, Apache Kafka Cluster Architecture, Apache Kafka Installation, Integration of Apache Kafka with Spark, producer & consumer messaging model, Kafka pipeline, ingestion patterns.*

| Syllabus topic | Where it is covered |
| --- | --- |
| Fundamentals of Apache Kafka | **Phase 1** (why Kafka, messaging models), **Phase 2** (topics, partitions, offsets, groups) |
| Apache Kafka cluster architecture | **Phase 3** (brokers, replication, ISR, controller, ZooKeeper/KRaft, storage) |
| Apache Kafka installation | **Phase 4** (Windows install, configuration, start-up, connector setup) |
| Producer & consumer messaging model | **Phase 5** (CLI, Python, Scala; keys, groups, offsets, lag) |
| Spark Streaming (title of the unit) | **Phase 6** (DStreams and Structured Streaming) |
| Integration of Kafka with Spark | **Phase 7** (connector, `readStream`/`writeStream`, parsing, offsets) |
| Kafka pipeline | **Phase 8** (pipeline stages and architecture) |
| Ingestion patterns | **Phase 8** (eleven patterns, Lambda vs Kappa) |
| Practical list: Spark Streaming; Kafka producer & consumer | **Phases 5, 6, 9** |
| Revision and viva | **Phase 10** (this file) |

**File index**

| File | Content |
| --- | --- |
| `1_Unit5_Streaming_and_Kafka_Introduction.md` | Phase 1 |
| `2_Unit5_Kafka_Core_Concepts.md` | Phase 2 |
| `3_Unit5_Kafka_Cluster_Architecture.md` | Phase 3 |
| `4_Unit5_Kafka_Installation_Windows.md` | Phase 4 |
| `5_Unit5_Producer_Consumer_Messaging_Model.md` | Phase 5 |
| `6_Unit5_Spark_Streaming_Foundations.md` | Phase 6 |
| `7_Unit5_Kafka_Spark_Integration.md` | Phase 7 |
| `8_Unit5_Kafka_Pipeline_Ingestion_Patterns.md` | Phase 8 |
| `9_Unit5_Complete_Practical_Integration.md` | Phase 9 |
| `10_Unit5_Final_Revision_30_Viva.md` | Phase 10 |

---

## 2. Your environment (one table to remember)

| Tool | Version | Used for |
| --- | --- | --- |
| Java | 1.8.0_501 | Runs Kafka, ZooKeeper, Spark |
| Python | 3.10.11 | `kafka-python` producers/consumers, socket server, PySpark |
| Kafka | 3.4.0 | Broker and console tools (`kafka_2.13-3.4.0`) |
| Spark | 3.5.9 | Structured Streaming |
| Scala | 2.13.8 | Decides the `_2.13` connector suffix |

**The one connector line**

```text
spark-shell --packages org.apache.spark:spark-sql-kafka-0-10_2.13:3.5.9
```

---

## 3. The Unit V mental model

```text
 PRODUCERS            KAFKA CLUSTER (brokers + ZooKeeper)              SPARK
┌──────────┐        ┌──────────────────────────────────────┐      ┌─────────────────┐
│ console  │        │ topic "sales"                        │      │ readStream      │
│ Python   │─write─▶│  P0 [0][1][2]   leader + followers   │─read▶│  .format(kafka) │
│ Scala    │        │  P1 [0][1]      replicated (ISR)     │      │ cast → parse    │
│ files    │        │  P2 [0]         retained 7 days      │      │ aggregate/filter│
└──────────┘        └──────────────────────────────────────┘      │ writeStream ────┼─▶ console / files /
                                                                  │ + checkpoint    │   another topic
                                                                  └─────────────────┘
```

**One-line summary**

> *Producers write events into partitioned, replicated Kafka topics; Spark Structured Streaming reads them as an unbounded DataFrame, processes them in micro-batches with checkpointed exactly-once recovery, and writes results to files, the console or another topic.*

---

## 4. Kafka cheat sheet

### 4.1 Core concepts

| Term | Meaning |
| --- | --- |
| Record | key + value + timestamp (+ headers), stored as bytes |
| Topic | Named stream, split into partitions |
| Partition | Ordered, append-only log; unit of parallelism |
| Offset | Position number in a partition (starts at 0, per partition) |
| Producer | Writes records; key → partition via hash (same key, same partition) |
| Consumer | Pulls records, remembers offset |
| Consumer group | Consumers sharing work: one partition → one consumer per group |
| Broker | Kafka server (`broker.id`, port 9092) |
| Cluster | Several brokers; clients start from `bootstrap.servers` |
| Leader / follower | Leader serves reads and writes; followers copy |
| ISR | Replicas in sync; new leader is chosen from ISR |
| Controller | One broker that elects leaders |
| ZooKeeper | Metadata and liveness (port 2181); KRaft is its replacement |
| `__consumer_offsets` | Internal topic storing group offsets |
| Retention | Records kept (default 7 days), not deleted when read |

### 4.2 Guarantees to remember

| Point | Fact |
| --- | --- |
| Ordering | Only within a partition |
| Same key | Same partition (while partition count is unchanged) |
| Max useful consumers in a group | Number of partitions |
| Replication factor | ≤ number of brokers |
| Safe producer | `acks=all`, `enable.idempotence=true`, RF 3, `min.insync.replicas=2` |
| Commit after processing | At-least-once |
| Commit before processing | At-most-once |

### 4.3 Windows commands

```powershell
REM ---- START (ZooKeeper first) ----
bin\windows\zookeeper-server-start.bat config\zookeeper.properties
bin\windows\kafka-server-start.bat config\server.properties
jps

REM ---- TOPICS ----
kafka-topics.bat --version
kafka-topics.bat --create   --topic T --bootstrap-server localhost:9092 --partitions 3 --replication-factor 1
kafka-topics.bat --list     --bootstrap-server localhost:9092
kafka-topics.bat --describe --topic T --bootstrap-server localhost:9092
kafka-topics.bat --alter    --topic T --partitions 4 --bootstrap-server localhost:9092

REM ---- PRODUCER ----
kafka-console-producer.bat --topic T --bootstrap-server localhost:9092
kafka-console-producer.bat --topic T --bootstrap-server localhost:9092 --property parse.key=true --property key.separator=:

REM ---- CONSUMER ----
kafka-console-consumer.bat --topic T --from-beginning --bootstrap-server localhost:9092
kafka-console-consumer.bat --topic T --group G --bootstrap-server localhost:9092 --property print.key=true --property print.partition=true --property print.offset=true
kafka-console-consumer.bat --topic T --partition 0 --offset 1 --bootstrap-server localhost:9092 --max-messages 2

REM ---- GROUPS ----
kafka-consumer-groups.bat --bootstrap-server localhost:9092 --list
kafka-consumer-groups.bat --bootstrap-server localhost:9092 --describe --group G
kafka-consumer-groups.bat --bootstrap-server localhost:9092 --group G --topic T --reset-offsets --to-earliest --execute

REM ---- TOPIC CONFIG ----
kafka-configs.bat --bootstrap-server localhost:9092 --entity-type topics --entity-name T --alter --add-config retention.ms=3600000
```

### 4.4 Key configuration properties

| File | Property | Value used |
| --- | --- | --- |
| `zookeeper.properties` | `dataDir` | `C:/kafka/data/zookeeper` |
| `server.properties` | `broker.id` | `0` |
| `server.properties` | `listeners` | `PLAINTEXT://localhost:9092` |
| `server.properties` | `log.dirs` | `C:/kafka/data/kafka-logs` |
| `server.properties` | `zookeeper.connect` | `localhost:2181` |
| `server.properties` | `offsets.topic.replication.factor` | `1` (single broker) |
| `server.properties` | `log.retention.hours` | `168` |

### 4.5 Python (kafka-python)

```python
from kafka import KafkaProducer, KafkaConsumer

p = KafkaProducer(bootstrap_servers="localhost:9092",
                  key_serializer=lambda k: k.encode(), value_serializer=lambda v: v.encode())
p.send("T", key="Delhi", value="Laptop,2").get(timeout=10)
p.flush(); p.close()

c = KafkaConsumer("T", bootstrap_servers="localhost:9092", group_id="g1",
                  auto_offset_reset="earliest", value_deserializer=lambda v: v.decode(),
                  consumer_timeout_ms=5000)
for m in c: print(m.partition, m.offset, m.value)
```

---

## 5. Spark Streaming cheat sheet

### 5.1 DStreams (legacy)

```scala
import org.apache.spark.streaming.{StreamingContext, Seconds}
val ssc = new StreamingContext(sc, Seconds(2))
val lines  = ssc.socketTextStream("localhost", 9999)
val counts = lines.flatMap(_.split(" ")).map((_, 1)).reduceByKey(_ + _)
counts.print()
ssc.start(); ssc.awaitTerminationOrTimeout(30000); ssc.stop(stopSparkContext = false)
```

| Operation | Purpose |
| --- | --- |
| `updateStateByKey` | Running state across batches (needs `ssc.checkpoint`) |
| `reduceByKeyAndWindow(f, window, slide)` | Last N seconds |
| One `StreamingContext` per JVM; a stopped one cannot restart | |

### 5.2 Structured Streaming recipe

```text
readStream  →  transform (DataFrame / SQL)  →  writeStream  →  start()
```

| Piece | Syntax |
| --- | --- |
| Sources | `rate`, `socket`, `csv/json/parquet` (needs schema), **`kafka`** |
| Sinks | `console`, `memory`, `csv/json/parquet`, **`kafka`**, `foreachBatch` |
| Output modes | `append` (new rows), `update` (changed rows), `complete` (whole table) |
| Trigger | `Trigger.ProcessingTime("5 seconds")`, `Trigger.AvailableNow()` |
| Checkpoint | `.option("checkpointLocation", "C:/sparkdata/checkpoints/q1")` |
| Event-time window | `window($"ts", "10 minutes")` |
| Watermark | `.withWatermark("ts", "10 minutes")` |
| Manage | `q.status`, `q.lastProgress`, `q.stop()`, `spark.streams.active` |

### 5.3 Kafka source and sink

```scala
// READ
val raw = spark.readStream.format("kafka")
  .option("kafka.bootstrap.servers", "localhost:9092")
  .option("subscribe", "T")
  .option("startingOffsets", "earliest")
  .load()

// Kafka columns: key value topic partition offset timestamp timestampType  (key/value = BINARY)
val text = raw.selectExpr("CAST(value AS STRING) AS value")

// PARSE CSV
val p = split($"value", ",")
text.select(p.getItem(0).cast("int").as("id"), p.getItem(1).as("name"))

// PARSE JSON
text.select(from_json($"value", schema).as("d")).select("d.*")

// WRITE to Kafka (needs a "value" column; "key" optional)
df.select($"k".as("key"), to_json(struct($"a", $"b")).as("value"))
  .writeStream.format("kafka")
  .option("kafka.bootstrap.servers", "localhost:9092")
  .option("topic", "out")
  .option("checkpointLocation", "C:/sparkdata/checkpoints/out")
  .start()
```

| Source option | Meaning |
| --- | --- |
| `subscribe` / `subscribePattern` / `assign` | Exactly one is required |
| `startingOffsets` | `earliest` / `latest` / JSON; **first run only** |
| `endingOffsets` | Batch reads only |
| `maxOffsetsPerTrigger` | Rate limit per batch |
| `failOnDataLoss` | Fail if expected data is gone |

### 5.4 Pipeline patterns in one line each

| Pattern | Code idea |
| --- | --- |
| Dead-letter queue | `bad = parsed.filter(field.isNull)` → write to `*-dlq` topic |
| De-duplication | `dropDuplicates("order_id")` |
| Enrichment | `stream.join(staticDF, "key")` (use `left` to keep unmatched) |
| Filter and route | One query per destination topic |
| Fan-out | Several queries, one checkpoint each |
| Windowed metric | `withWatermark` + `groupBy(window(...))` |
| Replay | New checkpoint + `startingOffsets=earliest` |

---

## 6. Combined error table

| Symptom | Cause | Fix |
| --- | --- | --- |
| `'kafka-topics.bat' is not recognized` | PATH | Add `C:\kafka\bin\windows` |
| *The input line is too long* | Long install path | Install in `C:\kafka` |
| `Timed out waiting for connection to ZooKeeper` | ZooKeeper not started | Start ZooKeeper first |
| `Address already in use` | Old process on 2181/9092 | Stop it (`taskkill`) |
| `InconsistentClusterIdException` | Only one data folder deleted | Delete both data folders |
| Broker crashes after topic delete | Windows file locking | Never delete topics on Windows |
| Replication factor error | RF > brokers | Use RF 1 |
| Consumer prints nothing | No `--from-beginning`, or group already committed | Add flag / new group |
| Producer exits, records missing | No `flush()` | `producer.flush()` |
| `Failed to find data source: kafka` | Connector missing | `--packages ...spark-sql-kafka-0-10_2.13:3.5.9` |
| `NoSuchMethodError` at start | Scala/Spark version mismatch | `_2.13` and `3.5.9` |
| Multi-line chain: only first line runs | Typed line by line | `:paste`, Ctrl + D |
| Hex bytes in output | Binary not cast | `CAST(value AS STRING)` |
| Parsed columns NULL | Wrong delimiter/schema | Inspect the raw string |
| `Append output mode not supported` (aggregation) | Wrong mode | `complete` / `update` |
| `Queries with streaming sources must be executed with writeStream.start()` | `show()` on a stream | Use `writeStream` |
| Offset / data-loss error | Kafka reset, old checkpoint | Delete the checkpoint |
| `Connection refused` on socket source | Server not started | Start the Python server first |
| Spark slow on laptop | 200 shuffle partitions | `spark.sql.shuffle.partitions=4` |
| Duplicates in the output topic | Kafka sink is at-least-once | De-duplicate downstream |

---

## 7. Exactly 30 viva questions with answers

#### Q1. What is stream processing, and how does it differ from batch processing?

Stream processing handles data continuously as it arrives (low latency, endless data). Batch processing handles a finite, already collected dataset at intervals (higher latency).

#### Q2. Why do we need a messaging system between producers and consumers, and what are the two messaging models?

It decouples systems, buffers speed differences, prevents data loss and reduces connections from M × N to M + N. The models are **point-to-point** (each message goes to one consumer) and **publish-subscribe** (every subscriber receives every message).

#### Q3. What is Apache Kafka and what are its main features?

A distributed, fault-tolerant, high-throughput event streaming platform that stores streams of records durably. Features: high throughput, scalability, durability, replication-based fault tolerance, decoupling, replayability and multiple independent consumers.

#### Q4. How is Kafka different from a traditional message queue?

Kafka keeps messages after they are read (until retention expires), allows replay, scales through partitions, and consumers **pull** at their own pace. Traditional queues usually delete messages once consumed.

#### Q5. Explain topic, partition and offset.

A topic is a named stream of records. It is divided into partitions, each an ordered append-only log. An offset is the sequential position number of a record within a partition, starting at 0; (topic, partition, offset) identifies a record uniquely.

#### Q6. What ordering does Kafka guarantee, and how do keys help?

Order is guaranteed only within a single partition. Records with the same key go to the same partition (`hash(key) mod partitions`), so their relative order is preserved.

#### Q7. How does a producer decide the partition?

With a key: hash of the key modulo the number of partitions. Without a key: records are spread across partitions. A partition can also be specified explicitly.

#### Q8. What is a consumer group, and how does it provide both queue and pub-sub behaviour?

A set of consumers with the same `group.id`; each partition is read by only one member (queue/work sharing). Different groups each receive all records (pub-sub), with independent offsets.

#### Q9. What is retention, and why can Kafka replay data?

Records are kept for a configured time (default 7 days) or size, regardless of consumption. A consumer can therefore re-read from `earliest` or any offset.

#### Q10. What are a broker, a cluster and bootstrap servers?

A broker is one Kafka server; a cluster is several brokers working together; bootstrap servers are one or more entry-point brokers a client contacts to discover the full cluster metadata.

#### Q11. Explain replication, leader, follower, replication factor and ISR.

Each partition has copies (replication factor ≤ brokers). One replica is the **leader** and handles all reads and writes; **followers** copy it. The **ISR** is the set of replicas fully in sync; only they can become the new leader.

#### Q12. What happens when a leader broker fails?

The controller elects a new leader from the ISR, clients refresh metadata and continue with no loss of committed data. The failed broker later rejoins as a follower and catches up.

#### Q13. What are the roles of ZooKeeper and KRaft?

ZooKeeper stores cluster metadata, tracks live brokers through heartbeats and helps elect the controller (it does not store messages; port 2181). KRaft is Kafka's built-in Raft-based replacement that removes ZooKeeper (the only mode from Kafka 4.0).

#### Q14. Explain `acks` and `min.insync.replicas`.

`acks=0`: no acknowledgement; `acks=1`: leader acknowledges; `acks=all`: all in-sync replicas acknowledge (safest). `min.insync.replicas` is the minimum ISR size needed for an `acks=all` write to succeed.

#### Q15. Where are consumer offsets stored, and what is a rebalance?

In the internal topic `__consumer_offsets`. A rebalance redistributes partitions among consumers when one joins, leaves or fails.

#### Q16. How does Kafka store data on disk, and why is it fast?

Each partition is a folder of segment files (`.log`, `.index`, `.timeindex`). It is fast because of sequential disk appends, the OS page cache, batching, zero-copy transfer and partition-level parallelism.

#### Q17. What are the delivery semantics?

At most once (may lose), at least once (may duplicate; commit after processing) and exactly once (idempotent producer and transactions, or an idempotent sink with checkpointed offsets).

#### Q18. What are the steps to install and start Kafka on Windows?

Install Java; extract Kafka to `C:\kafka`; set `dataDir` in `zookeeper.properties` and `log.dirs`, `listeners`, `zookeeper.connect` in `server.properties` (forward slashes); start ZooKeeper (port 2181) first, then the broker (port 9092); verify with `jps` (`QuorumPeerMain` and `Kafka`).

#### Q19. Write the commands to create a topic, send messages and read them.

`kafka-topics.bat --create --topic T --bootstrap-server localhost:9092 --partitions 3 --replication-factor 1`; `kafka-console-producer.bat --topic T --bootstrap-server localhost:9092`; `kafka-console-consumer.bat --topic T --from-beginning --bootstrap-server localhost:9092`.

#### Q20. What is consumer lag and how do you check it?

Lag = `LOG-END-OFFSET − CURRENT-OFFSET`, the number of unread records. Check with `kafka-consumer-groups.bat --bootstrap-server localhost:9092 --describe --group G`.

#### Q21. What is Spark Streaming and what is micro-batching?

Spark Streaming processes live data by cutting the stream into small time slices (micro-batches) and running each as a small batch job, which gives high throughput and easy fault recovery with sub-second to seconds latency.

#### Q22. What is a DStream, and how do DStreams differ from Structured Streaming?

A DStream is a sequence of RDDs, one per batch interval (RDD API, processing time, manual state, legacy). Structured Streaming treats the stream as an unbounded DataFrame (SQL/Catalyst, event time, watermarks, automatic state, end-to-end exactly-once) and is the recommended API.

#### Q23. Explain the output modes and why append is not allowed for a plain aggregation.

**Append** writes only new rows, **update** writes only changed rows, **complete** writes the whole result table. Aggregates keep changing, but append mode writes a row once and never revises it, so it is only allowed for aggregations with a watermark.

#### Q24. What are triggers and checkpoints?

A trigger decides when micro-batches run (default, `ProcessingTime`, `AvailableNow`). A checkpoint folder stores the query's offsets and state, enabling recovery after failure without loss or duplication.

#### Q25. What are event time, windows and watermarks?

Event time is when an event occurred (a data column). A window groups events into time buckets (for example 10 minutes). A watermark sets how late data may arrive, allowing Spark to drop old state and bound memory.

#### Q26. Which package connects Spark to Kafka in your setup, and why does `_2.13` matter?

`org.apache.spark:spark-sql-kafka-0-10_2.13:3.5.9`, started with `spark-shell --packages ...`. The `_2.13` suffix must match Spark's Scala version (2.13.8) and `3.5.9` must match Spark's version; otherwise you get `NoSuchMethodError` or `Failed to find data source: kafka`.

#### Q27. What is the schema of a Kafka DataFrame, and what must we do with `value`?

`key` (binary), `value` (binary), `topic`, `partition`, `offset`, `timestamp`, `timestampType`. Because key and value are binary, we `CAST(value AS STRING)` and then parse it with `split` (CSV) or `from_json` (JSON).

#### Q28. How do you write a Spark result to Kafka?

Use `writeStream.format("kafka")` with `kafka.bootstrap.servers`, `topic` and `checkpointLocation`. The DataFrame needs a `value` column (and optionally `key`), commonly built with `to_json(struct(...))`.

#### Q29. How does Spark track Kafka offsets, and what delivery guarantee results?

Spark records offsets per batch in its **checkpoint** (not in Kafka consumer groups). With a replayable Kafka source plus checkpointing, processing is exactly-once; the Kafka sink is at-least-once, so downstream de-duplication is advised. `startingOffsets` applies only to the first run.

#### Q30. What is a Kafka pipeline, and which ingestion patterns do you know?

A pipeline moves data from source through ingestion, Kafka (transport), Spark (processing) and storage/serving, with monitoring. Ingestion can be batch, micro-batch or real-time. Patterns: batch file ingestion, real-time ingestion, log aggregation (fan-in), CDC, fan-out, filter-and-route, enrichment (stream–static join), windowed aggregation, dead-letter queue, de-duplication and replay. Lambda architecture uses separate batch and speed layers; Kappa uses a single streaming path with replay of the Kafka log.

---

## 8. Last 10-minute sheet

**Ten things to remember**

1. Kafka **transports and stores**; Spark **processes**.
2. Topic → partitions (ordered logs); offset is **per partition**.
3. Order is guaranteed **only within a partition**; same key → same partition.
4. One partition → one consumer **per group**; different groups each get everything.
5. Replication: **leader** serves, followers copy; new leader comes from the **ISR**; RF ≤ brokers.
6. ZooKeeper (2181) **before** Kafka (9092); stop in reverse; never delete topics on Windows.
7. Connector: `spark-sql-kafka-0-10_2.13:3.5.9`; Kafka `key`/`value` are **binary → cast**.
8. Structured Streaming = `readStream` → transform → `writeStream` → `start()`; modes **append / update / complete**.
9. **Checkpoint** per query = offsets + state = recovery; delete it if Kafka data was reset.
10. Multi-line chains in `spark-shell` need **`:paste`** (Ctrl + D).

**Five-minute rescue commands**

```powershell
kafka-topics.bat --create --topic t --bootstrap-server localhost:9092 --partitions 1 --replication-factor 1
kafka-console-producer.bat --topic t --bootstrap-server localhost:9092
kafka-console-consumer.bat --topic t --from-beginning --bootstrap-server localhost:9092
spark-shell --packages org.apache.spark:spark-sql-kafka-0-10_2.13:3.5.9 --conf spark.sql.shuffle.partitions=4
```

```scala
val raw = spark.readStream.format("kafka").option("kafka.bootstrap.servers","localhost:9092").option("subscribe","t").option("startingOffsets","earliest").load()
raw.selectExpr("CAST(value AS STRING) AS v","partition","offset").writeStream.format("console").outputMode("append").start()
```

---

## 9. Unit V status

```text
Phase 1   Streaming and Kafka introduction              ✅
Phase 2   Kafka core concepts (+ Scala mini-Kafka)      ✅
Phase 3   Kafka cluster architecture                    ✅
Phase 4   Kafka installation on Windows                 ✅
Phase 5   Producer and consumer messaging model         ✅
Phase 6   Spark Streaming foundations (verified)        ✅
Phase 7   Kafka and Spark integration                   ✅
Phase 8   Kafka pipeline and ingestion patterns         ✅
Phase 9   Complete practical integration                ✅
Phase 10  Final revision + 30 viva                      ✅
```

## ✅ UNIT V — COMPLETE

> **Next → Unit VI: Spark ML Programming and PySpark** (machine learning basics, Spark MLlib, linear/logistic regression, decision tree, K-means, SVM, Naive Bayes, evaluation metrics such as confusion matrix, R², RMSE, MAE, correlation heat map). Your **Python 3.10.11** is already ready for it.
