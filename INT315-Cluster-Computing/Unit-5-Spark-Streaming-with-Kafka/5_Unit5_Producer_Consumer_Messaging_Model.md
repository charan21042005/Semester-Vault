# Unit V — Phase 5: Producer and Consumer Messaging Model

> **Continuing from Phase 4.** Kafka is installed and running. Now we **use** it: create topics, write messages with producers, read them with consumers, work with keys, consumer groups, offsets and lag. This covers the syllabus topic **producer & consumer messaging model** and your practical *"Demonstrate the use of producer and consumer in Apache Kafka"*.
>
> **Verification note:** The commands follow the official Kafka 3.4.0 console tools. Outputs marked ⚠️ **Typical output** are what Kafka normally prints. The Python API calls were checked against the `kafka-python` library. Run the commands yourself to see your exact output.

---

## Phase 5 — Roadmap

| # | Topic |
| --- | --- |
| 1 | The messaging model in one picture |
| 2 | Before we start: windows and startup |
| 3 | Topic management with `kafka-topics.bat` |
| 4 | Console producer |
| 5 | Console consumer |
| 6 | Messages with keys |
| 7 | Seeing partitions and offsets |
| 8 | Consumer groups in action |
| 9 | Inspecting groups: offsets and lag |
| 10 | Reading a specific partition or offset |
| 11 | Topic configuration (retention) |
| 12 | Producer in Python |
| 13 | Consumer in Python |
| 14 | A fake sales-event generator |
| 15 | Producer and consumer in Scala (spark-shell) |
| 16 | Important producer settings |
| 17 | Important consumer settings |
| 18 | Common errors |
| 19 | Practical scenarios |
| 20 | Viva questions |
| 21 | Master revision |

---

### 1. The messaging model in one picture

```text
  PRODUCER SIDE                       KAFKA                       CONSUMER SIDE
 ┌──────────────────┐         ┌────────────────────┐        ┌───────────────────┐
 │ 1. build record  │         │  topic "sales"     │        │ 1. subscribe      │
 │ 2. serialize     │──send──▶│  P0 [0][1][2][3]   │◀─poll──│ 2. poll records   │
 │ 3. pick partition│         │  P1 [0][1][2]      │        │ 3. deserialize    │
 │ 4. batch + send  │◀──ack───│  P2 [0][1]         │        │ 4. process        │
 └──────────────────┘         └────────────────────┘        │ 5. commit offset  │
                                                            └───────────────────┘
```

**Producer lifecycle**

1. Create a record (key optional, value required).
2. **Serialize** key and value into bytes.
3. **Partitioner** chooses a partition.
4. Records are **batched** per partition.
5. Sent to the leader broker; **acknowledgement** returned according to `acks`.

**Consumer lifecycle**

1. Join a **group** and **subscribe** to topic(s).
2. **Poll** the broker for new records (pull model).
3. **Deserialize** bytes into usable values.
4. **Process** the records.
5. **Commit** the offset (bookmark) so the position is remembered.

---

### 2. Before we start: windows and startup

Start your environment (Phase 4, Section 18):

```text
CMD #1  ZooKeeper     (leave running)
CMD #2  Kafka broker  (leave running)
CMD #3  commands      (topics, producer)
CMD #4  consumer
```

All commands below assume `C:\kafka\bin\windows` is in your PATH, and the broker is at `localhost:9092`.

---

### 3. Topic management with `kafka-topics.bat`

#### 3.1 Create a topic

```powershell
kafka-topics.bat --create --topic sales --bootstrap-server localhost:9092 --partitions 3 --replication-factor 1
```

⚠️ **Typical output**

```text
Created topic sales.
```

| Option | Meaning |
| --- | --- |
| `--create` | Create a topic |
| `--topic sales` | The topic name |
| `--bootstrap-server localhost:9092` | Which broker to talk to |
| `--partitions 3` | Number of partitions |
| `--replication-factor 1` | Copies (1 on a single broker) |

#### 3.2 List topics

```powershell
kafka-topics.bat --list --bootstrap-server localhost:9092
```

```text
sales
```

(Internal topics such as `__consumer_offsets` are hidden from the plain list or appear after groups are used.)

#### 3.3 Describe a topic

```powershell
kafka-topics.bat --describe --topic sales --bootstrap-server localhost:9092
```

⚠️ **Typical output**

```text
Topic: sales   TopicId: xYz...   PartitionCount: 3   ReplicationFactor: 1   Configs:
    Topic: sales   Partition: 0   Leader: 0   Replicas: 0   Isr: 0
    Topic: sales   Partition: 1   Leader: 0   Replicas: 0   Isr: 0
    Topic: sales   Partition: 2   Leader: 0   Replicas: 0   Isr: 0
```

You can read this exactly as taught in Phase 3, Section 15.

#### 3.4 Increase partitions (never decrease)

```powershell
kafka-topics.bat --alter --topic sales --partitions 4 --bootstrap-server localhost:9092
```

> ⚠️ Increasing partitions changes the `hash(key) mod N` result, so the same key may now go to a **different** partition. Existing records stay where they were.

#### 3.5 Delete a topic

```powershell
kafka-topics.bat --delete --topic sales --bootstrap-server localhost:9092
```

> ⚠️ **On Windows, avoid this.** Topic deletion can crash the broker because of file locking (Phase 4, Section 13). Use a new topic name instead.

**Topic command summary**

| Task | Option |
| --- | --- |
| Create | `--create --topic T --partitions N --replication-factor R` |
| List | `--list` |
| Describe | `--describe --topic T` |
| Add partitions | `--alter --topic T --partitions N` |
| Delete (avoid on Windows) | `--delete --topic T` |
| Version | `--version` |

---

### 4. Console producer

The **console producer** reads lines you type and sends each line as one record.

**CMD #3**

```powershell
kafka-console-producer.bat --topic sales --bootstrap-server localhost:9092
```

A `>` prompt appears. Type three records (value only, no key):

```text
>Laptop,2,55000
>Phone,5,20000
>Tablet,3,15000
```

- Each line + **Enter** = **one record**.
- Press **Ctrl + C** to exit.

> **Old tutorials** use `--broker-list`. In Kafka 3.4 use `--bootstrap-server` (the `--broker-list` option is deprecated).

---

### 5. Console consumer

**CMD #4**

```powershell
kafka-console-consumer.bat --topic sales --bootstrap-server localhost:9092
```

By default the consumer starts at the **latest** offset, so it shows only **new** records. It will appear empty until you send more from the producer.

**Read from the beginning**

```powershell
kafka-console-consumer.bat --topic sales --from-beginning --bootstrap-server localhost:9092
```

⚠️ **Typical output** (order across partitions may mix):

```text
Laptop,2,55000
Phone,5,20000
Tablet,3,15000
```

**Useful options**

| Option | Meaning |
| --- | --- |
| `--from-beginning` | Start at the earliest offset |
| `--max-messages 5` | Exit after 5 records |
| `--group mygroup` | Join a named consumer group |
| `--partition 0` | Read one partition only |
| `--offset 2` | Start from offset 2 (needs `--partition`) |
| `--timeout-ms 10000` | Exit if no record arrives for 10 s |
| `--property print.key=true` | Also show the key |
| `--property print.partition=true` | Show the partition |
| `--property print.offset=true` | Show the offset |
| `--property print.timestamp=true` | Show the timestamp |

**Live demo:** keep the consumer open and type more lines in the producer window. They appear within a moment. This is **real-time delivery**.

---

### 6. Messages with keys

To send **key + value**, tell the producer how to split each line.

```powershell
kafka-console-producer.bat --topic sales --bootstrap-server localhost:9092 --property parse.key=true --property key.separator=:
```

Type lines as `key:value`:

```text
>Delhi:Laptop,2
>Mumbai:Phone,5
>Delhi:Tablet,3
>Pune:Phone,1
>Mumbai:Laptop,1
```

**Consumer that prints keys and partitions**

```powershell
kafka-console-consumer.bat --topic sales --from-beginning --bootstrap-server localhost:9092 --property print.key=true --property print.partition=true --property key.separator=" | "
```

⚠️ **Typical output** (partition numbers depend on Kafka's hash):

```text
Partition:0 | Delhi | Laptop,2
Partition:0 | Delhi | Tablet,3
Partition:1 | Mumbai | Phone,5
Partition:1 | Mumbai | Laptop,1
Partition:2 | Pune | Phone,1
```

**What to observe**

- Records with the same key (`Delhi`) always show the **same partition**.
- Within a key, the **order is preserved** (Laptop before Tablet).
- Real Kafka uses the murmur2 hash, so exact partition numbers differ from our toy simulation in Phase 2, but the behaviour is the same.

---

### 7. Seeing partitions and offsets

Print everything about each record:

```powershell
kafka-console-consumer.bat --topic sales --from-beginning --bootstrap-server localhost:9092 --property print.key=true --property print.partition=true --property print.offset=true --property print.timestamp=true
```

⚠️ **Typical output**

```text
CreateTime:1795000000001  Partition:0  Offset:0  Delhi   Laptop,2
CreateTime:1795000000150  Partition:0  Offset:1  Delhi   Tablet,3
CreateTime:1795000000075  Partition:1  Offset:0  Mumbai  Phone,5
CreateTime:1795000000200  Partition:1  Offset:1  Mumbai  Laptop,1
CreateTime:1795000000120  Partition:2  Offset:0  Pune    Phone,1
```

This is the Phase 2 theory shown on your own screen:

| What you see | Concept |
| --- | --- |
| `Partition:0 Offset:0`, `Partition:0 Offset:1` | Offsets increase inside a partition |
| `Partition:1 Offset:0` | Offsets restart from 0 in every partition |
| `CreateTime` | The record timestamp (milliseconds since 1970) |

---

### 8. Consumer groups in action

We will prove both models from Phase 2, Section 10.

#### 8.1 Same group = work sharing (queue behaviour)

Open **two** consumer windows (CMD #4 and CMD #5), each running:

```powershell
kafka-console-consumer.bat --topic sales --group workers --bootstrap-server localhost:9092 --property print.partition=true
```

Now send many keyed records from the producer. Each consumer receives **only some partitions**:

```text
CMD #4 (consumer 1):  Partition:0 ...   Partition:1 ...
CMD #5 (consumer 2):  Partition:2 ...
```

Each record appears in **exactly one** of the two windows. They are *sharing* the topic.

#### 8.2 Different groups = broadcast (pub-sub behaviour)

Run the **same topic** with **different group ids**:

```powershell
REM CMD #4
kafka-console-consumer.bat --topic sales --group dashboard --bootstrap-server localhost:9092

REM CMD #5
kafka-console-consumer.bat --topic sales --group fraud --bootstrap-server localhost:9092
```

Send records: **both windows show every record.**

```text
workers group    → records are split between members
dashboard + fraud → each group receives ALL records
```

#### 8.3 Watching a rebalance

1. Start consumer 1 in group `workers`. It reads all 3 partitions.
2. Start consumer 2 in the same group. Kafka **rebalances**; the partitions are divided.
3. Press **Ctrl + C** on consumer 1. Kafka **rebalances again**; consumer 2 takes everything.

---

### 9. Inspecting groups: offsets and lag

`kafka-consumer-groups.bat` shows each group's bookmark and how far behind it is.

**List groups**

```powershell
kafka-consumer-groups.bat --bootstrap-server localhost:9092 --list
```

```text
workers
dashboard
fraud
```

**Describe one group**

```powershell
kafka-consumer-groups.bat --bootstrap-server localhost:9092 --describe --group workers
```

⚠️ **Typical output**

```text
GROUP    TOPIC  PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG  CONSUMER-ID ...
workers  sales  0          2               2               0    console-consumer-...
workers  sales  1          2               2               0    console-consumer-...
workers  sales  2          1               1               0    console-consumer-...
```

| Column | Meaning |
| --- | --- |
| `CURRENT-OFFSET` | The group's bookmark (next offset to read) |
| `LOG-END-OFFSET` | The offset after the last record in the partition |
| **`LAG`** | `LOG-END-OFFSET − CURRENT-OFFSET` = **unread records** |

```text
LAG = LOG-END-OFFSET − CURRENT-OFFSET
LAG 0   → consumer is fully caught up
LAG big → consumer is falling behind (slow consumer / too few partitions)
```

> **Lag is the number-one health metric of a Kafka consumer.** Monitoring tools watch it constantly.

**Reset a group's offsets** (group must be inactive, so stop its consumers first):

```powershell
kafka-consumer-groups.bat --bootstrap-server localhost:9092 --group workers --topic sales --reset-offsets --to-earliest --execute
```

After this, the group re-reads from the beginning. Other options: `--to-latest`, `--to-offset N`, `--shift-by -5`. Without `--execute` the command only shows what *would* happen (a dry run).

---

### 10. Reading a specific partition or offset

```powershell
kafka-console-consumer.bat --topic sales --partition 0 --offset 1 --bootstrap-server localhost:9092 --max-messages 2
```

This reads **only partition 0**, starting at **offset 1**, and stops after 2 records. It is the fastest way to answer *"what exactly is stored at this position?"*.

Use `--offset earliest` or `--offset latest` as special values.

---

### 11. Topic configuration (retention)

Retention (Phase 2, Section 12) can be set **per topic**.

**View a topic's overrides**

```powershell
kafka-configs.bat --bootstrap-server localhost:9092 --entity-type topics --entity-name sales --describe
```

**Keep data for only 1 hour (3,600,000 ms)**

```powershell
kafka-configs.bat --bootstrap-server localhost:9092 --entity-type topics --entity-name sales --alter --add-config retention.ms=3600000
```

⚠️ **Typical output**

```text
Completed updating config for topic sales.
```

**Remove the override** (back to the broker default):

```powershell
kafka-configs.bat --bootstrap-server localhost:9092 --entity-type topics --entity-name sales --alter --delete-config retention.ms
```

| Common topic config | Meaning |
| --- | --- |
| `retention.ms` | Keep records this many milliseconds |
| `retention.bytes` | Max size per partition |
| `cleanup.policy` | `delete` or `compact` |
| `segment.bytes` | Segment file size |
| `min.insync.replicas` | Minimum ISR for `acks=all` writes |

---

### 12. Producer in Python

Now the same job from a program. Python 3.10 with `kafka-python` (installed in Phase 4).

Create `C:\kafka-labs\producer.py`:

```python
from kafka import KafkaProducer

producer = KafkaProducer(
    bootstrap_servers="localhost:9092",
    key_serializer=lambda k: k.encode("utf-8"),
    value_serializer=lambda v: v.encode("utf-8"),
)

orders = [
    ("Delhi",  "Laptop,2"),
    ("Mumbai", "Phone,5"),
    ("Delhi",  "Tablet,3"),
]

for city, value in orders:
    future = producer.send("sales", key=city, value=value)
    meta = future.get(timeout=10)          # wait for the broker's acknowledgement
    print(f"sent key={city} value={value} -> partition={meta.partition} offset={meta.offset}")

producer.flush()     # make sure everything is sent
producer.close()
```

Run:

```powershell
python C:\kafka-labs\producer.py
```

⚠️ **Typical output**

```text
sent key=Delhi value=Laptop,2 -> partition=0 offset=4
sent key=Mumbai value=Phone,5 -> partition=1 offset=2
sent key=Delhi value=Tablet,3 -> partition=0 offset=5
```

**Line-by-line**

| Code | Meaning |
| --- | --- |
| `KafkaProducer(bootstrap_servers=...)` | Connect to the cluster |
| `key_serializer`, `value_serializer` | Convert text to **bytes** (Kafka stores bytes) |
| `producer.send(topic, key=, value=)` | Queue one record (asynchronous) |
| `future.get(timeout=10)` | Wait until Kafka acknowledges; returns partition and offset |
| `producer.flush()` | Force all buffered records out |
| `producer.close()` | Release resources |

> `send()` does **not** block. Without `future.get()` or `flush()` the program may exit before the records are actually sent.

---

### 13. Consumer in Python

Create `C:\kafka-labs\consumer.py`:

```python
from kafka import KafkaConsumer

consumer = KafkaConsumer(
    "sales",
    bootstrap_servers="localhost:9092",
    group_id="py-group",
    auto_offset_reset="earliest",        # start from the beginning if no saved offset
    enable_auto_commit=True,             # save the bookmark automatically
    key_deserializer=lambda k: k.decode("utf-8") if k else None,
    value_deserializer=lambda v: v.decode("utf-8"),
    consumer_timeout_ms=10000,           # stop after 10 s of silence
)

for msg in consumer:
    print(f"topic={msg.topic} partition={msg.partition} offset={msg.offset} key={msg.key} value={msg.value}")

consumer.close()
```

Run:

```powershell
python C:\kafka-labs\consumer.py
```

⚠️ **Typical output**

```text
topic=sales partition=0 offset=0 key=Delhi value=Laptop,2
topic=sales partition=0 offset=1 key=Delhi value=Tablet,3
topic=sales partition=1 offset=0 key=Mumbai value=Phone,5
...
```

**Line-by-line**

| Code | Meaning |
| --- | --- |
| `KafkaConsumer("sales", ...)` | Subscribe to topic `sales` |
| `group_id="py-group"` | Consumer group name (offsets are saved per group) |
| `auto_offset_reset="earliest"` | Where to begin when the group has no bookmark |
| `enable_auto_commit=True` | Commit offsets automatically in the background |
| `value_deserializer` | Convert bytes back to text |
| `for msg in consumer` | Infinite pull loop; each `msg` has topic, partition, offset, key, value |
| `consumer_timeout_ms` | Leave the loop if no records arrive for that long |

**Run it twice.** The first run prints all records. The second run prints **nothing** (until new records arrive), because group `py-group` has already committed its offsets. Change `group_id` to read everything again.

---

### 14. A fake sales-event generator

We need a **live stream** to feed Spark later. This script sends one random sales event per second to the topic `sales`, in the same format we used in Unit IV's sales table.

Create `C:\kafka-labs\sales_generator.py`:

```python
import random
import time
from kafka import KafkaProducer

producer = KafkaProducer(
    bootstrap_servers="localhost:9092",
    key_serializer=lambda k: k.encode("utf-8"),
    value_serializer=lambda v: v.encode("utf-8"),
)

products = {"Laptop": 55000, "Phone": 20000, "Tablet": 15000}
cities = ["Delhi", "Mumbai", "Pune"]

order_id = 1
try:
    while True:
        product = random.choice(list(products))
        city = random.choice(cities)
        quantity = random.randint(1, 5)
        price = products[product]
        value = f"{order_id},{product},{city},{quantity},{price}"   # CSV line
        producer.send("sales", key=city, value=value)
        print("sent:", value)
        order_id += 1
        time.sleep(1)
except KeyboardInterrupt:
    print("stopping")
finally:
    producer.flush()
    producer.close()
```

Run it and leave it open (stop with **Ctrl + C**):

```powershell
python C:\kafka-labs\sales_generator.py
```

⚠️ **Typical output**

```text
sent: 1,Laptop,Delhi,2,55000
sent: 2,Phone,Mumbai,5,20000
sent: 3,Tablet,Pune,3,15000
...
```

Each message is a CSV line: `order_id,product,city,quantity,price`. In Phase 7 Spark will parse exactly this format.

**JSON variant** (used later in Phase 8) — change the `value` line to:

```python
import json
value = json.dumps({"order_id": order_id, "product": product, "city": city, "quantity": quantity, "price": price})
```

---

### 15. Producer and consumer in Scala (spark-shell)

When Spark is started with the Kafka connector, the Java Kafka client library is already on the class path, so we can write a producer **directly in the shell**.

```powershell
spark-shell --packages org.apache.spark:spark-sql-kafka-0-10_2.13:3.5.9
```

**Producer**

```scala
import java.util.Properties
import org.apache.kafka.clients.producer.{KafkaProducer, ProducerRecord}

val props = new Properties()
props.put("bootstrap.servers", "localhost:9092")
props.put("key.serializer",   "org.apache.kafka.common.serialization.StringSerializer")
props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer")

val producer = new KafkaProducer[String, String](props)

producer.send(new ProducerRecord[String, String]("sales", "Delhi", "101,Laptop,Delhi,1,55000"))
producer.send(new ProducerRecord[String, String]("sales", "Mumbai", "102,Phone,Mumbai,2,20000"))
producer.flush()
producer.close()
```

| Code | Meaning |
| --- | --- |
| `Properties` + `put(...)` | Client configuration as key-value pairs |
| `StringSerializer` | Converts Strings to bytes |
| `new ProducerRecord[String,String](topic, key, value)` | One record |
| `producer.send(...)` | Asynchronous send |
| `producer.flush()` / `close()` | Push out buffered records / release |

**Consumer**

```scala
import java.util.{Properties, Collections}
import java.time.Duration
import org.apache.kafka.clients.consumer.KafkaConsumer
import scala.jdk.CollectionConverters._

val cprops = new Properties()
cprops.put("bootstrap.servers", "localhost:9092")
cprops.put("group.id", "scala-group")
cprops.put("auto.offset.reset", "earliest")
cprops.put("key.deserializer",   "org.apache.kafka.common.serialization.StringDeserializer")
cprops.put("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer")

val consumer = new KafkaConsumer[String, String](cprops)
consumer.subscribe(Collections.singletonList("sales"))

val records = consumer.poll(Duration.ofSeconds(5))
for (r <- records.asScala)
  println(s"partition=${r.partition} offset=${r.offset} key=${r.key} value=${r.value}")

consumer.close()
```

> **Scala 2.13 note:** `scala.jdk.CollectionConverters._` is the 2.13 way to convert Java collections (`.asScala`). In Scala 2.12 it was `scala.collection.JavaConverters._`. Your Scala 2.13.8 uses the first form.

The `poll(...)` call is the heart of the consumer: it asks the broker for new records and waits up to 5 seconds.

---

### 16. Important producer settings

| Setting | Meaning | Typical choice |
| --- | --- | --- |
| `bootstrap.servers` | Entry-point brokers | `localhost:9092` |
| `key.serializer`, `value.serializer` | Object → bytes | `StringSerializer` |
| `acks` | When is a send successful? | `all` for safety |
| `retries` | Retry on transient errors | default is high |
| `enable.idempotence` | Prevent duplicates from retries | `true` |
| `linger.ms` | Wait to fill a batch | `5` (a few ms) |
| `batch.size` | Max batch size in bytes | default 16 KB |
| `compression.type` | Compress batches | `lz4` / `snappy` |
| `max.in.flight.requests.per.connection` | Unacknowledged requests allowed | ≤ 5 with idempotence |

**Trade-off summary**

```text
Higher linger.ms / batch.size  → better throughput, slightly more delay
acks=all + idempotence         → safest, slightly slower
acks=0                         → fastest, may lose data
```

---

### 17. Important consumer settings

| Setting | Meaning | Typical choice |
| --- | --- | --- |
| `bootstrap.servers` | Entry-point brokers | `localhost:9092` |
| `group.id` | Consumer group name | any unique name |
| `key.deserializer`, `value.deserializer` | Bytes → objects | `StringDeserializer` |
| `auto.offset.reset` | Start position when no bookmark | `earliest` or `latest` |
| `enable.auto.commit` | Commit offsets automatically | `true` (simple) / `false` (control) |
| `auto.commit.interval.ms` | How often to auto-commit | 5000 |
| `max.poll.records` | Records returned per poll | 500 |
| `session.timeout.ms` | Time without heartbeat before the consumer is considered dead | 45000 |

**Offset commit strategies and delivery semantics**

| Strategy | Result |
| --- | --- |
| Commit **before** processing | At most once (can lose records on crash) |
| Commit **after** processing | **At least once** (can duplicate on crash) |
| Transactions / idempotent sink | Exactly once (effectively) |

---

### 18. Common errors

| Symptom | Cause | Fix |
| --- | --- | --- |
| `Connection refused` / `NoBrokersAvailable` | Broker not running or wrong port | Check `jps`, use `localhost:9092` |
| Consumer prints nothing | Started after the messages, no `--from-beginning` / `earliest` | Add `--from-beginning` or change group |
| Second Python run prints nothing | Group already committed offsets | New `group_id` or reset offsets |
| `UNKNOWN_TOPIC_OR_PARTITION` | Typo in topic name | `--list` to check |
| Producer exits, records missing | No `flush()` / `get()` | Call `producer.flush()` |
| Replication factor error on topic create | RF larger than brokers | Use `--replication-factor 1` |
| `--broker-list` unrecognized | Old option removed | Use `--bootstrap-server` |
| Garbled characters | Wrong deserializer | Decode as UTF-8 |
| Same key lands in new partition | Partition count changed | Plan partitions up front |
| Records out of order | Spread across partitions | Use a **key** to group related records |

---

### 19. Practical scenarios — Phase 5 practice

1. **Create** a topic `students` with 2 partitions and RF 1, then **describe** it.
2. **Produce** three student records (`101,Alice,CSE,21,85` ...) with the console producer.
3. **Consume** them with `--from-beginning`.
4. **Produce keyed records** (`CSE:Alice,85`) and show keys and partitions in the consumer.
5. Prove **same key → same partition** by sending 5 records with one key and one with another.
6. Start **two consumers in the same group** and show that each record goes to only one.
7. Start **two consumers in different groups** and show both receive everything.
8. Run `kafka-consumer-groups.bat --describe` and **explain LAG**.
9. **Reset offsets** to earliest for a group and re-read the data.
10. Write the **Python producer**; print each record's partition and offset.
11. Write the **Python consumer** with `auto_offset_reset="earliest"`; run it twice and explain the difference.
12. Run the **sales generator** for 20 seconds, stop it, and count the records with `--max-messages`.
13. Set `retention.ms` on a topic using `kafka-configs.bat`.
14. Send two records from the **Scala shell** producer and read them back with the console consumer.

---

### 20. Phase 5 — viva questions and answers

#### Q1. What is a Kafka producer?

An application that publishes (writes) records to Kafka topics.

#### Q2. What is a Kafka consumer?

An application that subscribes to topics and reads (pulls) records from them.

#### Q3. Describe the steps a producer follows to send a record.

Create the record, serialize key and value, choose a partition, batch the records, send to the leader broker and receive an acknowledgement.

#### Q4. Does the broker push records to consumers?

No. Consumers **pull** records by polling the broker.

#### Q5. What is the command to create a topic?

`kafka-topics.bat --create --topic T --bootstrap-server localhost:9092 --partitions N --replication-factor R`.

#### Q6. How do you read all old messages with the console consumer?

Add `--from-beginning`.

#### Q7. How do you send keyed messages from the console producer?

Use `--property parse.key=true --property key.separator=:` and type `key:value`.

#### Q8. What is consumer lag?

The number of records a consumer group has not read yet: `LOG-END-OFFSET − CURRENT-OFFSET`.

#### Q9. Which tool shows a group's offsets and lag?

`kafka-consumer-groups.bat --describe --group <name>`.

#### Q10. What does `--from-beginning` do when the group already has committed offsets?

Nothing for that group. It applies only when there is no committed offset; to re-read, use a new group or reset offsets.

#### Q11. What is `auto.offset.reset`?

The starting position (`earliest` or `latest`) when a group has no saved offset.

#### Q12. What happens when two consumers share the same group id?

They split the partitions; each record is processed by only one of them.

#### Q13. What happens when two consumers have different group ids?

Each group receives all records independently.

#### Q14. Why do producers and consumers need serializers/deserializers?

Kafka stores bytes only, so objects must be converted to bytes before sending and back after reading.

#### Q15. What does `acks=all` mean?

The producer waits until all in-sync replicas have written the record.

#### Q16. Why call `producer.flush()` before exiting?

`send()` is asynchronous and buffers records; `flush()` ensures they are all actually sent.

#### Q17. What is the difference between committing offsets before and after processing?

Before = at most once (may lose); after = at least once (may duplicate).

#### Q18. What is `enable.auto.commit`?

A consumer setting that makes the client commit offsets automatically at regular intervals.

#### Q19. How can you reset a consumer group to read from the beginning?

`kafka-consumer-groups.bat --reset-offsets --to-earliest --execute --group G --topic T` (with the group inactive).

#### Q20. Why should we avoid deleting topics on Windows?

It can crash the broker due to file locking; new topic names or resetting data folders is safer.

---

### 21. Phase 5 — final master code revision

```powershell
REM ---------- TOPICS ----------
kafka-topics.bat --create --topic sales --bootstrap-server localhost:9092 --partitions 3 --replication-factor 1
kafka-topics.bat --list --bootstrap-server localhost:9092
kafka-topics.bat --describe --topic sales --bootstrap-server localhost:9092

REM ---------- PRODUCER ----------
kafka-console-producer.bat --topic sales --bootstrap-server localhost:9092
kafka-console-producer.bat --topic sales --bootstrap-server localhost:9092 --property parse.key=true --property key.separator=:

REM ---------- CONSUMER ----------
kafka-console-consumer.bat --topic sales --from-beginning --bootstrap-server localhost:9092
kafka-console-consumer.bat --topic sales --group g1 --bootstrap-server localhost:9092 --property print.key=true --property print.partition=true --property print.offset=true

REM ---------- GROUPS ----------
kafka-consumer-groups.bat --bootstrap-server localhost:9092 --list
kafka-consumer-groups.bat --bootstrap-server localhost:9092 --describe --group g1
kafka-consumer-groups.bat --bootstrap-server localhost:9092 --group g1 --topic sales --reset-offsets --to-earliest --execute
```

```python
# ---------- PYTHON PRODUCER ----------
from kafka import KafkaProducer
p = KafkaProducer(bootstrap_servers="localhost:9092",
                  key_serializer=lambda k: k.encode(), value_serializer=lambda v: v.encode())
p.send("sales", key="Delhi", value="Laptop,2").get(timeout=10)
p.flush(); p.close()

# ---------- PYTHON CONSUMER ----------
from kafka import KafkaConsumer
c = KafkaConsumer("sales", bootstrap_servers="localhost:9092", group_id="g1",
                  auto_offset_reset="earliest", value_deserializer=lambda v: v.decode(),
                  consumer_timeout_ms=5000)
for m in c: print(m.partition, m.offset, m.value)
```

**Quick-recall card**

| Need | Command / setting |
| --- | --- |
| Create topic | `kafka-topics.bat --create ...` |
| Send messages | `kafka-console-producer.bat` |
| Read messages | `kafka-console-consumer.bat` |
| Read old data | `--from-beginning` |
| Keys | `parse.key=true`, `key.separator=:` ; `print.key=true` |
| Group | `--group name` |
| Lag | `kafka-consumer-groups.bat --describe` |
| Safe producer | `acks=all`, `enable.idempotence=true` |
| Start position | `auto.offset.reset = earliest / latest` |

**Phase 5 completed**

---

> **Next → Phase 6: Spark Streaming Foundations** (DStreams and Structured Streaming: micro-batches, sockets, files, rate source, output modes, triggers, windows, checkpoints, all verified in a real `spark-shell`). File: `6_Unit5_Spark_Streaming_Foundations.md`
