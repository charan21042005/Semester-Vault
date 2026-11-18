# Unit V — Phase 2: Kafka Core Concepts from Absolute Scratch

> **Continuing from Phase 1.** We know *why* Kafka exists. Now we learn **what is inside it**: events, topics, partitions, offsets, producers, consumers and consumer groups. This is the most important theory phase of the unit. Every later phase depends on it.
>
> The Scala simulation in this phase was run in a real `spark-shell`, so the outputs are genuine. It is a *teaching toy*, not real Kafka.

---

## Phase 2 — Roadmap

| # | Topic |
| --- | --- |
| 1 | The Kafka vocabulary at a glance |
| 2 | Event / record / message |
| 3 | Topic |
| 4 | Partition |
| 5 | Offset |
| 6 | Ordering guarantee |
| 7 | Producer and how it picks a partition |
| 8 | Consumer |
| 9 | Consumer group |
| 10 | Queue model and pub-sub model using groups |
| 11 | Broker and cluster (preview) |
| 12 | Retention: Kafka keeps the data |
| 13 | Serialization: everything is bytes |
| 14 | Delivery semantics (preview) |
| 15 | Build a mini Kafka in Scala |
| 16 | Paper exercises |
| 17 | Common confusions |
| 18 | Viva questions |
| 19 | Master revision |

---

### 1. The Kafka vocabulary at a glance

```text
PRODUCER ──writes──▶  TOPIC "sales"                   ──read by──▶ CONSUMER
                      ┌─ Partition 0: [0][1][2][3] ─▶ ... new events appended here
                      ├─ Partition 1: [0][1][2]    ─▶
                      └─ Partition 2: [0][1][2][3][4]
                                     ↑
                                  offsets
```

| Term | One-line meaning |
| --- | --- |
| **Event / record / message** | One piece of data (a key, a value, a timestamp) |
| **Topic** | A named category or feed of events (like a table or folder) |
| **Partition** | A topic is split into ordered logs called partitions |
| **Offset** | The position number of a record inside a partition |
| **Producer** | An application that **writes** events to a topic |
| **Consumer** | An application that **reads** events from a topic |
| **Consumer group** | A team of consumers that share the work of reading a topic |
| **Broker** | A Kafka server that stores partitions |
| **Cluster** | Several brokers working together |

Learn this table now. Everything else is detail.

---

### 2. Event / record / message

Kafka's unit of data is a **record**. The words *event*, *record* and *message* are used interchangeably.

A record has these parts:

| Part | Purpose | Example |
| --- | --- | --- |
| **Key** (optional) | Decides the partition; groups related events | `Delhi` |
| **Value** | The actual data | `Laptop,2,55000` |
| **Timestamp** | When the event happened or was stored | `2026-11-18 10:00:00` |
| **Headers** (optional) | Extra metadata (key-value pairs) | `source=web` |

```text
┌───────────────────────────────────────────┐
│ key:       "Delhi"                        │
│ value:     "Laptop,2,55000"               │
│ timestamp: 2026-11-18 10:00:00            │
│ headers:   source=web                     │
└───────────────────────────────────────────┘
```

**Important:** Kafka does **not** care what the value looks like. It can be text, CSV, JSON, even an image. Kafka stores it as **bytes**. Meaning is decided by producers and consumers.

---

### 3. Topic

A **topic** is a named stream of records. Producers write to a topic; consumers read from a topic.

| Analogy | Meaning |
| --- | --- |
| A folder | Related files go in one folder |
| A database table | Related rows go in one table |
| A TV channel | Everything broadcast on that channel |

**Example topics for an online shop**

```text
orders        → every order placed
payments      → every payment event
clicks        → every click on the website
alerts        → suspicious activity
```

**Topic rules to remember**

1. A topic has a **unique name** (for example `sales`).
2. A topic can have **many producers** and **many consumers**.
3. Records in a topic are **not deleted when read**. They are kept until the retention period expires (Section 12).
4. A topic is divided into **partitions**.

> **Naming tip:** use lowercase letters, digits, dots, dashes or underscores (for example `student-marks`). Avoid spaces.

---

### 4. Partition

A topic is split into one or more **partitions**. A partition is an **ordered, append-only log**: new records are only ever added at the end.

```text
Topic: sales (3 partitions)

Partition 0:  [Delhi,Laptop] [Delhi,Tablet] [Pune,Phone] → new records here
Partition 1:  [Mumbai,Phone] [Mumbai,Laptop]              → new records here
Partition 2:  (empty so far)
```

**Why do we need partitions?**

| Reason | Explanation |
| --- | --- |
| **Parallelism** | Several consumers can read different partitions at the same time |
| **Scalability** | Partitions can sit on different brokers (machines) |
| **Throughput** | Several producers can write to different partitions simultaneously |

One log on one machine would become a bottleneck. Partitions spread the load.

**Key facts**

- Each record lives in **exactly one** partition.
- A partition is stored on a broker and can be replicated (Phase 3).
- The number of partitions is chosen when creating a topic (you can increase it later, never decrease it).

---

### 5. Offset

Every record in a partition gets a sequential number called its **offset**, starting from **0**.

```text
Partition 0
 offset:   0        1        2        3
         [rec A] [rec B] [rec C] [rec D]  → next record will get offset 4
```

**Very important points**

1. Offsets are **per partition**. Partition 0 has its own offsets 0, 1, 2...; Partition 1 also starts from 0.
2. An offset is **permanent and never reused**.
3. A record is uniquely identified by **(topic, partition, offset)**.
4. A consumer remembers *"I have read up to offset N"*. That memory is its **bookmark**.

```text
Unique address of a record:   sales / partition 1 / offset 4
```

> **Exam trap:** "Are offsets global across the topic?" → **No**, they are unique only within a partition.

---

### 6. Ordering guarantee

> **Kafka guarantees order only within a single partition, not across the whole topic.**

```text
Partition 0:  A → B → C        ✅ always read in this order
Partition 1:  X → Y → Z        ✅ always read in this order

Across partitions: A, X, B, Y ... or X, A, Y, B ...  ❌ no guarantee
```

**How do we keep related events in order?** Give them the **same key**. All records with the same key go to the same partition (Section 7), so their order is preserved.

Example: all events for customer `1001` use key `1001`. They land in one partition, so "order placed → payment done → shipped" stays in sequence.

---

### 7. Producer and how it picks a partition

A **producer** is any application that writes records to a topic. It could be a Python script, a Scala program, or the Kafka console tool.

**How does the producer choose a partition?**

| Situation | Rule |
| --- | --- |
| Record has a **key** | `partition = hash(key) mod numberOfPartitions` (Kafka uses the murmur2 hash) |
| Record has **no key** | Spread the records across partitions (sticky batches / round-robin style) |
| Producer names a partition explicitly | That partition is used |

```text
key = "Delhi"   ─▶ hash ─▶ mod 3 ─▶ partition 0   (always the same for "Delhi")
key = "Mumbai"  ─▶ hash ─▶ mod 3 ─▶ partition 1
key = null      ─▶ spread across partitions
```

**Rule:** the same key always goes to the same partition (as long as the partition count does not change).

**Producer flow**

```text
Record ─▶ Serialize to bytes ─▶ Choose partition ─▶ Batch ─▶ Send to broker ─▶ Acknowledgement
```

---

### 8. Consumer

A **consumer** reads records from a topic. Three important behaviours:

1. **Pull model:** the consumer *asks* the broker for new records (it polls). The broker does not push.
2. **Own pace:** a slow consumer does not slow down a fast one.
3. **Tracks its position:** it stores the **offset** up to which it has read, so it can continue after a restart.

```text
Consumer reading Partition 0:

 offset:   0    1    2    3    4    5
          [x]  [x]  [x]  [ ]  [ ]  [ ]
                         ↑
                   current position = 3  (next record to read)
```

**Starting position when a consumer first starts**

| Setting | Meaning |
| --- | --- |
| `earliest` | Start from the **oldest** available record |
| `latest` | Start from **new** records only (default for most clients) |

---

### 9. Consumer group

A **consumer group** is a set of consumers that cooperate to read a topic. Each consumer declares a **group id**.

**The golden rule:**

> **Within one consumer group, each partition is read by exactly ONE consumer. But one consumer can read several partitions.**

**Example: topic with 3 partitions**

| Consumers in the group | Assignment |
| --- | --- |
| 1 consumer | C1 reads P0, P1, P2 |
| 2 consumers | C1 reads P0, P1; C2 reads P2 |
| 3 consumers | C1 reads P0; C2 reads P1; C3 reads P2 |
| 4 consumers | C1, C2, C3 read one each; **C4 sits idle** |

```text
          Topic sales                    Consumer group "dashboard"
        ┌── P0 ──────────────────────▶  Consumer C1
        ├── P1 ──────────────────────▶  Consumer C2
        └── P2 ──────────────────────▶  Consumer C3
```

**Why does this matter?**

- It lets us **scale reading**: add consumers to read faster.
- The **maximum useful consumers = number of partitions**.
- If a consumer fails, Kafka **reassigns** its partitions to the others (a *rebalance*).

---

### 10. Queue model and pub-sub model using groups

Remember the two messaging models from Phase 1? Kafka gives **both** through consumer groups.

**Case 1 — Same group → queue behaviour (work sharing)**

Consumers with the **same** group id *split* the records.

```text
Topic ─▶ Group "workers": C1 gets some records, C2 gets the others
```

**Case 2 — Different groups → pub-sub behaviour (broadcast)**

Each **different** group gets **all** the records, independently.

```text
Topic ─▶ Group "dashboard":  gets ALL records
      └▶ Group "fraud":      gets ALL records
      └▶ Group "archive":    gets ALL records
```

| Same group id | Different group ids |
| --- | --- |
| Records are **shared** among consumers | Each group gets **every** record |
| Queue behaviour | Pub-sub behaviour |
| Used to **scale** processing | Used to serve **different applications** |

Each group keeps its **own offsets**, so the groups never disturb each other.

> **Viva gold:** "How does Kafka support both queuing and publish-subscribe?" → **Through consumer groups.**

---

### 11. Broker and cluster (preview)

- A **broker** is a Kafka server process. It stores partitions and serves producers and consumers.
- A **cluster** is a group of brokers working together.
- Partitions of a topic are spread over brokers, and copies (replicas) protect against failures.

```text
Cluster:  Broker 1 [P0, P2]   Broker 2 [P1, P0']   Broker 3 [P2', P1']
```

Full details in **Phase 3**.

---

### 12. Retention: Kafka keeps the data

In a normal queue, a message disappears after it is read. In Kafka it **stays**.

Records are deleted only by **retention rules**:

| Rule | Meaning | Default |
| --- | --- | --- |
| Time based | Delete records older than N time | 7 days (`log.retention.hours=168`) |
| Size based | Delete oldest records when a partition grows beyond N bytes | Unlimited |

```text
Written:  day 1 ................................ day 8
Kept:     [day 1 ... day 7]  ← day 1 records deleted on day 8
```

**Why is this powerful?**

1. **Replay:** a new consumer can start from `earliest` and read old data.
2. **Recovery:** if the consumer crashed, it resumes from its saved offset.
3. **Multiple consumers:** reading never removes data for others.

---

### 13. Serialization: everything is bytes

Kafka stores and transfers **bytes** only.

| Step | Who | What |
| --- | --- | --- |
| Serialize | Producer | Converts text/number/object into bytes |
| Store | Kafka | Keeps the bytes unchanged |
| Deserialize | Consumer | Converts bytes back into text/number/object |

```text
"Laptop,2" ──serialize──▶ 4C 61 70 74 6F 70 2C 32 ──deserialize──▶ "Laptop,2"
```

**Why we will see this again:** In Spark (Phase 7), the Kafka DataFrame has `key` and `value` columns of type **binary**. We must `CAST` them to `STRING` before using them.

---

### 14. Delivery semantics (preview)

Three possible guarantees about delivery:

| Semantics | Meaning | Risk |
| --- | --- | --- |
| **At most once** | Delivered 0 or 1 time | Messages may be **lost** |
| **At least once** | Delivered 1 or more times | Messages may be **duplicated** |
| **Exactly once** | Delivered exactly 1 time | Hardest to achieve |

Kafka can provide all three, depending on producer settings and how offsets are committed. We discuss this properly in Phase 3 (acks) and Phase 8 (pipelines).

---

### 15. Build a mini Kafka in Scala

Let's *see* topics, partitions and offsets working. This is a small teaching program, not real Kafka.

Open Spark Shell (any Scala REPL will do):

```powershell
spark-shell
```

**Step 1 — A record and a mini topic**

```scala
import scala.collection.mutable.ArrayBuffer

case class Record(key: String, value: String)

class MiniTopic(val name: String, val numPartitions: Int) {
  // one append-only log per partition
  val partitions = Array.fill(numPartitions)(ArrayBuffer[Record]())

  // producer side: choose a partition from the key, append, return (partition, offset)
  def send(key: String, value: String): (Int, Long) = {
    val p = math.abs(key.hashCode) % numPartitions
    partitions(p) += Record(key, value)
    (p, partitions(p).size - 1L)
  }

  // consumer side: read a partition starting at a given offset
  def read(p: Int, fromOffset: Long): Seq[(Long, Record)] =
    partitions(p).zipWithIndex.drop(fromOffset.toInt)
      .map { case (r, i) => (i.toLong, r) }.toSeq
}
```

**What each part does**

| Code | Concept |
| --- | --- |
| `partitions = Array.fill(n)(ArrayBuffer...)` | A topic = several logs |
| `ArrayBuffer` + `+=` | Append-only log |
| `hashCode % numPartitions` | Key-based partitioning (real Kafka uses murmur2) |
| `size - 1` | The new record's offset |
| `read(p, fromOffset)` | A consumer reading from its bookmark |

**Step 2 — Produce five events**

```scala
val sales = new MiniTopic("sales", 3)

sales.send("Delhi",  "Laptop,2")
sales.send("Mumbai", "Phone,5")
sales.send("Delhi",  "Tablet,3")
sales.send("Pune",   "Phone,1")
sales.send("Mumbai", "Laptop,1")
```

**Step 3 — Look inside each partition**

```scala
for (p <- 0 until 3)
  println(s"partition $p -> " + sales.partitions(p).map(r => r.key + ":" + r.value).mkString(" | "))
```

**Output**

```text
partition 0 -> Delhi:Laptop,2 | Delhi:Tablet,3 | Pune:Phone,1
partition 1 -> Mumbai:Phone,5 | Mumbai:Laptop,1
partition 2 ->
```

**Observations**

- Both `Delhi` records are in partition 0, in the order they were sent. **Same key → same partition → order kept.**
- Both `Mumbai` records are in partition 1.
- Partition 2 is empty. Partitions need not be balanced when there are few keys.
- (The exact partition numbers come from our toy hash. Real Kafka uses a different hash, but the *behaviour* is identical.)

**Step 4 — Two consumer groups with independent bookmarks**

```scala
val groupA = scala.collection.mutable.Map[Int, Long](0 -> 0L, 1 -> 0L, 2 -> 0L)
val groupB = scala.collection.mutable.Map[Int, Long](0 -> 0L, 1 -> 0L, 2 -> 0L)

def poll(group: scala.collection.mutable.Map[Int, Long], who: String) =
  for (p <- 0 until 3) {
    val msgs = sales.read(p, group(p))
    msgs.foreach(m => println(s"$who p$p offset=${m._1} ${m._2.key}:${m._2.value}"))
    group(p) = group(p) + msgs.size       // move the bookmark forward
  }
```

`group` is the map of **partition → next offset to read**. This is exactly what Kafka stores as *committed offsets*.

**Group A reads everything**

```scala
poll(groupA, "A")
```

**Output**

```text
A p0 offset=0 Delhi:Laptop,2
A p0 offset=1 Delhi:Tablet,3
A p0 offset=2 Pune:Phone,1
A p1 offset=0 Mumbai:Phone,5
A p1 offset=1 Mumbai:Laptop,1
```

**A new event arrives; group A polls again and gets only the new one**

```scala
sales.send("Delhi", "Phone,2")
poll(groupA, "A")
```

**Output**

```text
A p0 offset=3 Delhi:Phone,2
```

Group A's bookmark was already at offset 3, so it reads only the new record.

**Group B starts now: it still gets everything (data was not deleted)**

```scala
poll(groupB, "B")
```

**Output**

```text
B p0 offset=0 Delhi:Laptop,2
B p0 offset=1 Delhi:Tablet,3
B p0 offset=2 Pune:Phone,1
B p0 offset=3 Delhi:Phone,2
B p1 offset=0 Mumbai:Phone,5
B p1 offset=1 Mumbai:Laptop,1
```

**What this proved**

| Behaviour | Seen in the simulation |
| --- | --- |
| Append-only log with offsets | Records 0, 1, 2, 3 in partition 0 |
| Order kept for one key | Both Delhi records stay in sequence |
| Consumers keep independent bookmarks | A was at offset 3 while B started at 0 |
| Reading does not delete data | B read records A had already read |
| Different groups each get everything | A and B both received all records |

---

### 16. Paper exercises

Do these without a computer. Answers follow.

**Exercise 1.** A topic has 3 partitions. Records with keys `A`, `B`, `A`, `C`, `B`, `A` arrive in this order. Suppose `A → P0`, `B → P1`, `C → P2`. Write the contents of each partition.

**Exercise 2.** A topic has 4 partitions. A consumer group has 6 consumers. How many consumers are idle?

**Exercise 3.** Two groups, `dashboard` and `archive`, read the same topic that has 2 partitions. Each group has 2 consumers. How many consumers read partition 0 in total?

**Exercise 4.** A partition contains offsets 0 to 9. A consumer has committed offset 6. Which record will it read next, and how many are still unread?

**Exercise 5.** Why is `(partition 1, offset 5)` alone not enough to identify a record uniquely?

**Answers**

1. P0 = `A, A, A` (in arrival order), P1 = `B, B`, P2 = `C`.
2. 2 idle (only 4 consumers can be active, one per partition).
3. **2.** One consumer from each group reads partition 0 (one per group).
4. It reads **offset 6** next; offsets 6, 7, 8, 9 = **4 unread** records.
5. Offsets are per partition *and per topic*. The full address is (topic, partition, offset).

---

### 17. Phase 2 — common confusions

| Confusion | Clarification |
| --- | --- |
| "Offsets are global for the topic" | Offsets are **per partition**. |
| "Kafka guarantees order for the entire topic" | Only **within a partition**. |
| "A record is deleted after it is consumed" | No. It stays until **retention** expires. |
| "More consumers always means faster" | Not beyond the number of **partitions**. Extra consumers idle. |
| "Two consumers in different groups split the data" | No. **Different groups each get everything.** |
| "The broker pushes data to consumers" | Consumers **pull**. |
| "Key is mandatory" | Optional. Needed only when you want same-key ordering. |
| "Kafka understands JSON/CSV" | It stores **bytes**. Meaning belongs to the applications. |

---

### 18. Phase 2 — viva questions and answers

#### Q1. What is a Kafka record?

The unit of data in Kafka, made of an optional key, a value, a timestamp and optional headers.

#### Q2. What is a topic?

A named category or feed to which records are published and from which they are read.

#### Q3. What is a partition? Why are partitions needed?

A partition is an ordered, append-only log that forms part of a topic. Partitions give parallelism, scalability and higher throughput.

#### Q4. What is an offset?

A sequential number, starting at 0, that identifies a record's position within a partition.

#### Q5. Are offsets unique across the whole topic?

No. They are unique only within a partition.

#### Q6. What does Kafka guarantee about message order?

Order is guaranteed only within a single partition.

#### Q7. How can you make sure related events stay in order?

Send them with the same key so they go to the same partition.

#### Q8. How does a producer choose a partition?

With a key: `hash(key) mod numPartitions`. Without a key: records are spread across partitions. A partition can also be specified explicitly.

#### Q9. What is a consumer? Does Kafka push or does the consumer pull?

A consumer reads records from a topic. Consumers **pull** records from the broker.

#### Q10. What is a consumer group?

A set of consumers sharing a group id, which cooperate to read a topic so that each partition is read by only one member of the group.

#### Q11. What happens if there are more consumers than partitions in a group?

The extra consumers stay idle.

#### Q12. How does Kafka support both queue and pub-sub models?

Consumers in the **same** group share records (queue behaviour); **different** groups each receive all records (pub-sub behaviour).

#### Q13. Are records deleted after being consumed?

No. They are kept until the retention time or size limit is reached.

#### Q14. Why can a new consumer read old data?

Because Kafka stores records durably and a consumer can start from the `earliest` offset.

#### Q15. What data type does Kafka store?

Bytes. Producers serialize and consumers deserialize.

#### Q16. What are the delivery semantics?

At most once (may lose), at least once (may duplicate), exactly once.

#### Q17. What is `auto.offset.reset`?

It tells a consumer where to start when it has no saved offset: `earliest` (oldest record) or `latest` (only new records).

#### Q18. What uniquely identifies a record in Kafka?

The combination of topic, partition and offset.

---

### 19. Phase 2 — final master recap

```text
KAFKA CORE CONCEPTS
  Record  = key + value + timestamp (+ headers)   → stored as bytes
  Topic   = named stream, split into partitions
  Partition = ordered append-only log, records numbered by OFFSET
  Producer  = writes; key → partition = hash(key) mod N
  Consumer  = reads (pull), remembers its offset
  Group     = consumers sharing work; 1 partition → 1 consumer per group
  Retention = records kept for 7 days by default; replay is possible
```

**Seven facts to memorise**

1. Topic → many partitions.
2. Offset is per partition, starts at 0.
3. Order is guaranteed only inside a partition.
4. Same key → same partition.
5. One partition → one consumer per group.
6. Different groups → each gets all records.
7. Reading does not delete records; retention does.

### Phase 2 completed

---

> **Next → Phase 3: Kafka Cluster Architecture** (brokers, controller, ZooKeeper vs KRaft, replication, leaders and followers, ISR, `acks`, rebalancing, log segments). File: `3_Unit5_Kafka_Cluster_Architecture.md`
