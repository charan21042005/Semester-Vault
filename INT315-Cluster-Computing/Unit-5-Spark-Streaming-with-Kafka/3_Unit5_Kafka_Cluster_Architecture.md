# Unit V — Phase 3: Apache Kafka Cluster Architecture

> **Continuing from Phase 2.** We know topics, partitions, offsets, producers and consumers. Now we look **inside the machine room**: how brokers form a cluster, how data is copied for safety, and what happens when a machine dies.
>
> This phase is theory-heavy and diagram-heavy. The `--describe` outputs shown are **⚠️ typical outputs** of Kafka 3.4.0. Your IDs and folder names will differ. Practical commands start in Phase 4.

---

## Phase 3 — Roadmap

| # | Topic |
| --- | --- |
| 1 | The big picture: a Kafka cluster |
| 2 | Broker |
| 3 | Bootstrap servers |
| 4 | How partitions spread across brokers |
| 5 | Replication: leader and followers |
| 6 | In-Sync Replicas (ISR) |
| 7 | What happens when a broker fails |
| 8 | Producer acknowledgements (`acks`) |
| 9 | The controller |
| 10 | ZooKeeper: the coordinator |
| 11 | KRaft: the ZooKeeper-free future |
| 12 | Consumer groups inside the cluster |
| 13 | Storage: logs, segments and files |
| 14 | Retention and cleanup policies |
| 15 | Reading a `--describe` output |
| 16 | Important broker configurations |
| 17 | Sizing rules of thumb |
| 18 | Kafka cluster and Spark cluster together |
| 19 | Paper exercises |
| 20 | Common confusions |
| 21 | Viva questions |
| 22 | Master revision |

---

### 1. The big picture: a Kafka cluster

```text
                       KAFKA CLUSTER
   ┌──────────────────────────────────────────────────────┐
   │   Broker 1          Broker 2          Broker 3        │
   │  ┌────────┐        ┌────────┐        ┌────────┐       │
   │  │ P0 (L) │        │ P1 (L) │        │ P2 (L) │       │
   │  │ P1 (F) │        │ P2 (F) │        │ P0 (F) │       │
   │  └────────┘        └────────┘        └────────┘       │
   └──────────────▲───────────────────────────▲────────────┘
                  │                           │
             Producers                    Consumers
        ( write to leaders )        ( read from leaders )

   L = Leader replica      F = Follower replica (a copy)
   ZooKeeper (or KRaft controllers) coordinates the cluster
```

**In one sentence:**

> A Kafka cluster is a group of **brokers** that together store the **partitions** of many topics, keep **copies** of each partition for safety, and are **coordinated** by ZooKeeper or KRaft.

---

### 2. Broker

A **broker** is one Kafka server process.

**What a broker does**

1. Receives records from producers and **writes them to disk**.
2. Serves records to consumers.
3. Stores some partitions (as leader or follower).
4. Replicates data with other brokers.

**Identity:** every broker has a unique integer **`broker.id`** (0, 1, 2 ...).

```text
Broker = one JVM process + one data folder + one port (default 9092)
```

A cluster of **one** broker works (that is what we use for learning). A real company runs **three or more**.

---

### 3. Bootstrap servers

A client (producer, consumer or Spark) does not need to know **every** broker. It needs only the address of **one or two** brokers to start.

```text
--bootstrap-server localhost:9092
```

**What happens**

1. The client connects to the bootstrap broker.
2. That broker returns **metadata**: which brokers exist, which broker is the leader for each partition.
3. The client then talks **directly to the right leader brokers**.

```text
Client ──"who leads partition 1 of sales?"──▶ Broker 1 (bootstrap)
Client ◀──"Broker 2 leads it"───────────────── Broker 1
Client ──────────── writes directly ─────────▶ Broker 2
```

> **Remember:** `bootstrap.servers` is only the **first contact**. It is a *list of entry points*, not the full cluster list.

---

### 4. How partitions spread across brokers

When a topic is created, Kafka distributes its partitions over the brokers.

**Example: topic `sales`, 3 partitions, 3 brokers**

```text
Broker 1 ── Partition 0
Broker 2 ── Partition 1
Broker 3 ── Partition 2
```

Reading and writing now use **three machines in parallel**. This is how Kafka reaches very high throughput.

**More partitions than brokers is fine**

```text
6 partitions, 3 brokers:
Broker 1: P0, P3     Broker 2: P1, P4     Broker 3: P2, P5
```

---

### 5. Replication: leader and followers

Storing each partition on only one machine is dangerous: if that machine dies, the data is lost. So Kafka keeps **copies** of each partition.

- The number of copies is the **replication factor** (RF).
- RF = 3 means **three copies** of every partition, on **three different brokers**.

**Roles**

| Role | Meaning |
| --- | --- |
| **Leader** | The one replica that handles **all reads and writes** for the partition |
| **Follower** | A replica that **copies** the leader's data continuously |

```text
Partition 0, replication factor 3:

   Broker 1: [P0 LEADER]  ◀── producers write here, consumers read here
                 │ copies
        ┌────────┴────────┐
        ▼                 ▼
   Broker 2: [P0 follower]  Broker 3: [P0 follower]
```

**Key rules**

1. Each partition has **exactly one leader** at a time.
2. Followers do **not** serve clients normally. They only replicate.
3. The leaders of different partitions are **spread** across brokers, so no single broker handles all traffic.
4. **Replication factor cannot be larger than the number of brokers.**

> **Learning setup warning:** with a single broker, replication factor must be **1**. If you try RF = 3 with one broker, topic creation fails.

---

### 6. In-Sync Replicas (ISR)

A follower can fall behind (slow network, busy machine). Kafka tracks which replicas are **fully caught up**.

> **ISR = the set of replicas (leader included) that are in sync with the leader.**

```text
Partition 0: Replicas: 1,2,3     ISR: 1,2,3     (all caught up)
Partition 0: Replicas: 1,2,3     ISR: 1,3       (broker 2 is lagging, dropped from ISR)
```

**Why ISR matters**

- Only members of the ISR are **eligible to become the new leader**. This guarantees the new leader has all committed data.
- A record is considered **committed** only when **all ISR members** have it. Consumers only see committed records.

**Related setting:** `min.insync.replicas` is the minimum ISR size required to accept writes when the producer uses `acks=all` (Section 8).

---

### 7. What happens when a broker fails

**Setup:** RF = 3, brokers 1, 2, 3. Broker 1 is the leader of partition 0. ISR = {1, 2, 3}.

```text
STEP 1 — Normal
   B1: P0 LEADER      B2: P0 follower      B3: P0 follower

STEP 2 — Broker 1 crashes  💥
   B1: ✖ down         B2: P0 follower      B3: P0 follower

STEP 3 — The controller notices and elects a new leader from the ISR
   B1: ✖ down         B2: P0 LEADER ⭐     B3: P0 follower

STEP 4 — Clients refresh metadata and continue with Broker 2
   Producers and consumers now talk to B2. No data lost.

STEP 5 — Broker 1 comes back
   B1: P0 follower (catches up from B2, then rejoins ISR)
```

**Observations**

- Failover is **automatic**. The application usually notices only a short pause.
- No committed data is lost, because the new leader was in the ISR.
- This is what **fault tolerance** means in Kafka.

**How many failures can we survive?** With RF = 3, a partition can survive **2 broker failures** in terms of data existence. With `min.insync.replicas = 2` and `acks=all`, writes continue as long as at least 2 replicas are alive (so 1 failure is tolerated for writing).

---

### 8. Producer acknowledgements (`acks`)

The producer setting `acks` decides **when a send counts as successful**.

| `acks` | Producer waits for | Speed | Safety |
| --- | --- | --- | --- |
| `0` | **Nothing**. Fire and forget | Fastest | May lose data silently |
| `1` | The **leader** wrote it | Fast | Lost if the leader dies before copying |
| `all` (or `-1`) | **All in-sync replicas** wrote it | Slowest | **Safest** |

```text
acks=0  : Producer ──▶ Leader                      (no reply expected)
acks=1  : Producer ──▶ Leader ──ack──▶ Producer
acks=all: Producer ──▶ Leader ──▶ Followers ──▶ Leader ──ack──▶ Producer
```

**Typical safe combination**

```text
replication factor = 3
min.insync.replicas = 2
acks = all
```

This means: *a write is acknowledged only after at least 2 replicas have it*, so a single broker failure cannot lose acknowledged data.

**Other producer reliability settings**

| Setting | Purpose |
| --- | --- |
| `retries` | Retry failed sends automatically |
| `enable.idempotence=true` | Prevents duplicates caused by retries (default on in recent clients) |
| `linger.ms`, `batch.size` | Batch small records for throughput |
| `compression.type` | Compress batches (`gzip`, `snappy`, `lz4`, `zstd`) |

---

### 9. The controller

In every Kafka cluster, **one broker** acts as the **controller**.

**Controller's jobs**

1. Detect when brokers join or fail.
2. **Elect partition leaders** (for example, after a failure).
3. Keep the cluster-wide **metadata** (topics, partitions, replicas) consistent.

```text
Brokers:  [B1] [B2*] [B3]        B2* = currently the controller
If B2 dies → another broker is elected controller.
```

The controller is **not a separate machine**. It is an ordinary broker with the extra responsibility.

---

### 10. ZooKeeper: the coordinator

In classic Kafka (and in **your version 3.4.0 default setup**), **Apache ZooKeeper** keeps the cluster's shared state.

**What ZooKeeper stores and does**

| Task | Details |
| --- | --- |
| Broker registry | Which brokers are alive |
| Controller election | Decides which broker is the controller |
| Topic metadata | Topics, partitions, replica placement, configuration |
| Health detection | Brokers send heartbeats; missing heartbeats = broker considered dead |

```text
 Broker 1 ─┐
 Broker 2 ─┼──── heartbeats / metadata ────▶ ZooKeeper (port 2181)
 Broker 3 ─┘
```

**Practical consequences**

1. ZooKeeper must be started **before** the Kafka broker.
2. ZooKeeper default port: **2181**; Kafka broker default port: **9092**.
3. Real clusters run ZooKeeper as an **ensemble** of 3 or 5 servers; for learning, **one** is fine.
4. In server.properties: `zookeeper.connect=localhost:2181`.

---

### 11. KRaft: the ZooKeeper-free future

Running two systems (ZooKeeper + Kafka) is extra work. **KRaft** (Kafka Raft) lets Kafka manage its own metadata with built-in controller nodes.

| | ZooKeeper mode | KRaft mode |
| --- | --- | --- |
| Metadata stored in | ZooKeeper | Kafka's internal metadata log |
| Systems to run | ZooKeeper **and** Kafka | **Kafka only** |
| Status in Kafka 3.4 | Default, widely used | Available (production-ready since 3.3) |
| Status in newer versions | Removed in Kafka 4.0 | The only mode |
| Start-up | Start ZooKeeper, then Kafka | Format storage once, start Kafka |

> **For your exam and lab:** learn the **ZooKeeper architecture** first because your syllabus says "cluster architecture" and your installed version uses it by default. Know that **KRaft is the modern replacement**.

---

### 12. Consumer groups inside the cluster

Two special pieces of machinery run inside the cluster for consumer groups.

**1. Group coordinator**

- One broker acts as the **group coordinator** for each consumer group.
- It tracks which consumers are alive (heartbeats) and assigns partitions.

**2. Rebalancing**

A **rebalance** redistributes partitions among consumers when:

- A consumer **joins** the group,
- A consumer **leaves or crashes**,
- The number of partitions **changes**.

```text
Before:  C1 → P0,P1     C2 → P2
C2 crashes  💥
After:   C1 → P0,P1,P2
```

**3. Where are offsets saved?**

Consumers commit their positions to an internal topic named **`__consumer_offsets`**. That is why a consumer can restart and continue where it stopped.

```text
Group "dashboard" committed:  sales-P0 → offset 42, sales-P1 → offset 37, sales-P2 → offset 40
(stored inside Kafka itself in __consumer_offsets)
```

---

### 13. Storage: logs, segments and files

Each partition is a **folder** on the broker's disk.

```text
C:\kafka\data\kafka-logs\
 ├── sales-0\                 ← partition 0 of topic "sales"
 │     ├── 00000000000000000000.log        ← the records
 │     ├── 00000000000000000000.index      ← offset → file position
 │     ├── 00000000000000000000.timeindex  ← timestamp → offset
 │     └── leader-epoch-checkpoint
 ├── sales-1\
 ├── sales-2\
 └── __consumer_offsets-0\ ... (internal topic, 50 partitions by default)
```

**Segments**

A partition's log is cut into **segments** (files). The active segment receives new writes; old segments are closed.

```text
sales-0\
   00000000000000000000.log   (offsets 0 .. 999)      ← closed
   00000000000000001000.log   (offsets 1000 .. 1999)  ← closed
   00000000000000002000.log   (offsets 2000 ... )     ← ACTIVE
```

The file name is the **first offset** stored in that segment.

**Why segments?**

1. Deleting old data = **deleting whole files** (very fast).
2. Reads use the **index** to jump straight to an offset.
3. Writing is **sequential append**, which is fast even on ordinary disks.

**Why is Kafka so fast? A few reasons**

| Reason | Explanation |
| --- | --- |
| Sequential disk I/O | Appends only; no random writes |
| Page cache | Uses the operating system's memory cache |
| Batching | Groups many records per request |
| Zero-copy | Sends data from disk cache to network without extra copies |
| Partitioning | Work spread across brokers |

---

### 14. Retention and cleanup policies

`log.cleanup.policy` decides how old data is removed.

| Policy | Behaviour | Typical use |
| --- | --- | --- |
| **delete** (default) | Remove segments older than the retention time or beyond the size limit | Event streams, logs |
| **compact** | Keep only the **latest record per key** | "Current state" topics (for example latest address of a customer) |

**Retention settings**

| Setting | Meaning | Default |
| --- | --- | --- |
| `log.retention.hours` | Keep data this long | 168 hours (7 days) |
| `log.retention.bytes` | Maximum size per partition | -1 (no limit) |
| `log.segment.bytes` | Size at which a new segment starts | 1 GB |

**Compaction illustration**

```text
Before:  K1:A  K2:B  K1:C  K3:D  K2:E
After:   K1:C  K3:D  K2:E       (only the latest value for each key survives)
```

---

### 15. Reading a `--describe` output

⚠️ **Typical output** for a topic created with 3 partitions and replication factor 3 on a 3-broker cluster:

```text
Topic: sales   TopicId: 5kQ0xT1ZQ6eWk9Yw1n8y9A   PartitionCount: 3   ReplicationFactor: 3   Configs:
    Topic: sales   Partition: 0   Leader: 1   Replicas: 1,2,3   Isr: 1,2,3
    Topic: sales   Partition: 1   Leader: 2   Replicas: 2,3,1   Isr: 2,3,1
    Topic: sales   Partition: 2   Leader: 3   Replicas: 3,1,2   Isr: 3,1,2
```

**How to read one line**

```text
Partition: 1   Leader: 2   Replicas: 2,3,1   Isr: 2,3,1
     │            │             │                │
 partition no.  broker that    brokers holding   replicas currently
                leads it       a copy            in sync
```

| Column | Meaning |
| --- | --- |
| `PartitionCount` | Number of partitions |
| `ReplicationFactor` | Copies per partition |
| `Leader` | Broker id of the leader |
| `Replicas` | All brokers holding the partition (first one is the preferred leader) |
| `Isr` | Replicas that are in sync right now |

**Notice:** leaders are **different** brokers (1, 2, 3), so the load is balanced.

**After broker 2 fails** (⚠️ typical):

```text
    Topic: sales   Partition: 1   Leader: 3   Replicas: 2,3,1   Isr: 3,1
```

The leader moved to 3, and `Isr` no longer contains 2. This is the failover from Section 7, visible in a real command.

**Your single-broker practice setup** will look like:

```text
Topic: sales   PartitionCount: 3   ReplicationFactor: 1
    Topic: sales   Partition: 0   Leader: 0   Replicas: 0   Isr: 0
    Topic: sales   Partition: 1   Leader: 0   Replicas: 0   Isr: 0
    Topic: sales   Partition: 2   Leader: 0   Replicas: 0   Isr: 0
```

All partitions are on broker 0 with no copies.

---

### 16. Important broker configurations

These are in `config\server.properties`. We edit some of them in Phase 4.

| Property | Meaning | Learning value |
| --- | --- | --- |
| `broker.id` | Unique id of this broker | `0` |
| `listeners` | Address and port the broker listens on | `PLAINTEXT://localhost:9092` |
| `log.dirs` | Folder where partitions are stored | `C:/kafka/data/kafka-logs` |
| `zookeeper.connect` | Address of ZooKeeper | `localhost:2181` |
| `num.partitions` | Default partitions for auto-created topics | `1` |
| `default.replication.factor` | Default copies | `1` (single broker) |
| `offsets.topic.replication.factor` | Copies of `__consumer_offsets` | `1` (single broker) |
| `log.retention.hours` | How long data is kept | `168` |
| `auto.create.topics.enable` | Create topic automatically on first use | `true` |
| `delete.topic.enable` | Allow topic deletion | `true` |

---

### 17. Sizing rules of thumb

| Question | Guideline |
| --- | --- |
| How many partitions? | At least the number of consumers you want to run **in parallel** in one group |
| How many brokers? | Minimum 3 for a real cluster (for replication factor 3) |
| Replication factor? | 3 in production, 1 for single-machine learning |
| More partitions always better? | No. Too many increase overhead and recovery time |
| Key choice? | Pick a key with many distinct values so load spreads evenly |
| Hot key problem | If 90% of records share one key, one partition receives 90% of the load |

---

### 17b. Delivery semantics revisited

Now that we know acknowledgements, offsets and commits, we can state the guarantees precisely.

| Semantics | How it happens |
| --- | --- |
| **At most once** | Consumer commits the offset **before** processing. If it crashes, the record is skipped |
| **At least once** | Consumer commits **after** processing. If it crashes before committing, the record is processed again (**duplicates possible**) |
| **Exactly once** | Idempotent producer + transactions in Kafka, or an **idempotent sink** with checkpointed offsets (as Spark Structured Streaming does) |

Most pipelines use **at-least-once delivery + idempotent processing** to get effectively-once results.

---

### 18. Kafka cluster and Spark cluster together

They are **two separate clusters** that meet only through the network.

```text
   KAFKA CLUSTER (storage + transport)             SPARK CLUSTER (processing)
 ┌────────────────────────────────────┐       ┌────────────────────────────────┐
 │ Broker 1   Broker 2   Broker 3     │       │ Driver                          │
 │  topic "sales" (P0, P1, P2)        │──────▶│  Executor 1 ← reads P0          │
 │                                    │ reads │  Executor 2 ← reads P1          │
 │                                    │       │  Executor 3 ← reads P2          │
 └────────────────────────────────────┘       └────────────────────────────────┘
```

**Key idea:** Spark creates **one task per Kafka partition**, so Kafka partitions directly decide Spark's reading parallelism. More partitions → more parallel Spark tasks (Phase 7).

On your laptop both clusters shrink to a single machine: one Kafka broker plus Spark in local mode.

---

### 19. Paper exercises

**Exercise 1.** A cluster has 3 brokers. Can you create a topic with replication factor 4? Why?

**Exercise 2.** Topic with RF = 3, `min.insync.replicas = 2`, `acks=all`. Two brokers fail. Can producers still write? Why?

**Exercise 3.** A partition's `Replicas: 1,2,3` and `Isr: 1,3`, `Leader: 1`. Broker 1 crashes. Which broker can become leader and why not broker 2?

**Exercise 4.** Which `acks` value would you choose for a log of mouse-click events where losing a few is acceptable? For bank transactions?

**Exercise 5.** A consumer group commits offsets. Where does Kafka store them?

**Answers**

1. **No.** Replication factor cannot exceed the number of brokers.
2. **No.** Only 1 replica remains, which is below `min.insync.replicas = 2`, so writes with `acks=all` are rejected.
3. Broker **3** becomes leader. Broker 2 is not in the ISR, so it may be missing committed records.
4. Clicks: `acks=0` or `1` (speed matters). Bank transactions: `acks=all`.
5. In the internal topic **`__consumer_offsets`**.

---

### 20. Phase 3 — common confusions

| Confusion | Clarification |
| --- | --- |
| "Followers serve consumers too" | By default all reads and writes go to the **leader**. |
| "Replication factor = number of brokers" | RF is the number of **copies** and must be **≤ brokers**. |
| "Controller is a separate server" | It is an ordinary **broker** with an extra role. |
| "ZooKeeper stores the messages" | ZooKeeper stores only **metadata**. Messages are on the brokers. |
| "bootstrap.servers must list all brokers" | Only **one or more entry points** are needed. |
| "More replicas = faster" | More replicas = **safer** but slightly slower writes. |
| "ISR and replicas are the same" | ISR is the **in-sync subset** of the replicas. |

---

### 21. Phase 3 — viva questions and answers

#### Q1. What is a Kafka broker?

A Kafka server process that stores partitions, receives records from producers and serves them to consumers.

#### Q2. What is a Kafka cluster?

A group of brokers working together to store topic partitions and replicate them.

#### Q3. What are bootstrap servers?

The initial brokers a client connects to in order to discover the full cluster metadata.

#### Q4. What is replication and why is it needed?

Keeping multiple copies of each partition on different brokers, so that data survives broker failures.

#### Q5. What are leader and follower replicas?

The leader handles all reads and writes of a partition; followers copy the leader's data.

#### Q6. What is the replication factor, and what limits it?

The number of copies of each partition. It cannot exceed the number of brokers.

#### Q7. What is ISR?

The set of replicas that are fully in sync with the leader. Only ISR members can be elected leader.

#### Q8. What happens when a leader broker fails?

The controller elects a new leader from the ISR; clients refresh metadata and continue.

#### Q9. What is the role of the controller?

One broker that detects failures, elects partition leaders and manages cluster metadata.

#### Q10. What is the role of ZooKeeper in Kafka?

It stores cluster metadata, tracks live brokers via heartbeats and helps elect the controller. (It does not store the messages.)

#### Q11. What is KRaft?

Kafka's built-in Raft-based metadata management, which removes the need for ZooKeeper.

#### Q12. Explain the `acks` setting.

`0` = no acknowledgement, `1` = leader acknowledges, `all` = all in-sync replicas acknowledge (safest).

#### Q13. What is `min.insync.replicas`?

The minimum number of in-sync replicas required for a write to succeed when the producer uses `acks=all`.

#### Q14. What is a consumer group rebalance?

Redistribution of partitions among group members when a consumer joins, leaves or fails.

#### Q15. Where are consumer offsets stored?

In the internal Kafka topic `__consumer_offsets`.

#### Q16. How is a partition stored on disk?

As a folder (`topic-partition`) containing segment files: `.log` (records), `.index` and `.timeindex`.

#### Q17. What are the two cleanup policies?

`delete` (remove old segments by time or size) and `compact` (keep the latest record per key).

#### Q18. Why is Kafka fast?

Sequential disk writes, OS page cache, batching, zero-copy transfer and partition-level parallelism.

#### Q19. How do Kafka partitions relate to Spark tasks?

Spark's Kafka source creates one read task per Kafka partition, so partitions control reading parallelism.

#### Q20. How can Kafka achieve at-least-once delivery?

By committing consumer offsets only after processing the records, so a crash causes reprocessing rather than loss.

---

### 22. Phase 3 — final master recap

```text
KAFKA CLUSTER ARCHITECTURE
  Broker      = server (broker.id, port 9092, log.dirs)
  Cluster     = several brokers; clients start from bootstrap.servers
  Partition   → leader (reads/writes) + followers (copies)
  RF          = number of copies (≤ brokers)
  ISR         = replicas in sync; new leader chosen from ISR
  Controller  = one broker that elects leaders
  ZooKeeper   = metadata + liveness (port 2181)   |  KRaft = built-in replacement
  acks        = 0 / 1 / all   (+ min.insync.replicas)
  Storage     = topic-partition folders → segments (.log .index .timeindex)
  Retention   = delete (time/size) or compact (latest per key)
  Offsets     = saved in __consumer_offsets
```

**Seven facts to memorise**

1. One leader per partition; followers copy it.
2. RF ≤ number of brokers.
3. New leader always comes from the ISR.
4. ZooKeeper stores metadata, not messages.
5. `acks=all` + `min.insync.replicas=2` + RF=3 is the safe combination.
6. Offsets live in `__consumer_offsets`.
7. Kafka partitions decide Spark read parallelism.

### Phase 3 completed

---

> **Next → Phase 4: Kafka Installation on Windows** (verifying Java/Kafka/Spark, folder layout, editing configs, starting ZooKeeper and the broker, smoke test, multi-broker option, KRaft option, Windows pitfalls and the Spark connector). File: `4_Unit5_Kafka_Installation_Windows.md`
