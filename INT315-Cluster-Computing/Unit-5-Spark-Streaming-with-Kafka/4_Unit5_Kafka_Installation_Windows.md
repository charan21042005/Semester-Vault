# Unit V — Phase 4: Apache Kafka Installation (Windows)

> **Continuing from Phase 3.** We understand the architecture. Now we bring Kafka alive on your laptop: verify the environment, configure folders, start ZooKeeper and the broker, run a smoke test, and prepare Spark's Kafka connector.
>
> **Verification note (honest):** I cannot run a Kafka broker in my own environment, so every Kafka command here follows the official **Kafka 3.4.0** scripts and syntax. Outputs labelled ⚠️ **Typical output** are what Kafka normally prints; IDs, timestamps and paths on your machine will differ. If something looks different, use the troubleshooting table in Section 13.

---

## Phase 4 — Roadmap

| # | Topic |
| --- | --- |
| 1 | What we are installing |
| 2 | Verifying your environment |
| 3 | Getting Kafka (download and extract) |
| 4 | Folder structure of Kafka |
| 5 | Environment variables and PATH |
| 6 | Editing the configuration files |
| 7 | Starting ZooKeeper |
| 8 | Starting the Kafka broker |
| 9 | Verifying that everything is running |
| 10 | The smoke test |
| 11 | Start order, stop order and helper scripts |
| 12 | Where data lives and how to reset |
| 13 | Windows pitfalls and troubleshooting |
| 14 | Optional: a 3-broker cluster on one machine |
| 15 | Optional: KRaft mode |
| 16 | Preparing Spark: the Kafka connector |
| 17 | Preparing Python: `kafka-python` |
| 18 | Daily start-up routine and checklist |
| 19 | Practical scenarios |
| 20 | Viva questions |
| 21 | Master revision |

---

### 1. What we are installing

```text
┌─────────────────────────────────────────────────────────┐
│  YOUR LAPTOP                                            │
│                                                         │
│   ZooKeeper  ←──────  Kafka broker (id 0)               │
│   port 2181           port 9092                         │
│                          ▲        ▲                     │
│              producer ───┘        └─── consumer / Spark │
└─────────────────────────────────────────────────────────┘
```

Kafka comes as a **zip/tar download with no installer**. "Installing" means: extract, configure two small files, start two programs.

**Programs involved**

| Program | Role | Port |
| --- | --- | --- |
| ZooKeeper | Metadata and coordination | 2181 |
| Kafka broker | Stores and serves records | 9092 |
| Producer / consumer tools | Console programs we run when needed | none |

---

### 2. Verifying your environment

Open **Windows CMD** and run each check.

**Java**

```powershell
java -version
```

```text
java version "1.8.0_501"
Java(TM) SE Runtime Environment (build 1.8.0_501-b08)
Java HotSpot(TM) 64-Bit Server VM (build 25.501-b08, mixed mode)
```

**Python**

```powershell
python --version
```

```text
Python 3.10.11
```

**Kafka**

```powershell
kafka-topics.bat --version
```

```text
3.4.0 (Commit:2e1947d240607d53)
```

**Spark and Scala**

```powershell
spark-shell
```

The banner shows the versions:

```text
Spark version 3.5.9
Using Scala version 2.13.8 (Java HotSpot(TM) 64-Bit Server VM, Java 1.8.0_501)
```

**Your verified environment**

| Tool | Version | Meaning for Unit V |
| --- | --- | --- |
| Java | 1.8.0_501 | Runs Kafka, ZooKeeper and Spark |
| Python | 3.10.11 | For `kafka-python` producer/consumer scripts |
| Kafka | 3.4.0 | Broker and console tools |
| Spark | 3.5.9 | Structured Streaming engine |
| Scala | 2.13.8 | Decides connector suffix `_2.13` |

> If `kafka-topics.bat --version` says *'kafka-topics.bat' is not recognized*, Kafka is installed but `C:\kafka\bin\windows` is not on your PATH. We fix that in Section 5. Until then, use the full path or `cd C:\kafka` and run `bin\windows\kafka-topics.bat`.

**Compatibility summary**

| Pair | Status |
| --- | --- |
| Java 8 + Kafka 3.4.0 | ✅ Works (Java 8 support is deprecated but present) |
| Java 8 + Spark 3.5.9 | ✅ Works |
| Spark 3.5.9 connector + Kafka 3.4.0 broker | ✅ Compatible (connector uses `kafka-clients` 3.4.x) |
| Python 3.10 + `kafka-python` | ✅ Works |

---

### 3. Getting Kafka (download and extract)

*(Skip if Kafka is already installed on your system, since your `--version` check already worked.)*

1. Open `https://kafka.apache.org/downloads`.
2. Older releases such as 3.4.0 are at `https://archive.apache.org/dist/kafka/3.4.0/`.
3. Download **`kafka_2.13-3.4.0.tgz`** (the "Scala 2.13" binary).

**Reading the file name**

```text
kafka_2.13-3.4.0.tgz
      │     │
      │     └── Kafka version 3.4.0
      └──────── Scala version the Kafka code was compiled with
```

> This Scala 2.13 belongs to **Kafka itself**. It is independent of Spark's Scala. It just happens to match yours.

4. **Extract** to a **short path with no spaces**:

```powershell
cd C:\
mkdir kafka
tar -xvzf "C:\Users\<you>\Downloads\kafka_2.13-3.4.0.tgz" -C C:\kafka --strip-components=1
```

`tar` is built into Windows 10/11. `--strip-components=1` removes the extra top folder, so files land in `C:\kafka\bin`, not `C:\kafka\kafka_2.13-3.4.0\bin`. (7-Zip also works: extract twice, `.tgz` → `.tar` → folder.)

> **Why `C:\kafka` and not Downloads or Program Files?** Windows has a command-line length limit, and Kafka's start scripts build a long class path. A short folder name avoids the error *"The input line is too long"*.

---

### 4. Folder structure of Kafka

```text
C:\kafka
 ├── bin\
 │    ├── kafka-topics.sh ...            (Linux scripts, ignore on Windows)
 │    └── windows\                       ← OUR scripts (.bat)
 │         ├── zookeeper-server-start.bat
 │         ├── kafka-server-start.bat
 │         ├── kafka-topics.bat
 │         ├── kafka-console-producer.bat
 │         ├── kafka-console-consumer.bat
 │         ├── kafka-consumer-groups.bat
 │         ├── kafka-configs.bat
 │         └── kafka-storage.bat         (for KRaft)
 ├── config\
 │    ├── zookeeper.properties           ← edit
 │    ├── server.properties              ← edit
 │    ├── producer.properties
 │    ├── consumer.properties
 │    └── kraft\server.properties        (KRaft mode)
 ├── libs\                               (all the .jar files)
 ├── logs\                               (the broker's own diagnostic logs)
 └── LICENSE, NOTICE, site-docs\
```

| Folder | Purpose |
| --- | --- |
| `bin\windows` | The `.bat` commands you will type |
| `config` | Settings files |
| `libs` | Java libraries Kafka needs |
| `logs` | Kafka's **server logs** (not your data) |

> **Important difference:** `C:\kafka\logs` holds the broker's **diagnostic log files**. Your **message data** is in `log.dirs`, which we set in Section 6. Do not confuse the two.

---

### 5. Environment variables and PATH

**5.1 Check `JAVA_HOME`**

```powershell
echo %JAVA_HOME%
```

Expected: something like `C:\Program Files\Java\jdk1.8.0_501`. If it is empty, set it (System Properties → Environment Variables → New). Spark already needs Java, so it is likely set.

**5.2 Add Kafka's script folder to PATH** (optional but convenient)

System Properties → Environment Variables → `Path` → New → add:

```text
C:\kafka\bin\windows
```

Open a **new** CMD window and test:

```powershell
kafka-topics.bat --version
```

```text
3.4.0 (Commit:2e1947d240607d53)
```

Now you can run `kafka-topics.bat` from any folder. The rest of these notes assume this PATH setting.

---

### 6. Editing the configuration files

Edit with Notepad (run as normal user; the folder `C:\kafka` is yours).

#### 6.1 `C:\kafka\config\zookeeper.properties`

Default content:

```properties
dataDir=/tmp/zookeeper
clientPort=2181
maxClientCnxns=0
admin.enableServer=false
```

Change the data directory to a real Windows folder:

```properties
dataDir=C:/kafka/data/zookeeper
clientPort=2181
maxClientCnxns=0
admin.enableServer=false
```

| Property | Meaning |
| --- | --- |
| `dataDir` | Where ZooKeeper keeps its snapshots |
| `clientPort` | Port Kafka uses to talk to ZooKeeper |
| `maxClientCnxns` | Max connections per client (0 = unlimited) |

#### 6.2 `C:\kafka\config\server.properties`

Find and set these lines (the rest can stay as is):

```properties
broker.id=0
listeners=PLAINTEXT://localhost:9092
log.dirs=C:/kafka/data/kafka-logs
zookeeper.connect=localhost:2181
num.partitions=1
offsets.topic.replication.factor=1
transaction.state.log.replication.factor=1
transaction.state.log.min.isr=1
log.retention.hours=168
```

| Property | Why this value |
| --- | --- |
| `broker.id=0` | Unique id of our only broker |
| `listeners=PLAINTEXT://localhost:9092` | Listen on port 9092 for local clients |
| `log.dirs=C:/kafka/data/kafka-logs` | Message data folder |
| `zookeeper.connect=localhost:2181` | Where ZooKeeper is |
| `offsets.topic.replication.factor=1` | Single broker, so one copy only |
| `log.retention.hours=168` | Keep data 7 days |

> **Windows path rule:** write paths with **forward slashes** (`C:/kafka/data/...`). A single backslash is treated as an escape character in `.properties` files and breaks the path.

> The `/tmp` defaults also work on Windows (they map to `C:\tmp`), but that folder is often cleaned by the system. A dedicated data folder is safer.

---

### 7. Starting ZooKeeper

**Open CMD window #1.**

```powershell
cd C:\kafka
bin\windows\zookeeper-server-start.bat config\zookeeper.properties
```

(With PATH set you can also write `zookeeper-server-start.bat config\zookeeper.properties` from `C:\kafka`.)

⚠️ **Typical output** (the last lines):

```text
[2026-11-18 10:00:01,123] INFO binding to port 0.0.0.0/0.0.0.0:2181 (org.apache.zookeeper.server.NIOServerCnxnFactory)
[2026-11-18 10:00:01,140] INFO Using org.apache.zookeeper.server.NIOServerCnxnFactory as server connection factory
...
```

The key line is **"binding to port ... 2181"**. The prompt does **not** return. ZooKeeper is now running. **Leave this window open.**

If Windows Defender Firewall pops up asking about Java, click **Allow access**.

---

### 8. Starting the Kafka broker

**Open CMD window #2.**

```powershell
cd C:\kafka
bin\windows\kafka-server-start.bat config\server.properties
```

⚠️ **Typical output** (last lines):

```text
[2026-11-18 10:00:10,456] INFO Registered broker 0 at path /brokers/ids/0 with addresses: PLAINTEXT://localhost:9092
[2026-11-18 10:00:10,500] INFO [KafkaServer id=0] started (kafka.server.KafkaServer)
```

The key line is **`[KafkaServer id=0] started`**. Again the prompt does not return. **Leave this window open.**

```text
CMD #1:  ZooKeeper   (running)
CMD #2:  Kafka       (running)
CMD #3:  free for your commands
```

> **Always start ZooKeeper first, then Kafka.** If you start Kafka first it will fail with *Timed out waiting for connection to ZooKeeper*.

---

### 9. Verifying that everything is running

**Open CMD window #3.**

**9.1 `jps` lists Java processes**

```powershell
jps
```

⚠️ **Typical output**

```text
4412 QuorumPeerMain
5876 Kafka
7120 Jps
```

| Name | Meaning |
| --- | --- |
| `QuorumPeerMain` | **ZooKeeper** |
| `Kafka` | The **Kafka broker** |

**9.2 Check the ports**

```powershell
netstat -ano | findstr "2181 9092"
```

⚠️ **Typical output** (`LISTENING` lines):

```text
TCP    0.0.0.0:2181      0.0.0.0:0      LISTENING    4412
TCP    127.0.0.1:9092    0.0.0.0:0      LISTENING    5876
```

Ports 2181 and 9092 are both listening, so the setup is healthy.

---

### 10. The smoke test

A quick end-to-end test to prove Kafka works. (Phase 5 explains every option in depth.)

**Step 1 — Create a topic** (CMD #3)

```powershell
kafka-topics.bat --create --topic test --bootstrap-server localhost:9092 --partitions 1 --replication-factor 1
```

⚠️ **Typical output**

```text
Created topic test.
```

**Step 2 — List topics**

```powershell
kafka-topics.bat --list --bootstrap-server localhost:9092
```

```text
test
```

**Step 3 — Start a producer** (CMD #3)

```powershell
kafka-console-producer.bat --topic test --bootstrap-server localhost:9092
```

A `>` prompt appears. Type:

```text
>hello kafka
>this is my first message
>spark will read this soon
```

Press **Enter** after each line. Keep it open.

**Step 4 — Open CMD #4 and start a consumer**

```powershell
kafka-console-consumer.bat --topic test --from-beginning --bootstrap-server localhost:9092
```

⚠️ **Typical output**

```text
hello kafka
this is my first message
spark will read this soon
```

**The messages typed in the producer window appear in the consumer window.** Type another line in the producer; it appears instantly in the consumer. 🎉

Stop the producer and consumer with **Ctrl + C**.

```text
Producer window ──▶ Kafka (topic test) ──▶ Consumer window
```

---

### 11. Start order, stop order and helper scripts

| Action | Order |
| --- | --- |
| **Start** | 1) ZooKeeper → 2) Kafka broker → 3) producers/consumers/Spark |
| **Stop** | 1) producers/consumers/Spark → 2) Kafka broker → 3) ZooKeeper |

Stop each by pressing **Ctrl + C** in its window (wait until the prompt returns). A *graceful* stop lets Kafka close its files cleanly.

> **Never just close the window with the X button** or shut the laptop down while Kafka is running. This can leave files in an inconsistent state.

**Helper scripts to save typing**

`C:\kafka\start-zookeeper.bat`

```bat
@echo off
cd C:\kafka
bin\windows\zookeeper-server-start.bat config\zookeeper.properties
```

`C:\kafka\start-kafka.bat`

```bat
@echo off
cd C:\kafka
bin\windows\kafka-server-start.bat config\server.properties
```

Now you start each by double-clicking, or typing `start-zookeeper.bat` in a CMD.

---

### 12. Where data lives and how to reset

After the smoke test, look inside your data folder:

```text
C:\kafka\data\
 ├── zookeeper\                 ← ZooKeeper snapshots
 └── kafka-logs\
      ├── test-0\               ← our topic's partition
      │     └── 00000000000000000000.log, .index, .timeindex
      ├── __consumer_offsets-0\ ...
      ├── meta.properties       ← broker id and cluster id
      └── ...
```

**Clean reset (when things are badly broken, or to start fresh)**

1. Stop the consumers/producers, then **Kafka**, then **ZooKeeper** (Ctrl + C).
2. Delete **both** folders:

```powershell
rmdir /S /Q C:\kafka\data\kafka-logs
rmdir /S /Q C:\kafka\data\zookeeper
```

3. Start ZooKeeper, then Kafka again. All topics and messages are gone; you have a fresh cluster.

> **Rule:** always reset **both** folders together. If you delete only one, Kafka and ZooKeeper disagree about the cluster id and the broker refuses to start (Section 13).

---

### 13. Windows pitfalls and troubleshooting

| Symptom | Cause | Fix |
| --- | --- | --- |
| `'kafka-topics.bat' is not recognized` | PATH not set | Add `C:\kafka\bin\windows` or use the full path |
| **The input line is too long** / *The syntax of the command is incorrect* | Kafka folder path too long | Install in `C:\kafka` |
| `Timed out waiting for connection to ZooKeeper` | ZooKeeper not running or wrong port | Start ZooKeeper first; check `zookeeper.connect` |
| `Address already in use` / `BindException` on 9092 or 2181 | Another copy is running | Run `netstat -ano` (filter for 9092 with `findstr`), then `taskkill /PID <pid> /F`, or close the old window |
| `InconsistentClusterIdException` | Deleted only one of the data folders | Reset **both** folders (Section 12) |
| Broker crashes with `AccessDeniedException` / file-in-use errors after **deleting a topic** | A known Windows file-locking problem with topic deletion | **Do not delete topics on Windows.** Reset the data folders instead |
| `JAVA_HOME is not set` | Environment variable missing | Set it to the JDK folder |
| Topic creation: *Replication factor: 3 larger than available brokers: 1* | One broker only | Use `--replication-factor 1` |
| Consumer shows nothing | Started after messages, without `--from-beginning` | Add `--from-beginning` |
| Python/Spark cannot connect | Broker not started, or `listeners` not `localhost:9092` | Check `jps`, `server.properties` |
| Windows Firewall pop-up | Java wants network access | Click **Allow access** |
| Config path ignored | Backslashes in `.properties` | Use forward slashes `C:/kafka/...` |

> ⚠️ **Most important Windows rule:** do **not** use `kafka-topics.bat --delete` on Windows. Create new topic names instead, or reset the data folders.

---

### 14. Optional: a 3-broker cluster on one machine

This lets you **see** replication, leaders and failover from Phase 3 on your own laptop. We run three brokers on the same machine, on different ports.

**Step 1 — Copy the config three times**

```powershell
cd C:\kafka\config
copy server.properties server-0.properties
copy server.properties server-1.properties
copy server.properties server-2.properties
```

**Step 2 — Edit each file**

`server-0.properties`

```properties
broker.id=0
listeners=PLAINTEXT://localhost:9092
log.dirs=C:/kafka/data/kafka-logs-0
zookeeper.connect=localhost:2181
```

`server-1.properties`

```properties
broker.id=1
listeners=PLAINTEXT://localhost:9093
log.dirs=C:/kafka/data/kafka-logs-1
zookeeper.connect=localhost:2181
```

`server-2.properties`

```properties
broker.id=2
listeners=PLAINTEXT://localhost:9094
log.dirs=C:/kafka/data/kafka-logs-2
zookeeper.connect=localhost:2181
```

**What must differ per broker:** `broker.id`, the **port** and `log.dirs`.

For a 3-broker cluster, you can also set `offsets.topic.replication.factor=3` (and `transaction.state.log.replication.factor=3`, `transaction.state.log.min.isr=2`) in each file.

**Step 3 — Start ZooKeeper, then each broker in its own CMD**

```powershell
bin\windows\kafka-server-start.bat config\server-0.properties
bin\windows\kafka-server-start.bat config\server-1.properties
bin\windows\kafka-server-start.bat config\server-2.properties
```

**Step 4 — Create a replicated topic**

```powershell
kafka-topics.bat --create --topic sales --bootstrap-server localhost:9092 --partitions 3 --replication-factor 3
kafka-topics.bat --describe --topic sales --bootstrap-server localhost:9092
```

⚠️ **Typical output**

```text
Topic: sales   PartitionCount: 3   ReplicationFactor: 3
    Topic: sales   Partition: 0   Leader: 0   Replicas: 0,1,2   Isr: 0,1,2
    Topic: sales   Partition: 1   Leader: 1   Replicas: 1,2,0   Isr: 1,2,0
    Topic: sales   Partition: 2   Leader: 2   Replicas: 2,0,1   Isr: 2,0,1
```

**Step 5 — Watch failover (the Phase 3 story, live)**

Press **Ctrl + C** in the window of broker 1, then describe again (use port 9092):

```powershell
kafka-topics.bat --describe --topic sales --bootstrap-server localhost:9092
```

⚠️ **Typical output**

```text
    Topic: sales   Partition: 1   Leader: 2   Replicas: 1,2,0   Isr: 2,0
```

The leader of partition 1 moved to broker 2, and broker 1 vanished from the ISR. Start broker 1 again and, after a few seconds, `Isr` returns to `1,2,0`.

> Running three JVMs plus ZooKeeper needs RAM. If your laptop is slow, stay with a single broker; the practicals do not need the cluster.

---

### 15. Optional: KRaft mode

KRaft runs Kafka **without ZooKeeper** (Phase 3, Section 11). In Kafka 3.4 the steps are:

```powershell
cd C:\kafka

REM 1) generate a cluster id
bin\windows\kafka-storage.bat random-uuid

REM 2) format the storage with that id (replace <UUID>)
bin\windows\kafka-storage.bat format -t <UUID> -c config\kraft\server.properties

REM 3) start the broker (no ZooKeeper needed)
bin\windows\kafka-server-start.bat config\kraft\server.properties
```

Edit `log.dirs` in `config\kraft\server.properties` to a Windows folder such as `C:/kafka/data/kraft-logs` first.

| | ZooKeeper mode | KRaft mode |
| --- | --- | --- |
| Programs to start | 2 (ZooKeeper + Kafka) | 1 (Kafka) |
| Config file | `config\server.properties` | `config\kraft\server.properties` |
| One-time step | none | `kafka-storage format` |

> **Do not mix them.** Use either ZooKeeper mode or KRaft mode with separate data folders. The rest of Unit V works the same in both, because clients only see `localhost:9092`. We use **ZooKeeper mode** in these notes.

---

### 16. Preparing Spark: the Kafka connector

Spark does **not** include the Kafka source by default. We attach it when starting the shell.

**The connector coordinate for your machine**

```text
org.apache.spark:spark-sql-kafka-0-10_2.13:3.5.9
         │              │          │    │      │
      group        artifact   Kafka   Scala  Spark
                              0.10+  2.13    3.5.9
```

| Part | Must match |
| --- | --- |
| `_2.13` | Spark's **Scala** version (2.13.8) |
| `3.5.9` | Spark's own version |

**Start Spark with the connector**

```powershell
spark-shell --packages org.apache.spark:spark-sql-kafka-0-10_2.13:3.5.9
```

**First run:** Spark downloads the connector and its dependencies from the internet (Maven Central) into a cache, normally `C:\Users\<you>\.ivy2`. You will see lines like:

```text
org.apache.spark#spark-sql-kafka-0-10_2.13 added as a dependency
:: resolving dependencies ...
```

**Later runs** reuse the cache, so no internet is needed.

**Check that the Kafka source is available**

```scala
spark.readStream.format("kafka")
```

- If you see a result like `DataStreamReader@...`, the connector is loaded. ✅
- If you later get `Failed to find data source: kafka`, the package was not loaded (Phase 7 troubleshooting).

**Dependencies pulled in automatically** (for your information)

| Library | Purpose |
| --- | --- |
| `spark-sql-kafka-0-10_2.13` | The Kafka source and sink |
| `spark-token-provider-kafka-0-10_2.13` | Security token support |
| `kafka-clients` (3.4.x) | The Java Kafka client |
| `commons-pool2` | Connection pooling |

**Offline fallback (no internet in the lab):** download those jars once on a machine with internet, copy them to a folder such as `C:\spark-jars`, and start Spark with:

```powershell
spark-shell --jars C:\spark-jars\spark-sql-kafka-0-10_2.13-3.5.9.jar,C:\spark-jars\spark-token-provider-kafka-0-10_2.13-3.5.9.jar,C:\spark-jars\kafka-clients-3.4.1.jar,C:\spark-jars\commons-pool2-2.11.1.jar
```

(Check the exact dependency versions on the connector's Maven page, under *Dependencies*.)

**Useful extra option for laptops**

```powershell
spark-shell --packages org.apache.spark:spark-sql-kafka-0-10_2.13:3.5.9 --conf spark.sql.shuffle.partitions=4
```

This reduces the default 200 shuffle partitions to 4, which makes small streaming jobs much faster on a laptop.

> **Optional shortcut:** save the long command in `C:\kafka\spark-kafka.bat` and run it each time.

---

### 17. Preparing Python: `kafka-python`

For Python producer/consumer scripts (Phase 5):

```powershell
pip install kafka-python
```

Check:

```powershell
python -c "import kafka; print(kafka.__version__)"
```

If a version number prints, Python is ready. (Your Python 3.10.11 is supported.)

---

### 18. Daily start-up routine and checklist

**Each practice session**

```text
1. CMD #1  → start-zookeeper.bat         (wait for "binding to port 2181")
2. CMD #2  → start-kafka.bat             (wait for "[KafkaServer id=0] started")
3. CMD #3  → kafka-topics / producer / python scripts
4. CMD #4  → consumer or spark-shell --packages ...
```

**Checklist before starting any streaming practical**

| ✔ | Check |
| --- | --- |
| ☐ | `jps` shows `QuorumPeerMain` and `Kafka` |
| ☐ | Topic exists: `kafka-topics.bat --list --bootstrap-server localhost:9092` |
| ☐ | Spark started with `--packages ...spark-sql-kafka-0-10_2.13:3.5.9` |
| ☐ | Using `localhost:9092` everywhere |
| ☐ | Fresh checkpoint folder for each new streaming query (Phase 6) |

**Shutdown routine**

```text
Ctrl+C on consumers/producers → exit spark-shell (:quit) → Ctrl+C on Kafka → Ctrl+C on ZooKeeper
```

---

### 19. Practical scenarios — Phase 4 practice

1. **Verify all versions** (Java, Python, Kafka, Spark/Scala) and write them down.
2. **Start ZooKeeper** and find the line showing port 2181.
3. **Start the broker** and find the `started` line.
4. Run **`jps`** and identify both processes.
5. Run **`netstat`** and confirm ports 2181 and 9092 are listening.
6. **Create a topic** `hello` with 1 partition and replication factor 1.
7. **List topics** and confirm `hello` exists.
8. **Send three messages** from the console producer; **read them** with `--from-beginning`.
9. **Stop everything in the correct order**, then start again and confirm that old messages are still there (they live in `data\kafka-logs`).
10. **Reset both data folders**, restart, and confirm that the topic list is now empty.
11. *(Optional)* Build the 3-broker cluster and observe a leader change after stopping one broker.
12. Start `spark-shell` with the connector and run `spark.readStream.format("kafka")`.

---

### 20. Phase 4 — viva questions and answers

#### Q1. What do we need to run before starting a Kafka broker (in ZooKeeper mode)?

ZooKeeper. The broker registers with it at start-up.

#### Q2. What are the default ports of ZooKeeper and Kafka?

ZooKeeper 2181; Kafka broker 9092.

#### Q3. Which two programs appear in `jps` for a running setup?

`QuorumPeerMain` (ZooKeeper) and `Kafka` (the broker).

#### Q4. Which configuration property sets where Kafka stores message data?

`log.dirs` in `server.properties`.

#### Q5. What is the purpose of `zookeeper.connect`?

It tells the broker where ZooKeeper is running (for example `localhost:2181`).

#### Q6. What does `broker.id` do?

It gives each broker a unique integer identity in the cluster.

#### Q7. How do you check the installed Kafka version?

`kafka-topics.bat --version`.

#### Q8. Why should Kafka be installed in a short path like `C:\kafka` on Windows?

To avoid the *"input line is too long"* error caused by long class paths.

#### Q9. What is the correct start and stop order?

Start ZooKeeper, then Kafka. Stop in reverse order: clients, Kafka, ZooKeeper.

#### Q10. Which Spark package connects Spark 3.5.9 (Scala 2.13) to Kafka?

`org.apache.spark:spark-sql-kafka-0-10_2.13:3.5.9`.

#### Q11. What does `_2.13` in the package name mean?

The Scala binary version that Spark was built with. It must match Spark's Scala version.

#### Q12. How do we attach the Kafka connector to the Spark shell?

`spark-shell --packages org.apache.spark:spark-sql-kafka-0-10_2.13:3.5.9`.

#### Q13. Why must replication factor be 1 on our single-broker setup?

Replication factor cannot exceed the number of brokers.

#### Q14. Why is topic deletion avoided on Windows?

A known file-locking problem can crash the broker; resetting the data folders is the safe alternative.

#### Q15. What is KRaft mode and how does start-up differ?

Kafka without ZooKeeper. We format storage once with `kafka-storage format` and then start only the broker.

#### Q16. What is the difference between `C:\kafka\logs` and `log.dirs`?

`logs` holds the broker's diagnostic log files; `log.dirs` holds the actual topic data (messages).

---

### 21. Phase 4 — final master recap

```text
INSTALLATION AT A GLANCE
  Folder        C:\kafka          (short path, no spaces)
  Edit          zookeeper.properties → dataDir=C:/kafka/data/zookeeper
                server.properties    → broker.id=0, listeners=PLAINTEXT://localhost:9092,
                                       log.dirs=C:/kafka/data/kafka-logs,
                                       zookeeper.connect=localhost:2181
  Start         1) zookeeper-server-start.bat config\zookeeper.properties
                2) kafka-server-start.bat config\server.properties
  Verify        jps → QuorumPeerMain + Kafka ; ports 2181 + 9092
  Smoke test    create topic → console producer → console consumer
  Spark         spark-shell --packages org.apache.spark:spark-sql-kafka-0-10_2.13:3.5.9
  Python        pip install kafka-python
  Windows rule  never delete topics; reset BOTH data folders instead
```

**Commands to remember**

```powershell
kafka-topics.bat --version
zookeeper-server-start.bat config\zookeeper.properties
kafka-server-start.bat config\server.properties
kafka-topics.bat --create --topic test --bootstrap-server localhost:9092 --partitions 1 --replication-factor 1
kafka-topics.bat --list --bootstrap-server localhost:9092
spark-shell --packages org.apache.spark:spark-sql-kafka-0-10_2.13:3.5.9
```

### Phase 4 completed

---

> **Next → Phase 5: Producer and Consumer Messaging Model** (topic commands, console producer/consumer, keys, consumer groups, offsets and lag, Python and Scala producers/consumers, producer and consumer settings). File: `5_Unit5_Producer_Consumer_Messaging_Model.md`
