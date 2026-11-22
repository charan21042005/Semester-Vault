# 🟢 CSE494 — INTELLIGENT NOSQL DATABASES: THE COMPLETE PRACTICAL MANUAL + VIVA & INTERVIEW GUIDE

> **Covered in this document:** How to attack any lab / practical exam (the Aim → Theory → Procedure → Code → Output → Explanation → Viva format) → one common sample dataset used everywhere → **Practical 1** Install and prepare the MongoDB environment (local, Docker, Atlas, mongosh, Compass) → **Practical 2** Create a NoSQL database and compare with SQL → **Practical 3** CRUD operations on documents → **Practical 4** BSON data types → **Practical 5** Modelling one-to-one / one-to-many / many-to-many → **Practical 6** Schema validation and constraints → **Practical 7** Indexes and performance analysis with `explain()` → **Practical 8** Aggregation pipeline using stages → **Practical 9** Integrate MongoDB with Node.js (driver + Mongoose) → **Practical 10** Connect with Python PyMongo → **Bonus practicals** (AI predictive analytics and profiler-based query optimisation, mini project) → **Master viva bank for Units I–VI with full "best-fit" answers** → **industry interview round** (design, performance, production, scenario and coding questions) → how to answer like a professional → lab record template, troubleshooting table, final cheat-sheet.

---

Absolutely mate. 😤🔥 This is the **one-file practical bible** for CSE494.

Your syllabus lists **10 practicals**. Your exam will test them in a lab, and your viva will ask *"why did you do it that way?"* — and in a job interview the same topics come back as *"tell me how you'd design / debug / scale this."*

So every practical here is written in **three layers**:

```
LAYER 1  DO IT      exact steps + code you can type + the output you should see
LAYER 2  UNDERSTAND  line-by-line explanation, theory, diagrams, why it works
LAYER 3  DEFEND IT   viva questions with full answers + the industry-interview angle
```

> **Honest notes before we start**
> 1. I could not run a live MongoDB server while writing this, so every *expected output* is what MongoDB prints for these exact inputs (I computed the aggregation results independently from the sample data). Your `ObjectId` values, timings and sizes **will** differ — that is normal. Where a detail depends on your MongoDB version (for example, the shape of `explain()` output), I say so.
> 2. Product-specific things (Atlas menus, install package names, version numbers) change. Use the **concepts** from here and confirm the click-paths against the current official docs.
> 3. Your faculty's lab manual decides the exact format/marks. Use this as the knowledge base and adapt the write-up.

---

# 🧭 SECTION 0 — HOW TO ATTACK ANY PRACTICAL

## 0.1 The standard write-up format (examiners love structure)

```
1. AIM          one sentence: "To ..."
2. THEORY       short definition + diagram (this guide gives you medium/long versions)
3. REQUIREMENTS software + versions
4. PROCEDURE    numbered steps
5. PROGRAM / COMMANDS   exact code
6. OUTPUT       screenshot or pasted output
7. RESULT       "Thus ... was successfully performed."
8. VIVA         be ready for 5–8 questions
```

## 0.2 The 10-minute pre-lab checklist

```
□ mongod running?          (mongosh → db.runCommand({ping:1}) → { ok: 1 })
□ Which database am I in?  (db → prints current db)
□ Atlas? IP allow-listed + DB user created + password URL-encoded?
□ Node installed?          (node -v)      □ Python installed?   (python --version)
□ Editor + terminal ready; project folder created; .env file created (NOT committed)
□ Sample data script ready (Section 0.3)
```

## 0.3 One sample dataset used throughout 📦

Most practicals reuse the same eight students so outputs are predictable and comparable.

```js
// mongosh
use college

db.students.drop()        // start clean (safe if it doesn't exist)

db.students.insertMany([
  { rollNo: "CSE2301", name: "Aarav", branch: "CSE", year: 2, cgpa: 8.4, skills: ["python","sql"],     city: "Bengaluru" },
  { rollNo: "CSE2302", name: "Diya",  branch: "CSE", year: 3, cgpa: 9.1, skills: ["python","ml"],      city: "Hyderabad" },
  { rollNo: "ECE2301", name: "Kabir", branch: "ECE", year: 2, cgpa: 7.2, skills: ["c","vlsi"],         city: "Pune" },
  { rollNo: "ECE2302", name: "Meera", branch: "ECE", year: 3, cgpa: 8.0, skills: ["matlab","python"],  city: "Chennai" },
  { rollNo: "ME2301",  name: "Rohan", branch: "ME",  year: 4, cgpa: 6.9, skills: ["cad"],              city: "Pune" },
  { rollNo: "CSE2303", name: "Sana",  branch: "CSE", year: 3, cgpa: 8.8, skills: ["node","mongodb"],   city: "Hyderabad" },
  { rollNo: "EEE2301", name: "Tarun", branch: "EEE", year: 1, cgpa: 7.5, skills: ["circuits"],         city: "Chennai" },
  { rollNo: "CSE2304", name: "Isha",  branch: "CSE", year: 1, cgpa: 9.4, skills: ["python","java"],    city: "Bengaluru" }
])
```

Output (IDs differ):
```
{
  acknowledged: true,
  insertedIds: { '0': ObjectId('...'), '1': ObjectId('...'), ... '7': ObjectId('...') }
}
```

> **Two facts about mongosh you must know:**
> • `use college` switches to the database `college` — it is **created lazily** only when you insert the first document.
> • mongosh is a real **JavaScript** shell: you can use variables, loops, `const`, functions. That's why the performance practical can generate 100,000 documents with a loop.

---

# 🧪 PRACTICAL 1 — INSTALL AND PREPARE THE MONGODB ENVIRONMENT

*(Syllabus item: "Install and prepare the MongoDB environment for use". Do this first — every other practical needs it.)*

## 1.1 Aim
To install MongoDB Community Edition (or use Docker / Atlas), start the server, connect using **mongosh** and **Compass**, and verify the setup.

## 1.2 Theory — what exactly are you installing?

```
┌────────────────────┐      wire protocol       ┌──────────────────────────┐
│ CLIENTS            │ ───────────────────────► │ mongod  (the DATABASE     │
│ • mongosh (shell)  │      TCP port 27017       │         SERVER process)   │
│ • Compass (GUI)    │ ◄─────────────────────── │  ┌────────────────────┐   │
│ • Node/Python app  │                           │  │ WiredTiger storage │   │
└────────────────────┘                           │  └────────────────────┘   │
                                                 │  data files on disk       │
                                                 └──────────────────────────┘
```

| Piece | What it is | Remember |
|---|---|---|
| **mongod** | The database **server** daemon. Stores data, executes queries. | Must be running first. Default port **27017**. |
| **mongosh** | The interactive **shell** (JavaScript/Node-based). A *client*. | Installed separately from the server on some platforms. |
| **Compass** | Official **GUI** client: browse, query, index, explain, aggregate visually. | Great for screenshots in your record. |
| **Atlas** | MongoDB's **managed cloud** service (free tier "M0" exists). | No local install; you connect with a `mongodb+srv://` URI. |
| **Drivers** | Libraries (Node, Python, Java…) your *application* uses. | Practicals 9 and 10. |
| **Community vs Enterprise** | Community = free source-available edition (your lab). Enterprise = paid extras (e.g., advanced auditing/auth integrations). | Know the one-line difference. |

**Three important locations (Linux package install):**
```
Config file : /etc/mongod.conf
Data files  : /var/lib/mongodb
Log file    : /var/log/mongodb/mongod.log
```
(Windows installer defaults are under `C:\Program Files\MongoDB\Server\<version>\` with the service named **MongoDB**; if you run `mongod` manually the default data path is `/data/db` on Linux/macOS — you must create it.)

## 1.3 Choose ONE install route

### Route A — Local install (what labs usually expect)

**Windows**
```
1. Download the MongoDB Community Server MSI from the official MongoDB download page.
2. Run the installer → "Complete" → tick "Install MongoDB as a Service" (Run as Network Service).
3. Tick "Install MongoDB Compass" (optional but recommended).
4. Install mongosh separately if the installer didn't include it.
5. Open Services → confirm "MongoDB Server" is Running.
```

**Ubuntu / Debian (outline — copy exact commands for your version from the official docs, they change per release)**
```bash
# 1. Import the MongoDB public GPG key
# 2. Add the MongoDB apt repository for your Ubuntu release
# 3. Install
sudo apt-get update
sudo apt-get install -y mongodb-org

# 4. Start and enable the service
sudo systemctl start mongod
sudo systemctl enable mongod
sudo systemctl status mongod       # should say: active (running)
```

**macOS (Homebrew)**
```bash
brew tap mongodb/brew
brew install mongodb-community
brew services start mongodb-community
```

### Route B — Docker (fastest, nothing installed on your OS)
```bash
docker run -d --name mongo-lab -p 27017:27017 -v mongo_lab_data:/data/db mongo
docker ps                        # confirm container is running
docker exec -it mongo-lab mongosh   # open a shell INSIDE the container
```
`-p 27017:27017` publishes the port; `-v` keeps your data when the container restarts.

### Route C — MongoDB Atlas (cloud)
```
1. Create an Atlas account → create a project → "Create" a free (M0) cluster.
2. Database Access  → Add a database USER (username + password). Note them.
3. Network Access   → Add your IP address (or 0.0.0.0/0 ONLY for a short classroom test — never for real data).
4. Connect → "Drivers" or "Shell" → copy the connection string:
      mongodb+srv://<user>:<password>@cluster0.xxxxx.mongodb.net/
5. Connect:
      mongosh "mongodb+srv://cluster0.xxxxx.mongodb.net/" --username <user>
```
> Special characters in a password (`@ : / ?` etc.) must be **URL-encoded** inside a URI (e.g., `@` → `%40`).

## 1.4 Verify the installation (the proof for your record)

```bash
mongosh                     # connects to mongodb://127.0.0.1:27017 by default
```
Then inside the shell:
```js
db.runCommand({ ping: 1 })       // → { ok: 1 }
db                                // → test   (default database)
show dbs                          // lists databases (admin, config, local at minimum)
db.version()                      // e.g. '8.0.x' — your version
use college                       // switch (database appears after first insert)
db.hello().isWritablePrimary      // true on a standalone / primary
```
✔ Screenshot: **the `ping` output + `db.version()`**.

## 1.5 Connect with Compass
```
1. Open Compass → New connection
2. URI:  mongodb://localhost:27017        (Atlas: paste the mongodb+srv URI)
3. Connect → left panel shows databases → select "college" → "students" → run a filter {branch:"CSE"}
4. Explore tabs: Documents · Aggregations · Schema · Explain Plan · Indexes · Validation
```

## 1.6 Understanding the connection string (viva favourite)

```
mongodb://  user:pass  @  host1:27017,host2:27017  /  mydb  ?  replicaSet=rs0&retryWrites=true
 └scheme┘  └credentials┘   └────── hosts ───────┘   └db┘      └──────── options ────────┘

mongodb+srv://user:pass@cluster0.abcde.mongodb.net/mydb
 └ "srv" = the hostname is looked up via DNS (SRV/TXT records) to discover all members + options; TLS is on by default
```

## 1.7 Common errors

| Error | Cause | Fix |
|---|---|---|
| `connect ECONNREFUSED 127.0.0.1:27017` | `mongod` not running | start the service / container |
| `MongoServerSelectionError` (Atlas) | IP not allow-listed, wrong URI, no internet | add IP in Network Access; recheck URI |
| `Authentication failed` | wrong user/password/auth database | re-check; URL-encode password |
| `mongosh: command not found` | mongosh not installed / not on PATH | install mongosh; reopen terminal |
| `Data directory /data/db not found` (manual run) | folder missing | `mkdir -p /data/db` or pass `--dbpath` |
| Port already in use | another `mongod` running | stop it or use `--port` |

## 1.8 RESULT
"MongoDB Community Server / Docker container / Atlas cluster was installed, started and verified using mongosh (`ping` → `ok: 1`) and Compass."

## 1.9 🎤 VIVA — Practical 1

**Q1. What is the difference between `mongod` and `mongosh`?**
> `mongod` is the **database server** — a long-running process that owns the data files, listens on a port (27017 by default) and executes reads and writes. `mongosh` is only a **client**: an interactive JavaScript shell that sends commands to a `mongod` over the network. You can stop `mongosh` at any time and the database keeps running; if `mongod` stops, no client can work. The same relationship holds for Compass, a Node app or a Python script — they are all clients.

**Q2. What is MongoDB Atlas, and when would you prefer it over a local install?**
> Atlas is MongoDB's **managed cloud database service** (DBaaS). MongoDB runs and operates the cluster for you: provisioning, replication, backups, monitoring, upgrades, scaling and security defaults such as TLS and IP access lists. I'd prefer it when I don't want operational overhead, when I need high availability out of the box, when a team must share a database, or for an app deployed on a cloud platform. I'd use a local install or Docker for offline learning, fast iteration, and when data must stay on-premises. The trade-off is cost and less low-level control.

**Q3. What does the default port 27017 mean and can it be changed?**
> It is the TCP port on which `mongod` accepts client connections. It can be changed with `--port` or `net.port` in `mongod.conf`. In production you should also bind to specific interfaces (`net.bindIp`) instead of all interfaces and restrict access using firewalls and authentication.

**Q4. Explain a `mongodb+srv://` connection string.**
> It is a DNS seed-list style URI. The part after `@` is not a single host but a name that is resolved through **DNS SRV and TXT records** to find the actual member hosts and some options (like the replica-set name). It enables TLS by default and means clients don't need to hard-code every node, so Atlas can change topology without breaking applications.

**Q5. After installation, how do you confirm the server is healthy?**
> Connect with `mongosh` and run `db.runCommand({ ping: 1 })` expecting `{ ok: 1 }`; check `db.version()`; check the service status (`systemctl status mongod` / Windows Services / `docker ps`); look at the log file for errors; and optionally run `db.serverStatus()` to see connections and uptime.

**Q6. Why must an Atlas IP access list entry exist before you can connect?**
> Atlas is secure by default: it accepts connections only from IP addresses you have explicitly allowed. Without an entry (and a database user), the connection is refused or times out with a server-selection error. Using `0.0.0.0/0` allows everyone and is acceptable only briefly for demos.

**🧑‍💼 Interview angle:** *"How would you set up MongoDB for a new project?"* → Mention: managed (Atlas) vs self-hosted, replica set from day one (even in staging), authentication on, least-privilege users, network restrictions, TLS, backups, monitoring — and that a standalone `mongod` without auth is for local learning only.

---

# 🧪 PRACTICAL 2 — CREATE A NoSQL DATABASE AND COMPARE WITH SQL

## 2.1 Aim
To create a MongoDB (document) database, store the same student information that a relational database would hold, and compare the two models.

## 2.2 Theory

### What is NoSQL?
**NoSQL ("Not Only SQL")** is a family of database systems that store data in models **other than fixed relational tables** and are designed for flexible schemas, horizontal scalability and specific access patterns.

```
KEY-VALUE        COLUMN-FAMILY          DOCUMENT              GRAPH
(Redis)          (Cassandra)            (MongoDB)             (Neo4j)
key → value      row key → column       JSON-like documents   nodes + edges
fast cache       groups, huge writes    nested, flexible      relationships first
```

### Why did NoSQL appear?
```
1. Data became huge (volume), arrived fast (velocity) and varied (variety).
2. Scaling a single relational server UP gets expensive; NoSQL systems were built to scale OUT across many cheap machines.
3. Object-oriented code and JSON APIs mismatch rows/tables ("impedance mismatch"); documents map naturally.
4. Many applications don't need every relational guarantee on every operation.
```

### CAP theorem (must know)
In a **distributed** system, during a **network partition** you must choose between **Consistency** (everyone sees the same latest data) and **Availability** (every request gets a response). Partition tolerance isn't optional in real networks, so the real choice is **C vs A**. MongoDB is typically described as **CP-leaning**: writes go to one primary; if a partition isolates a primary from the majority, it steps down and stops accepting writes, favouring consistency. Read/write concerns let you tune the trade-off.

## 2.3 Procedure — same data, two worlds

### 2.3.1 Relational (SQL) version
A student with several subject marks needs **two tables and a join**:

```sql
CREATE TABLE students (
  roll_no  VARCHAR(10) PRIMARY KEY,
  name     VARCHAR(50) NOT NULL,
  branch   VARCHAR(10),
  city     VARCHAR(30)
);

CREATE TABLE marks (
  id       INT AUTO_INCREMENT PRIMARY KEY,
  roll_no  VARCHAR(10),
  subject  VARCHAR(30),
  score    INT,
  FOREIGN KEY (roll_no) REFERENCES students(roll_no)
);

INSERT INTO students VALUES ('CSE2301','Aarav','CSE','Bengaluru');
INSERT INTO marks (roll_no, subject, score) VALUES
  ('CSE2301','DBMS',88), ('CSE2301','OS',79);

SELECT s.name, m.subject, m.score
FROM students s JOIN marks m ON s.roll_no = m.roll_no
WHERE s.roll_no = 'CSE2301';
```
(Run this in MySQL/PostgreSQL if your lab has one; PostgreSQL uses `SERIAL`/`GENERATED` instead of `AUTO_INCREMENT`.)

### 2.3.2 MongoDB version — one document, no join
```js
use college_demo

db.students.insertOne({
  rollNo: "CSE2301",
  name: "Aarav",
  branch: "CSE",
  city: "Bengaluru",
  marks: [
    { subject: "DBMS", score: 88 },
    { subject: "OS",   score: 79 }
  ]
})

db.students.findOne({ rollNo: "CSE2301" })
```
Output:
```js
{
  _id: ObjectId('...'),
  rollNo: 'CSE2301',
  name: 'Aarav',
  branch: 'CSE',
  city: 'Bengaluru',
  marks: [ { subject: 'DBMS', score: 88 }, { subject: 'OS', score: 79 } ]
}
```

### 2.3.3 Show schema flexibility (the "wow" step)
Another student has **extra, different fields** — no `ALTER TABLE` needed:
```js
db.students.insertOne({
  rollNo: "ECE2301", name: "Kabir", branch: "ECE",
  hobbies: ["chess", "drones"],              // a field the first doc doesn't have
  guardian: { name: "R. Singh", phone: "98xxxxxx01" }   // nested object
})

db.students.find({}, { _id: 0, rollNo: 1, hobbies: 1, guardian: 1 })
```
In SQL you'd add nullable columns or new tables; here documents in one collection can differ.

### 2.3.4 Command-by-command equivalence

| Task | SQL | MongoDB |
|---|---|---|
| Create database | `CREATE DATABASE college;` | `use college` (created on first insert) |
| Create table / collection | `CREATE TABLE students (...)` | `db.createCollection("students")` (or implicit on insert) |
| Insert | `INSERT INTO students ... VALUES (...)` | `db.students.insertOne({...})` |
| Select all | `SELECT * FROM students;` | `db.students.find()` |
| Filter | `SELECT * FROM students WHERE branch='CSE' AND cgpa>8;` | `db.students.find({branch:"CSE", cgpa:{$gt:8}})` |
| Columns | `SELECT name, cgpa FROM students;` | `db.students.find({}, {name:1, cgpa:1, _id:0})` |
| Sort / limit | `ORDER BY cgpa DESC LIMIT 3` | `.sort({cgpa:-1}).limit(3)` |
| Update | `UPDATE students SET cgpa=9 WHERE roll_no='X';` | `db.students.updateOne({rollNo:"X"},{$set:{cgpa:9}})` |
| Delete | `DELETE FROM students WHERE roll_no='X';` | `db.students.deleteOne({rollNo:"X"})` |
| Group | `SELECT branch, COUNT(*) FROM students GROUP BY branch;` | `db.students.aggregate([{$group:{_id:"$branch", n:{$sum:1}}}])` |
| Join | `JOIN` | `$lookup` (or embed) |
| Index | `CREATE INDEX ...` | `db.students.createIndex({...})` |
| Primary key | `PRIMARY KEY` | `_id` (automatic, unique) |

## 2.4 The comparison table (write this in your record)

| Aspect | SQL (relational) | MongoDB (document NoSQL) |
|---|---|---|
| Data model | Tables of rows and columns | Collections of BSON documents |
| Schema | Fixed; defined **before** data | Flexible; optional validation |
| Relationships | Foreign keys + joins | Embedding, or references + `$lookup` |
| Normalisation | Normalise to remove duplication | Model for **access patterns**; controlled duplication |
| Query language | SQL (standard) | MQL + aggregation pipeline (JSON-like) |
| Scaling | Mostly vertical; sharding is complex | Designed for horizontal scale (sharding) + replica sets |
| Transactions | Mature multi-row ACID | Single-document atomic always; multi-document transactions available (replica set/sharded cluster) |
| Best fit | Structured, highly relational, strict integrity, complex joins | Evolving schemas, nested/hierarchical data, high volume, read-together access |
| Examples | MySQL, PostgreSQL, Oracle | MongoDB, (others: Redis, Cassandra, Neo4j in other NoSQL families) |

> ⚠️ Never conclude "NoSQL is better". Conclude: **"each is better suited to different workloads; many real systems use both (polyglot persistence)."**

## 2.5 RESULT
A MongoDB database and collection were created, the same data was modelled as a single document versus two relational tables, and the two models were compared.

## 2.6 🎤 VIVA — Practical 2

**Q1. What is NoSQL? Does it mean "no SQL"?**
> No — it is commonly read as **"Not Only SQL"**. It describes databases that do not rely exclusively on the relational table model. They offer other data models (key-value, column-family, document, graph), usually with flexible schemas and horizontal scaling. It does not mean they are query-less; MongoDB has a rich query language and even an aggregation framework comparable in power to SQL's GROUP BY/JOIN features.

**Q2. Name the four NoSQL data models with one example and one use-case each.**
> **Key-value** — Redis; caching, sessions. **Column-family (wide-column)** — Cassandra; massive write-heavy time-ordered data. **Document** — MongoDB; catalogues, profiles, content, general-purpose apps. **Graph** — Neo4j; social networks, recommendations, fraud rings where relationships are the query.

**Q3. Explain the CAP theorem and where MongoDB sits.**
> CAP says a distributed data store cannot simultaneously guarantee Consistency, Availability and Partition tolerance. Since network partitions are unavoidable, the practical trade-off during a partition is between consistency and availability. MongoDB uses a single-primary replica set: if the primary loses contact with the majority, it steps down and an election happens; the minority side can't accept writes. So it leans to **consistency (CP)** by default, and you can tune behaviour with write concern, read concern and read preference — for example, reading from secondaries accepts potentially stale data for availability/latency.

**Q4. Why can MongoDB store the student with marks in one document, and what is the benefit?**
> Because documents can contain arrays and nested objects, related data that is *read together* can live together. The benefit is a **single read** with no join, better data locality on disk, and atomic updates to the whole document. The cost is possible duplication and the 16 MB document limit, so it suits bounded, tightly related data.

**Q5. What are the disadvantages of NoSQL compared with SQL?**
> Less standardised querying; fewer built-in relational guarantees (foreign keys aren't enforced by the database); joins are less natural and can be costly at scale; multi-document transactions exist but have performance costs; schema discipline moves to the application unless validation is used; and consistency models can be more complex to reason about in distributed deployments.

**Q6. Is MongoDB ACID-compliant?**
> Single-document operations are **atomic** by design (all-or-nothing, even across nested fields and arrays). Since MongoDB 4.0 (replica sets) and 4.2 (sharded clusters) it also supports **multi-document ACID transactions**. They should be used sparingly because they add overhead; good schema design often makes them unnecessary.

**🧑‍💼 Interview angle:** *"When would you NOT use MongoDB?"* → heavy multi-table relational integrity with complex joins as the core need (classic ERP/double-entry ledgers), or when the team's strength and tooling is entirely SQL and the data is naturally tabular. Show balance, not loyalty.

---

# 🧪 PRACTICAL 3 — PERFORM CRUD OPERATIONS ON DOCUMENTS

## 3.1 Aim
To perform **C**reate, **R**ead, **U**pdate and **D**elete operations on MongoDB documents using mongosh.

## 3.2 Theory
```
CRUD        MongoDB methods
─────────   ───────────────────────────────────────────────────────
Create      insertOne · insertMany
Read        find · findOne · countDocuments · distinct
Update      updateOne · updateMany · replaceOne · findOneAndUpdate (+ upsert option)
Delete      deleteOne · deleteMany · findOneAndDelete
```

Key ideas:
- A **filter** `{ field: value }` selects documents; **operators** like `$gt`, `$in`, `$or` extend it.
- An **update document** must use **update operators** (`$set`, `$inc`…) — otherwise `updateOne` throws an error.
- Results are **objects** (`acknowledged`, `matchedCount`, `modifiedCount`, `insertedId`, `deletedCount`) — not just "OK".

(Run Section 0.3 first so `college.students` has the 8 documents.)

## 3.3 CREATE

```js
use college

db.students.insertOne({ rollNo: "CSE2305", name: "Zoya", branch: "CSE", year: 2, cgpa: 8.1, skills: ["go"], city: "Delhi" })
```
Output:
```js
{ acknowledged: true, insertedId: ObjectId('...') }
```

```js
db.students.insertMany([
  { rollNo: "ME2302", name: "Vikram", branch: "ME", year: 2, cgpa: 7.0, skills: ["cad","ansys"], city: "Pune" },
  { rollNo: "ECE2303", name: "Nisha",  branch: "ECE", year: 1, cgpa: 8.3, skills: ["c"],           city: "Delhi" }
])
```
*`insertMany` is **ordered** by default: if one document fails (e.g. duplicate `_id`), the rest after it are not inserted. Pass `{ ordered: false }` to continue past errors.*

## 3.4 READ

```js
db.students.find()                                   // all documents
db.students.find().pretty()                          // (mongosh already pretty-prints)
db.students.findOne({ rollNo: "CSE2301" })           // first match (or null)
```

### Filters
```js
db.students.find({ branch: "CSE" })
db.students.find({ branch: "CSE", year: 3 })                         // implicit AND
db.students.find({ cgpa: { $gte: 8 } })                              // comparison
db.students.find({ branch: { $in: ["CSE", "ECE"] } })                // membership
db.students.find({ $or: [ { branch: "ME" }, { cgpa: { $gt: 9 } } ] })// logical OR
db.students.find({ skills: "python" })                               // array contains
db.students.find({ skills: { $all: ["python", "ml"] } })             // array has ALL
db.students.find({ name: /^S/ })                                      // regex: starts with S
db.students.find({ guardian: { $exists: false } })                   // field missing
```

### Projection, sort, skip, limit
```js
db.students.find({ branch: "CSE" }, { _id: 0, name: 1, cgpa: 1 })
          .sort({ cgpa: -1 })
          .skip(0)
          .limit(3)
```
With the 8 original students + the three new ones, the CSE sorted-by-cgpa-desc output begins:
```js
{ name: 'Isha',  cgpa: 9.4 }
{ name: 'Diya',  cgpa: 9.1 }
{ name: 'Sana',  cgpa: 8.8 }
```
> Logical order is always **filter → sort → skip → limit**, regardless of the order you type the methods.

### Counting and distinct
```js
db.students.countDocuments({ branch: "CSE" })        // accurate count for a filter
db.students.estimatedDocumentCount()                 // fast, whole-collection estimate
db.students.distinct("branch")                       // unique values, e.g. [ 'CSE', 'ECE', 'EEE', 'ME' ] (order not guaranteed)
```

### Array-of-objects matching with `$elemMatch`
```js
db.marksdemo.insertOne({ roll: 1, marks: [ { sub: "DBMS", s: 88 }, { sub: "OS", s: 45 } ] })
// Which student has DBMS score > 80?
db.marksdemo.find({ marks: { $elemMatch: { sub: "DBMS", s: { $gt: 80 } } } })
```
Without `$elemMatch`, conditions could match **different** array elements; `$elemMatch` forces both on the **same** element.

## 3.5 UPDATE

```js
// set a field
db.students.updateOne({ rollNo: "CSE2301" }, { $set: { cgpa: 8.6 } })
```
Output:
```js
{ acknowledged: true, insertedId: null, matchedCount: 1, modifiedCount: 1, upsertedCount: 0 }
```

```js
db.students.updateMany({ branch: "CSE" }, { $set: { dept: "Computer Science" } })   // many
db.students.updateOne({ rollNo: "CSE2301" }, { $inc: { year: 1 } })                  // promote to next year ($inc adds)
db.students.updateOne({ rollNo: "CSE2302" }, { $inc: { cgpa: 0.1 } })                // increment
db.students.updateOne({ rollNo: "CSE2302" }, { $unset: { dept: "" } })               // remove a field
db.students.updateOne({ rollNo: "CSE2302" }, { $rename: { city: "hometown" } })      // rename
```

### Array updates
```js
db.students.updateOne({ rollNo: "CSE2301" }, { $push: { skills: "mongodb" } })       // always append
db.students.updateOne({ rollNo: "CSE2301" }, { $addToSet: { skills: "mongodb" } })   // append only if absent → no change now
db.students.updateOne({ rollNo: "CSE2301" }, { $pull: { skills: "sql" } })           // remove matching value
db.students.updateOne({ rollNo: "CSE2301" }, { $pop: { skills: 1 } })                // remove last (-1 = first)
```
Run the `$addToSet` twice and notice the second result has `modifiedCount: 0` — **matched but nothing changed** (not an error!).

### Upsert
```js
db.students.updateOne(
  { rollNo: "CSE9999" },
  { $set: { name: "New Student", branch: "CSE" } },
  { upsert: true }
)
```
Output (no such roll existed):
```js
{ acknowledged: true, insertedId: ObjectId('...'), matchedCount: 0, modifiedCount: 0, upsertedCount: 1 }
```

### Replace the whole document
```js
db.students.replaceOne({ rollNo: "CSE9999" }, { rollNo: "CSE9999", name: "Replaced", branch: "CSE" })
```
`_id` is kept; **all other old fields vanish**. A replacement document must **not** contain operators.

### Get the document back after updating
```js
db.students.findOneAndUpdate(
  { rollNo: "CSE2303" },
  { $inc: { cgpa: 0.1 } },
  { returnDocument: "after" }          // default is "before"
)
```

### What happens if you forget `$set`?
```js
db.students.updateOne({ rollNo: "CSE2301" }, { cgpa: 9 })
// MongoServerError: Update document requires atomic operators
```

## 3.6 DELETE

```js
db.students.deleteOne({ rollNo: "CSE9999" })            // { acknowledged: true, deletedCount: 1 }
db.students.deleteMany({ year: 4 })                      // deletes all matching
db.students.findOneAndDelete({ rollNo: "ME2302" })       // returns the deleted doc
```
> **Safety habit:** run `find(filter)` **first**, check the result, *then* run `deleteMany(filter)`. `deleteMany({})` removes **all** documents (but keeps the collection and its indexes); `drop()` removes the collection itself.

## 3.7 Dry run — how `updateOne` works inside
```
db.students.updateOne({ rollNo: "CSE2301" }, { $set: { cgpa: 8.6 } })

1. Client → server: update command with filter + update + options
2. Server finds the FIRST document matching the filter (uses an index on rollNo if one exists)
3. Applies $set → cgpa becomes 8.6 (the write is atomic on that one document)
4. Writes to the oplog / journal (durability)
5. Returns counters: matchedCount=1 (found), modifiedCount=1 (actually changed)
```
If cgpa was **already** 8.6: `matchedCount: 1, modifiedCount: 0`.

## 3.8 Common errors
| Mistake | Symptom | Fix |
|---|---|---|
| Update without operator | "Update document requires atomic operators" | wrap in `$set` |
| Filter typo (`brnch`) | `matchedCount: 0` | check field names |
| Number as string (`"8"` vs `8`) | query returns nothing | match the stored type |
| Duplicate `_id` | `E11000 duplicate key error` | let MongoDB generate `_id` |
| `deleteMany({})` by accident | whole collection emptied | test the filter with `find` first |

## 3.9 RESULT
All CRUD operations were executed successfully on the `students` collection and the result objects were verified.

## 3.10 🎤 VIVA — Practical 3

**Q1. Difference between `insertOne` and `insertMany`?**
> `insertOne` inserts a single document; `insertMany` takes an array and inserts them in one round trip, which is much more efficient than many `insertOne` calls. `insertMany` is ordered by default, so on the first error it stops; with `{ordered:false}` it attempts all documents and reports the errors afterwards. Both return `acknowledged` and the generated `_id`s.

**Q2. What happens if you insert a document without `_id`?**
> MongoDB automatically adds `_id` with a newly generated **ObjectId**. `_id` is mandatory, unique within a collection and always indexed. You may supply your own (a string, number, UUID…) as long as it is unique and immutable.

**Q3. Explain what an ObjectId contains.**
> A 12-byte value: a 4-byte **timestamp** (seconds since epoch), a 5-byte **random** value unique per process/machine, and a 3-byte incrementing **counter**. So ObjectIds are roughly time-ordered, and `ObjectId.getTimestamp()` returns the creation time. They aren't a security feature — don't treat them as secrets.

**Q4. Difference between `updateOne`, `updateMany` and `replaceOne`?**
> `updateOne` modifies the first matching document; `updateMany` modifies all matches; both need update operators and change only the specified fields. `replaceOne` swaps the **entire** document (except `_id`) with the new one, so omitted fields disappear. Use replace only when you really intend a full overwrite.

**Q5. `matchedCount` vs `modifiedCount`?**
> `matchedCount` is how many documents satisfied the filter; `modifiedCount` is how many were **actually changed**. If the new value equals the old one, you get `matchedCount: 1, modifiedCount: 0`. So `modifiedCount: 0` does **not** mean "not found".

**Q6. What is an upsert and when is it useful?**
> *Up*date + in*sert*: if no document matches the filter, MongoDB inserts a new document built from the filter's equality conditions plus the update. It's useful for idempotent "create-or-update" operations such as counters, caches, or syncing external data — but it needs a **unique index** on the filter key to be safe under concurrency, otherwise two simultaneous upserts may both insert.

**Q7. `$push` vs `$addToSet`?**
> `$push` always appends (duplicates allowed); `$addToSet` appends only if the value is not already in the array, giving set-like behaviour. For unique tag/skill lists use `$addToSet`.

**Q8. `deleteMany({})` vs `drop()`?**
> `deleteMany({})` deletes every document but **keeps** the collection, its indexes and validation rules; it logs each delete and is slower on huge collections. `drop()` removes the whole collection including its indexes and is much faster. To empty a giant collection quickly in production teams often drop and recreate (re-creating indexes and validators).

**Q9. Why is `countDocuments()` preferred over `count()`?**
> `count()` is deprecated in drivers/shell; `countDocuments(filter)` is accurate (it uses an aggregation `$match` + `$group`), while `estimatedDocumentCount()` uses collection metadata and is fast but takes no filter and is an estimate in some situations (like unclean shutdown or sharded clusters with orphans).

**Q10. How do you make a "read-modify-write" safe against concurrent users?**
> Use **atomic update operators** (`$inc`, `$set`, `$addToSet`) in one `updateOne` or `findOneAndUpdate` rather than reading a value, changing it in application code and writing it back. Put the condition in the **filter** (e.g., `{ seats: { $gt: 0 } }`), so only one of two simultaneous requests can match — this is how the CourseHub seat booking avoids overbooking.

**🧑‍💼 Interview angle:** *"How do you implement a safe counter / inventory decrement?"* → `findOneAndUpdate({ sku, stock: { $gte: qty } }, { $inc: { stock: -qty } }, { returnDocument: "after" })`; a `null` return means insufficient stock. No transaction needed for a single document.

---

# 🧪 PRACTICAL 4 — USE AND EXPLORE BSON DATA TYPES

## 4.1 Aim
To insert documents using various BSON data types, inspect their types, and understand why type choice matters.

## 4.2 Theory

**JSON** is text with only a few types (string, number, boolean, null, array, object). **BSON** (*Binary JSON*) is MongoDB's **binary** storage and wire format: faster to traverse, and it adds types JSON lacks.

| BSON type | `$type` alias | mongosh example | Notes |
|---|---|---|---|
| String | `"string"` | `"Aarav"` | UTF-8 |
| Int32 | `"int"` | `NumberInt(25)` | 32-bit integer |
| Int64 (Long) | `"long"` | `NumberLong("9007199254740993")` | use a string to avoid JS precision loss |
| Double | `"double"` | `8.4` | 64-bit floating point |
| Decimal128 | `"decimal"` | `NumberDecimal("19.99")` | exact decimals — money |
| Boolean | `"bool"` | `true` | |
| Date | `"date"` | `new Date()` / `ISODate("2026-11-22T10:00:00Z")` | stored as UTC milliseconds |
| ObjectId | `"objectId"` | `ObjectId()` | 12-byte id |
| Array | `"array"` | `["a","b"]` | |
| Embedded document | `"object"` | `{ city: "Pune" }` | |
| Null | `"null"` | `null` | |
| Regular expression | `"regex"` | `/^abc/i` | |
| Timestamp | `"timestamp"` | `Timestamp(...)` | internal (oplog) use |
| Binary data | `"binData"` | `UUID("...")` / `BinData(...)` | UUIDs, files bytes |

Others you should name: **MinKey/MaxKey** (compare lowest/highest), **JavaScript code** (discouraged), **Symbol** (deprecated).

## 4.3 Procedure

### Step 1 — insert one document with many types
```js
use bsonlab
db.types.drop()

db.types.insertOne({
  name: "Aarav",                              // String
  age: NumberInt(21),                         // Int32
  views: NumberLong("9007199254740993"),      // Int64
  cgpa: 8.4,                                  // Double
  fee: NumberDecimal("45999.50"),             // Decimal128
  isActive: true,                             // Boolean
  joinedAt: ISODate("2026-07-15T09:30:00Z"),  // Date
  skills: ["python", "sql"],                  // Array
  address: { city: "Bengaluru", pin: "560001" }, // Embedded document
  nickname: null,                             // Null
  pattern: /^CSE/i,                           // Regex
  token: UUID("123e4567-e89b-12d3-a456-426614174000") // Binary subtype UUID
})
```

### Step 2 — ask MongoDB what type each field has
```js
db.types.aggregate([
  { $project: {
      _id: 0,
      name:     { $type: "$name" },
      age:      { $type: "$age" },
      views:    { $type: "$views" },
      cgpa:     { $type: "$cgpa" },
      fee:      { $type: "$fee" },
      isActive: { $type: "$isActive" },
      joinedAt: { $type: "$joinedAt" },
      skills:   { $type: "$skills" },
      address:  { $type: "$address" },
      nickname: { $type: "$nickname" },
      pattern:  { $type: "$pattern" },
      token:    { $type: "$token" }
  } }
])
```
Output:
```js
{
  name: 'string', age: 'int', views: 'long', cgpa: 'double', fee: 'decimal',
  isActive: 'bool', joinedAt: 'date', skills: 'array', address: 'object',
  nickname: 'null', pattern: 'regex', token: 'binData'
}
```
✔ This single output is the best "proof" screenshot for the practical.

### Step 3 — query by type
```js
db.types.find({ age: { $type: "int" } })           // matches
db.types.find({ age: { $type: "string" } })        // no match
db.types.find({ cgpa: { $type: ["int","double"] } })// any of these types
db.types.find({ $or: [ {fee:{$type:"decimal"}}, {fee:{$type:"double"}} ] })
```

### Step 4 — dates
```js
db.types.insertOne({ label: "now", at: new Date() })
db.types.find({ at: { $gte: ISODate("2026-01-01T00:00:00Z") } })

// Pull parts of a date with aggregation operators
db.types.aggregate([
  { $match: { label: "now" } },
  { $project: { _id: 0, year: { $year: "$at" }, month: { $month: "$at" }, iso: { $dateToString: { format: "%Y-%m-%d", date: "$at" } } } }
])
```
Dates are stored in **UTC**; formatting in a time zone is done on output (e.g., `$dateToString` has a `timezone` option).

### Step 5 — ObjectId carries a timestamp
```js
const id = db.types.findOne()._id
id.getTimestamp()                 // ISODate('...') — when this doc was created
```

### Step 6 — floating-point vs Decimal128 (the money trap) 💰
```js
0.1 + 0.2                          // 0.30000000000000004  (JavaScript double arithmetic)

db.money.drop()
db.money.insertOne({ a: NumberDecimal("0.1"), b: NumberDecimal("0.2") })
db.money.aggregate([ { $project: { _id: 0, sum: { $add: ["$a", "$b"] } } } ])
// { sum: Decimal128('0.3') }   ← exact
```
Rule: **never store money as double**; use Decimal128 (or integer paise/cents).

### Step 7 — type-sensitive queries (the silent bug) 🐞
```js
db.t2.drop()
db.t2.insertMany([ { v: 10 }, { v: "10" }, { v: 10.0 } ])

db.t2.find({ v: 10 })      // matches numeric 10 and 10.0 (numbers compare by value) — NOT the string "10"
db.t2.find({ v: "10" })    // matches only the string
```
Numbers of different numeric types compare by value; a **string "10" is a different type** and won't match. This is why consistent types matter (and why validation helps).

### Step 8 — document size
```js
Object.bsonsize(db.types.findOne())      // size in bytes of that document
```
The maximum size of a single BSON document is **16 MB**. For larger files use **GridFS**.

## 4.4 What is the "plain number" in mongosh?
A plain integer like `25` is typically stored as **int** (when it fits in 32 bits) and `8.4` as **double** in current mongosh, whereas older shells stored all numbers as double. **Check on your setup with `$type`** (Step 2) rather than assuming, and use `NumberInt`/`NumberLong`/`NumberDecimal` explicitly when the type matters.

## 4.5 Common errors / traps
```
❌ Storing dates as strings ("22-11-2026") → can't range-query or sort correctly; use Date.
❌ Storing money as double → rounding errors; use Decimal128.
❌ Mixed types in the same field across documents → queries miss data.
❌ NumberLong(9007199254740993) with a JS number → precision lost; pass a STRING.
❌ Using ObjectId as string in a filter (e.g., { _id: "65f..." }) → no match; wrap with ObjectId("...").
```

## 4.6 RESULT
Documents using String, Int32, Int64, Double, Decimal128, Boolean, Date, Array, Embedded document, Null, Regex and Binary (UUID) types were inserted and verified with the `$type` operator.

## 4.7 🎤 VIVA — Practical 4

**Q1. What is BSON and why does MongoDB use it instead of JSON?**
> BSON is **Binary JSON**: a compact, binary-encoded serialisation of JSON-like documents. MongoDB uses it because (1) it supports **extra data types** — Date, ObjectId, Int32/Int64, Decimal128, Binary — that plain JSON doesn't; (2) length-prefixed encoding makes it fast to **traverse and skip** fields without parsing text; and (3) it's efficient for encoding/decoding in drivers. The trade-off is that BSON isn't always smaller than JSON (it stores field names and length prefixes), but it is optimised for traversal speed and type richness.

**Q2. List important BSON types and say which you'd use for money, timestamps and identifiers.**
> Money → **Decimal128** (or integer minor units like paise). Timestamps → **Date** (UTC milliseconds). Identifiers → **ObjectId** by default, or UUID (binary subtype) when IDs must be generated outside the database or be non-guessable; a natural unique key may also be indexed separately.

**Q3. Why is storing 0.1 + 0.2 as double a problem?**
> Binary floating point cannot represent many decimal fractions exactly, so 0.1 + 0.2 gives 0.30000000000000004. For currency this accumulates errors and breaks reconciliation. Decimal128 stores exact decimal digits (up to 34 significant) and is the right choice for financial amounts.

**Q4. What is the maximum document size and what do you do for bigger data?**
> **16 MB**. For large binary files use **GridFS**, which splits files into chunks (255 KB default) across two collections (`fs.files`, `fs.chunks`). For large structured data, redesign: reference or split into multiple documents (bucket pattern) instead of growing one document.

**Q5. How do you find which documents have a field of a particular type?**
> Use the `$type` query operator, e.g. `{ age: { $type: "int" } }`, or an array of types. In aggregation, `{ $type: "$field" }` returns the type name per document — handy for auditing messy data before adding validation.

**Q6. Are dates stored with time zones?**
> No. A BSON Date is a 64-bit integer of **UTC milliseconds since the Unix epoch**. Time-zone handling is a presentation concern: convert on output (`$dateToString` with `timezone`) or in the application. Always store UTC.

**Q7. Why might `{ _id: "65f2..." }` not find a document whose `_id` is an ObjectId?**
> Because the stored `_id` is an **ObjectId**, while the filter supplies a **string**; MongoDB compares by type as well as value, so they don't match. In mongosh use `ObjectId("65f2...")`; in Node `new ObjectId(str)`; in Python `ObjectId(str)` from `bson`.

**Q8. What's the difference between `NumberInt`, `NumberLong` and a plain number?**
> `NumberInt` forces a 32-bit integer; `NumberLong` forces a 64-bit integer; a plain JavaScript number is serialised by the shell/driver as int (if integral and small) or double. Choosing explicitly avoids surprises like integer overflow in arithmetic or unintended floating-point storage and matters for indexes, sorting and aggregation arithmetic.

**🧑‍💼 Interview angle:** *"What data-type mistakes have you seen in production?"* → dates as strings, money as doubles, IDs stored as both string and ObjectId in different collections (breaking `$lookup`), and mixed numeric/string values in one field. The fix: **schema validation + a typed access layer + migrations**.

---

# 🧪 PRACTICAL 5 — MODEL ONE-TO-ONE / ONE-TO-MANY / MANY-TO-MANY RELATIONSHIPS

## 5.1 Aim
To model all three relationship types in MongoDB using **embedding** and **referencing**, and to query them (including `$lookup`).

## 5.2 Theory — the only question that matters

```
Q: "When I read A, do I almost always need B as well?"

YES, B is small and bounded         → EMBED   (one document, one read)
NO / B is big / unbounded / shared  → REFERENCE (store B's _id, fetch B separately)
```

| | **Embed** | **Reference** |
|---|---|---|
| Read | One query | Two queries, or `$lookup`/`populate` |
| Write | Atomic on the single document | Atomic per document; multi-doc needs a transaction |
| Duplication | Possible (the same data in many parents) | Minimal (single source of truth) |
| Size risk | Document can grow toward 16 MB | None |
| Best when | Read together, small, bounded, rarely changes independently | Large, unbounded, shared by many, changes independently |

**Cardinality ladder (MongoDB-style):**
```
one-to-ONE        profile ↔ student            → embed (usually)
one-to-FEW        student → 2–3 phone numbers   → embed an array
one-to-MANY       department → hundreds of students → reference (child holds parent _id)
one-to-SQUILLIONS device → millions of log events    → reference to the parent, NEVER an array in the parent
many-to-MANY      students ↔ courses            → junction collection (if the link has data) or arrays of references
```

## 5.3 Procedure

```js
use modeling
db.dropDatabase()
```

### 5.3.1 One-to-One and one-to-few → EMBED
```js
db.students.insertOne({
  rollNo: "CSE2301",
  name: "Aarav",
  address: { line1: "12 MG Road", city: "Bengaluru", pin: "560001" },   // 1-1 embedded object
  phones: ["9800000001", "9800000002"]                                   // 1-few embedded array
})
db.students.findOne({ rollNo: "CSE2301" }, { _id: 0, name: 1, "address.city": 1, phones: 1 })
```
```js
{ name: 'Aarav', address: { city: 'Bengaluru' }, phones: [ '9800000001', '9800000002' ] }
```
*Why embed?* An address is always shown with the student, is small, and has no life of its own. One read gets everything; updating the address is one atomic write.

### 5.3.2 One-to-Many → REFERENCE (child stores the parent's `_id`)
```js
const cse = db.departments.insertOne({ code: "CSE", name: "Computer Science", hod: "Dr. Rao"  }).insertedId
const ece = db.departments.insertOne({ code: "ECE", name: "Electronics",      hod: "Dr. Iyer" }).insertedId

db.students.drop()
db.students.insertMany([
  { rollNo: "CSE2301", name: "Aarav", departmentId: cse },
  { rollNo: "CSE2302", name: "Diya",  departmentId: cse },
  { rollNo: "ECE2301", name: "Kabir", departmentId: ece }
])

// all students of a department  (simple reference query)
db.students.find({ departmentId: cse }, { _id: 0, name: 1 })
```
```js
{ name: 'Aarav' }
{ name: 'Diya' }
```

Join it back with `$lookup` (a left outer join):
```js
db.departments.aggregate([
  { $lookup: { from: "students", localField: "_id", foreignField: "departmentId", as: "students" } },
  { $project: { _id: 0, code: 1, studentCount: { $size: "$students" }, names: "$students.name" } }
])
```
```js
{ code: 'CSE', studentCount: 2, names: [ 'Aarav', 'Diya' ] }
{ code: 'ECE', studentCount: 1, names: [ 'Kabir' ] }
```
> **Index the foreign key** (`students.departmentId`) — otherwise every `$lookup` scans the students collection.
```js
db.students.createIndex({ departmentId: 1 })
```

### 5.3.3 Many-to-Many → JUNCTION COLLECTION (the relationship has its own data)
A student takes many courses; a course has many students; each link has **marks** → give the link its own collection.

```js
const c1 = db.courses.insertOne({ code: "CSE494", title: "NoSQL Databases",  credits: 4 }).insertedId
const c2 = db.courses.insertOne({ code: "CSE310", title: "Operating Systems", credits: 3 }).insertedId
const s1 = db.students.findOne({ rollNo: "CSE2301" })._id
const s2 = db.students.findOne({ rollNo: "CSE2302" })._id

db.enrollments.insertMany([
  { student: s1, course: c1, marks: 88 },
  { student: s1, course: c2, marks: 79 },
  { student: s2, course: c1, marks: 92 }
])

// guarantee no duplicate enrollment at the database level
db.enrollments.createIndex({ student: 1, course: 1 }, { unique: true })
```
Try a duplicate:
```js
db.enrollments.insertOne({ student: s1, course: c1, marks: 50 })
// MongoServerError: E11000 duplicate key error collection: modeling.enrollments index: student_1_course_1 dup key: {...}
```

**Query 1 — Aarav's courses with titles**
```js
db.enrollments.aggregate([
  { $match: { student: s1 } },
  { $lookup: { from: "courses", localField: "course", foreignField: "_id", as: "c" } },
  { $unwind: "$c" },
  { $project: { _id: 0, code: "$c.code", title: "$c.title", marks: 1 } }
])
```
```js
{ code: 'CSE494', title: 'NoSQL Databases',   marks: 88 }
{ code: 'CSE310', title: 'Operating Systems', marks: 79 }
```
*(Field order within each result may differ; the content is what matters.)*

**Query 2 — students in CSE494**
```js
db.enrollments.aggregate([
  { $match: { course: c1 } },
  { $lookup: { from: "students", localField: "student", foreignField: "_id", as: "s" } },
  { $unwind: "$s" },
  { $project: { _id: 0, name: "$s.name", marks: 1 } }
])
```
```js
{ name: 'Aarav', marks: 88 }
{ name: 'Diya',  marks: 92 }
```

### 5.3.4 Many-to-Many without data → arrays of references (lighter option)
When the link carries **no data** and each side stays small (e.g., a book with 2–3 authors):
```js
const a1 = db.authors.insertOne({ name: "Kyle Banker" }).insertedId
const a2 = db.authors.insertOne({ name: "Peter Bakkum" }).insertedId
db.books.insertOne({ title: "MongoDB in Action", authorIds: [a1, a2] })

db.books.aggregate([
  { $lookup: { from: "authors", localField: "authorIds", foreignField: "_id", as: "authors" } },
  { $project: { _id: 0, title: 1, authors: "$authors.name" } }
])
// { title: 'MongoDB in Action', authors: [ 'Kyle Banker', 'Peter Bakkum' ] }
```
(`$lookup` with an array `localField` matches each element.)

### 5.3.5 Handy patterns (name them in viva)

| Pattern | Idea | Example |
|---|---|---|
| **Subset** | Embed only the most-used few items; keep the rest referenced | latest 5 reviews inside the product; all reviews in `reviews` |
| **Extended reference** | Copy a few stable fields of the referenced doc to avoid a lookup | store `studentName` inside enrollment for list pages |
| **Bucket** | Group many small items (time-series) into one document per hour/day | sensor readings per hour in an array |
| **Computed** | Store a pre-calculated value | `enrolledCount`, `total`, `grade` |
| **Outlier** | Handle the rare huge document separately | a celebrity with millions of followers uses an overflow collection |
| **Schema versioning** | `schemaVersion` field to migrate gradually | `{ schemaVersion: 2, ... }` |

### 5.3.6 Anti-patterns
```
❌ Unbounded arrays (comments inside a post forever)  → 16 MB limit, slow updates
❌ Embedding data that changes independently & is shared (e.g., department name copied into 10,000 students)
❌ Referencing everything (SQL-style normalisation) → lookups everywhere, losing MongoDB's benefit
❌ References with no index on the foreign key
❌ Storing the same relationship in BOTH directions without a reason
```

## 5.4 RESULT
One-to-one and one-to-few were modelled by embedding, one-to-many by referencing, and many-to-many with a junction collection plus array references; all were queried using `find` and `$lookup`.

## 5.5 🎤 VIVA — Practical 5

**Q1. Embedding vs referencing — how do you decide?**
> I decide from the **access pattern**, the **size/growth** of the related data, and how it **changes**. If the data is read together, is small and bounded, and doesn't change independently, I embed it for one-query reads and single-document atomicity. If it is large, unbounded, shared by many parents, or updated independently, I reference it and index the reference. In CourseHub I embedded `address` and `skills` but referenced `enrollments`, because enrollments carry their own data (marks, status), grow without a hard bound and are queried from both the student and the course side.

**Q2. How do you model many-to-many in MongoDB?**
> Two options. If the relationship has **its own attributes** (marks, dates, status) or can become large, use a **junction collection** with two references and a unique compound index to prevent duplicates. If it has no attributes and both sides stay small, store an **array of ObjectIds** on one side (or both, with care to keep them in sync). I prefer the junction collection for anything that could grow.

**Q3. What does `$lookup` do and what are its limitations?**
> `$lookup` performs a **left outer join** to another collection in the same database, adding the matched documents as an array field. Limitations: it is more expensive than reading one embedded document; the foreign field must be indexed for good performance; and on sharded clusters the joined collection has restrictions (it can't always be sharded in the same way depending on version). If I `$lookup` on every request, that's a signal to reconsider the model (embed or use the extended reference pattern).

**Q4. Is a foreign key enforced in MongoDB?**
> No. References are just values; MongoDB does not check that the referenced document exists and has no cascading delete. Referential integrity is enforced by the **application** (e.g., refuse to delete a department that still has students) or with transactions in critical paths, and can be audited with `$lookup` queries for orphans.

**Q5. What is the unbounded array anti-pattern and how do you avoid it?**
> It's an array that grows without limit — like all comments inside a post. It risks exceeding the 16 MB document limit, makes the document grow and move/rewrite often, and slows updates and index maintenance on array fields. Avoid it by referencing (a `comments` collection with `postId`), bucketing, or using the subset pattern.

**Q6. Why a unique compound index on `{ student, course }`?**
> It makes duplicate enrollments **impossible at the database level**, even if two requests arrive simultaneously, because the check is atomic inside the index. It also speeds "all enrollments of a student" through the index **prefix** `student`, so one index does three jobs.

**Q7. Where does normalisation fit in MongoDB?**
> MongoDB schemas are designed **query-first**, not normal-form-first. Some duplication is accepted where it improves reads (e.g., storing a student's name inside an enrollment) provided you know how it will be kept consistent. Over-normalising turns MongoDB into a slower relational database; over-embedding creates huge, hard-to-update documents. Good design balances both.

**Q8. What is the extended-reference pattern?**
> Store a reference **plus** a handful of frequently needed, stable fields from the referenced document (e.g., `{ studentId, studentName }` inside an enrollment) so list screens don't need a `$lookup`. The cost is that if the copied field changes you must update the copies, so copy only fields that rarely change.

**🧑‍💼 Interview angle:** *"Design the schema for an e-commerce order system."* → Orders **embed** line items (bounded, always read together, price snapshot at purchase time), **reference** customer and product by id (with an extended reference for product name/price at order time), keep a `status` history array (bounded), index `{customerId:1, createdAt:-1}`, and discuss when you'd shard (shard key choice). Stating the *why* for each choice is what earns the offer.

---

# 🧪 PRACTICAL 6 — APPLY SCHEMA VALIDATION AND CONSTRAINTS

## 6.1 Aim
To enforce structure and rules on a collection using `$jsonSchema` validators and indexes (unique), and to test valid and invalid inserts.

## 6.2 Theory

MongoDB is **flexible by default**, but you can **add rules** when you need them.

```
CONSTRAINT                     HOW IN MONGODB
───────────────────────────    ─────────────────────────────────────────────
Required field                 $jsonSchema → required: [...]
Data type                      $jsonSchema → bsonType
Range (min/max)                minimum / maximum
Allowed values (enum)          enum: [...]
String format                  pattern (regular expression), minLength, maxLength
Array rules                    items, minItems, uniqueItems
Unique value                   UNIQUE INDEX  (not a validator!)
Cross-field rule               $expr in the validator
Default value                  NOT in the database → application / Mongoose default
Foreign key                    NOT enforced → application / transaction logic
```

Two switches you must know:

| Setting | Values | Meaning |
|---|---|---|
| `validationLevel` | `strict` (default) / `moderate` / `off` | which writes are checked. `moderate` checks inserts and updates of documents that are **already valid**; existing invalid documents can still be updated |
| `validationAction` | `error` (default) / `warn` | `error` **rejects** the invalid write; `warn` **allows** it but writes a warning to the log |

## 6.3 Procedure

### Step 1 — create a validated collection
```js
use validationlab
db.dropDatabase()

db.createCollection("students", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      title: "Student validation",
      required: ["rollNo", "name", "branch", "year", "cgpa", "email"],
      properties: {
        rollNo: { bsonType: "string", pattern: "^[A-Z]{2,4}[0-9]{4}$",
                  description: "e.g. CSE2301" },
        name:   { bsonType: "string", minLength: 2, maxLength: 50 },
        branch: { enum: ["CSE", "ECE", "EEE", "ME", "CE"],
                  description: "must be one of the listed branches" },
        year:   { bsonType: "number", minimum: 1, maximum: 4 },
        cgpa:   { bsonType: "number", minimum: 0, maximum: 10 },
        email:  { bsonType: "string", pattern: "^.+@.+\\..+$" },
        skills: { bsonType: "array", uniqueItems: true, items: { bsonType: "string" } }
      }
    }
  },
  validationLevel: "strict",
  validationAction: "error"
})
// { ok: 1 }
```

> **Note on `bsonType: "number"`:** `"number"` accepts int, long, double and decimal. If you wrote `bsonType: "int"` you would **reject** a plain `2` typed in a shell that stores it as a double. Choose types deliberately (and test!).

### Step 2 — add the **unique** constraints (these are indexes)
```js
db.students.createIndex({ rollNo: 1 }, { unique: true })
db.students.createIndex({ email: 1 },  { unique: true })
```

### Step 3 — a VALID insert
```js
db.students.insertOne({
  rollNo: "CSE2301", name: "Aarav", branch: "CSE", year: 2, cgpa: 8.4,
  email: "aarav@college.edu", skills: ["python", "sql"]
})
// { acknowledged: true, insertedId: ObjectId('...') }
```

### Step 4 — INVALID inserts (each must fail)
```js
// (a) missing required field "email"
db.students.insertOne({ rollNo: "CSE2302", name: "Diya", branch: "CSE", year: 3, cgpa: 9.1 })

// (b) cgpa out of range
db.students.insertOne({ rollNo: "CSE2303", name: "Sana", branch: "CSE", year: 3, cgpa: 11, email: "sana@college.edu" })

// (c) branch not in enum
db.students.insertOne({ rollNo: "XYZ2301", name: "Rohan", branch: "XYZ", year: 1, cgpa: 7, email: "r@college.edu" })

// (d) wrong type: cgpa as a string
db.students.insertOne({ rollNo: "CSE2304", name: "Isha", branch: "CSE", year: 1, cgpa: "9.4", email: "isha@college.edu" })

// (e) bad pattern: rollNo
db.students.insertOne({ rollNo: "23-CSE", name: "Tarun", branch: "EEE", year: 1, cgpa: 7.5, email: "t@college.edu" })

// (f) duplicate unique key
db.students.insertOne({ rollNo: "CSE2301", name: "Dup", branch: "CSE", year: 1, cgpa: 8, email: "dup@college.edu" })
```
Output for (a)–(e):
```
MongoServerError: Document failed validation
  code: 121,
  errInfo: { failingDocumentId: ObjectId('...'), details: { operatorName: '$jsonSchema', ... } }
```
Output for (f) — note it is a **different** error, from the index, not the validator:
```
MongoServerError: E11000 duplicate key error collection: validationlab.students index: rollNo_1 dup key: { rollNo: "CSE2301" }
```
(newer versions print which rule failed inside `errInfo.details`; the exact layout depends on your version.)

### Step 5 — validation also applies to **updates**
```js
db.students.updateOne({ rollNo: "CSE2301" }, { $set: { cgpa: 15 } })
// MongoServerError: Document failed validation
```

### Step 6 — look at / change the rules on a live collection
```js
db.getCollectionInfos({ name: "students" })[0].options      // shows validator, level, action

db.runCommand({
  collMod: "students",
  validationLevel: "moderate",       // don't break updates of OLD invalid docs
  validationAction: "warn"           // log instead of reject while migrating
})
```
Typical migration approach for an existing messy collection:
```
1. Audit: find bad docs with queries ({ cgpa: { $not: { $type: "number" } } }, missing fields…)
2. Set validationAction: "warn", level "moderate"  → nothing breaks, you see violations in the log
3. Fix the data (scripts / updates)
4. Switch to "error" + "strict"
```

### Step 7 — unique + optional field (partial index)
A plain unique index treats a **missing** field as `null`, so two documents without `phone` collide. Use a partial index:
```js
db.students.createIndex(
  { phone: 1 },
  { unique: true, partialFilterExpression: { phone: { $exists: true } } }
)
```

### Step 8 — cross-field rule with `$expr`
```js
db.createCollection("exams", {
  validator: { $expr: { $lte: ["$obtained", "$maxMarks"] } }   // obtained must not exceed max
})
db.exams.insertOne({ obtained: 95, maxMarks: 100 })   // ok
db.exams.insertOne({ obtained: 120, maxMarks: 100 })  // fails validation
```

### Step 9 — the same ideas in Mongoose (application level, preview of Practical 9)
```js
const studentSchema = new mongoose.Schema({
  rollNo: { type: String, required: true, unique: true, match: /^[A-Z]{2,4}[0-9]{4}$/ },
  cgpa:   { type: Number, min: 0, max: 10, required: true },
  branch: { type: String, enum: ["CSE","ECE","EEE","ME","CE"] }
});
```
`unique: true` creates a unique **index** (not a validator) → duplicates raise error **11000**.

## 6.4 Where should validation live? (important design answer)

```
APPLICATION (Mongoose / Joi / Zod)        DATABASE ($jsonSchema, unique indexes)
✔ friendly error messages                 ✔ protects against EVERY client (shell, Python, other services)
✔ complex business logic, defaults         ✔ last line of defence — bugs can't bypass it
✘ bypassed by other clients/scripts        ✘ limited expressiveness, harder messages
                    ───────────►  USE BOTH  ◄───────────
```

## 6.5 RESULT
A validated collection was created with required fields, type, range, enum, pattern and unique constraints; valid inserts succeeded and invalid inserts were rejected with validation (code 121) or duplicate-key (11000) errors.

## 6.6 🎤 VIVA — Practical 6

**Q1. MongoDB is "schema-less" — then why validation?**
> "Schema-less" really means **flexible-schema**: the database doesn't force every document to share a structure. But real applications need guarantees such as required fields, correct types and ranges. `$jsonSchema` validation lets me add those rules **per collection** only where needed, getting flexibility for evolving data and strictness for critical data.

**Q2. Explain `validationLevel` and `validationAction`.**
> `validationLevel` controls **which** writes are validated: `strict` validates all inserts and updates; `moderate` validates inserts and updates to documents that already satisfy the rules, leaving existing invalid documents alone; `off` disables it. `validationAction` controls **what happens** on failure: `error` rejects the write, `warn` accepts it but logs a warning. A common rollout is `moderate` + `warn` first, clean the data, then `strict` + `error`.

**Q3. Is `unique` a validation rule?**
> No. Uniqueness is enforced by a **unique index**, not by `$jsonSchema`. The failure is a duplicate-key error (code 11000), not a validation error (code 121). It's enforced atomically by the index, which is why it is safe under concurrency, unlike an application-level "check then insert".

**Q4. How do you enforce a default value or a foreign key?**
> Neither is a database feature in MongoDB. Defaults are applied by the application layer (e.g., Mongoose `default`) or in the insert code; foreign keys are not enforced — the application ensures references are valid, uses transactions when several documents must change together, and can run periodic orphan checks.

**Q5. A unique index on an optional field fails for the second document without the field. Why and how to fix?**
> Missing fields are indexed as `null`, so the second document without it duplicates `null`. Use a **partial unique index** with `partialFilterExpression: { field: { $exists: true } }` so only documents that actually have the field are indexed.

**Q6. How would you add validation to a collection that already has bad data?**
> Audit first with queries to find violations, then `collMod` with `validationLevel: "moderate"` and `validationAction: "warn"` so nothing breaks while I fix the data with scripts, then tighten to `strict`/`error`. I'd also add the rule to the application and tests, record the change in a migration and take a backup before bulk fixes.

**Q7. Why did a document with `year: 2` fail when I used `bsonType: "int"`?**
> Because depending on how the value was serialised (shell/driver version), the plain `2` may have been stored as **double**, and `"int"` accepts only 32-bit integers. Use `"number"` (accepts any numeric type) or insert `NumberInt(2)`, and always test the validator with real client data.

**Q8. Application-level vs database-level validation — which one?**
> Both. The application gives user-friendly messages and complex logic; the database protects the data from **every** client and from bugs. In CourseHub I used Mongoose validators plus unique indexes, and recommended `$jsonSchema` on critical collections for production.

**🧑‍💼 Interview angle:** *"How do you evolve a schema without downtime?"* → additive changes first, a `schemaVersion` field, readers that tolerate both versions, a background migration, validation switched from warn → error after the data is clean, and the old code path removed last.

---

# 🧪 PRACTICAL 7 — CREATE INDEXES AND ANALYSE PERFORMANCE

## 7.1 Aim
To measure query performance **before and after** creating indexes on a collection of 100,000 documents using `explain("executionStats")`, and to understand single, compound, covered, text, unique and TTL indexes.

## 7.2 Theory

### What is an index?
A special data structure (a **B-tree**) that stores the values of chosen fields **in sorted order**, each with a pointer to the document. Like the index at the back of a textbook: instead of reading every page, you look up the term and jump to the page.

```
WITHOUT INDEX                                 WITH INDEX on { rollNo: 1 }
COLLSCAN: read all 100,000 docs               B-tree:      ┌──[R050000]──┐
→ check rollNo of each                                 ┌───┘             └───┐
→ O(n)                                              ...                     ...
                                                   lookup → pointer → 1 document  → O(log n)
```

**Cost of indexes:** extra disk and RAM; every insert/update/delete must update every relevant index (slower writes). **Index what your queries need — no more.**

### Index types you must know
| Type | Command | Use |
|---|---|---|
| Single-field | `createIndex({ rollNo: 1 })` | equality / range / sort on one field |
| Compound | `createIndex({ branch: 1, cgpa: -1 })` | queries on several fields; **order matters** |
| Unique | `{ unique: true }` | no duplicates |
| Text | `createIndex({ title: "text" })` | word search with `$text` (one per collection) |
| TTL | `{ expireAfterSeconds: n }` on a Date field | auto-delete expired docs |
| Partial | `{ partialFilterExpression: {...} }` | index only some documents (smaller) |
| Sparse | `{ sparse: true }` | skip docs missing the field |
| Multikey | automatic on array fields | index each array element |
| Geospatial / Hashed / Wildcard / Vector | specialised | location queries, sharding, unknown fields, similarity search |

### Rules
```
PREFIX RULE   index {a,b,c} supports queries on  a · a+b · a+b+c   — NOT on b alone, NOT on c alone.
ESR RULE      order compound fields:  Equality → Sort → Range.
COVERED QUERY every filter + projected field is inside the index (and _id excluded) → no document fetch.
```

### Reading `explain("executionStats")`
| Field | Meaning | Healthy |
|---|---|---|
| `winningPlan.stage` (find the leaf) | `COLLSCAN` (full scan) vs `IXSCAN` (index scan) → then `FETCH` | `IXSCAN` |
| `nReturned` | documents returned | — |
| `totalKeysExamined` | index entries examined | close to `nReturned` |
| `totalDocsExamined` | documents read | close to `nReturned` (0 if covered) |
| `executionTimeMillis` | time | low |
| `SORT` stage in plan | sort done **in memory** (no index for the sort) | avoid on large sets |

> **Version note:** in newer MongoDB versions the plan may be nested (e.g., `winningPlan.queryPlan.stage`) because of the slot-based execution engine. Search the printed plan for `COLLSCAN` / `IXSCAN` / `SORT` — the idea is identical.

## 7.3 Procedure

### Step 1 — generate 100,000 documents
```js
use perflab
db.perf.drop()

const branches = ["CSE", "ECE", "EEE", "ME", "CE"];
let batch = [];
for (let i = 1; i <= 100000; i++) {
  batch.push({
    rollNo: "R" + String(i).padStart(6, "0"),
    name: "Student" + i,
    branch: branches[i % 5],
    year: (i % 4) + 1,
    cgpa: Number((6 + Math.random() * 4).toFixed(2)),
    createdAt: new Date()
  });
  if (batch.length === 5000) { db.perf.insertMany(batch); batch = []; }
}
db.perf.countDocuments()        // 100000
db.perf.getIndexes()            // only the automatic _id_ index
```
(Inserting in batches of 5,000 is far faster than 100,000 separate `insertOne` calls.)

### Step 2 — BEFORE: equality query without an index
```js
db.perf.find({ rollNo: "R050000" }).explain("executionStats")
```
Look at (abridged):
```js
executionStats: {
  nReturned: 1,
  totalKeysExamined: 0,
  totalDocsExamined: 100000,      // ← read EVERY document to find ONE
  executionTimeMillis: 40         // example value; yours will differ
},
winningPlan: { stage: 'COLLSCAN', ... }
```
**Efficiency ratio:** examined 100,000 → returned 1. Terrible.

### Step 3 — AFTER: create a single-field index
```js
db.perf.createIndex({ rollNo: 1 })       // 'rollNo_1'
db.perf.find({ rollNo: "R050000" }).explain("executionStats")
```
```js
executionStats: { nReturned: 1, totalKeysExamined: 1, totalDocsExamined: 1, executionTimeMillis: 0 },
winningPlan: { stage: 'FETCH', inputStage: { stage: 'IXSCAN', indexName: 'rollNo_1', ... } }
```
100,000 → **1**. Record both screenshots in your lab record.

### Step 4 — compound index and the ESR rule
Query: *CSE students of year 3 with cgpa above 9, sorted by name.*
```js
const q = { branch: "CSE", year: 3, cgpa: { $gt: 9 } };

db.perf.find(q).sort({ name: 1 }).explain("executionStats")
//  COLLSCAN + SORT stage, ~100000 docs examined, nReturned ≈ 1,250 (random data → approximate)
```
Apply ESR: **E**quality `branch`, `year` → **S**ort `name` → **R**ange `cgpa`.
```js
db.perf.createIndex({ branch: 1, year: 1, name: 1, cgpa: 1 })
db.perf.find(q).sort({ name: 1 }).explain("executionStats")
//  IXSCAN on branch_1_year_1_name_1_cgpa_1, NO separate SORT stage
//  totalKeysExamined ≈ 5,000 (all CSE year-3 keys), nReturned ≈ 1,250
```
> Why put `cgpa` **after** `name`? If the range field came before the sort field, MongoDB couldn't read the index in sorted order and would need an in-memory sort. (Trade-off: ESR is a strong guideline, not a law — always confirm with `explain`.)

### Step 5 — the prefix rule in action
```js
db.perf.find({ branch: "CSE" }).explain("executionStats")                 // can use the index (prefix: branch)
db.perf.find({ branch: "CSE", year: 3 }).explain("executionStats")        // can use the index (prefix: branch+year)
db.perf.find({ year: 3 }).explain("executionStats")                       // year alone is NOT a prefix → no efficient lookup (expect COLLSCAN or a full index scan; compare examined counts)
```

### Step 6 — covered query (index-only)
```js
db.perf.createIndex({ branch: 1, cgpa: -1 })

db.perf.find({ branch: "CSE" }, { _id: 0, branch: 1, cgpa: 1 })
       .sort({ cgpa: -1 })
       .limit(5)
       .explain("executionStats")
```
```js
totalDocsExamined: 0         // ← never touched a document
// plan shows PROJECTION_COVERED over IXSCAN
```
If you forget `_id: 0`, `_id` isn't in the index, so MongoDB must fetch the document → **not covered**.

### Step 7 — text index
```js
db.articles.drop()
db.articles.insertMany([
  { title: "MongoDB indexing basics",  body: "Indexes speed up reads in MongoDB." },
  { title: "Cooking pasta",            body: "Boil water, add salt." },
  { title: "Aggregation pipeline",     body: "Use stages like match and group in MongoDB." }
])
db.articles.createIndex({ title: "text", body: "text" })

db.articles.find(
  { $text: { $search: "mongodb index" } },
  { _id: 0, title: 1, score: { $meta: "textScore" } }
).sort({ score: { $meta: "textScore" } })
```
Returns the two MongoDB articles ranked by relevance (the pasta document is not returned). Text search stems words (index/indexes/indexing) and ignores common stop-words; **only one text index per collection** is allowed.

### Step 8 — TTL index (auto-expiry)
```js
db.sessions.drop()
db.sessions.createIndex({ createdAt: 1 }, { expireAfterSeconds: 60 })
db.sessions.insertOne({ user: "aarav", createdAt: new Date() })
// ~1-2 minutes later:  db.sessions.countDocuments()  → 0
```
A background task removes expired documents about **every 60 seconds**, so deletion isn't instant. The indexed field must be a **Date**.

### Step 9 — list, inspect, hint, drop
```js
db.perf.getIndexes()                                   // all indexes
db.perf.aggregate([ { $indexStats: {} } ])             // usage counts per index (find unused ones)
db.perf.totalIndexSize()                               // bytes used by indexes
db.perf.find({ rollNo: "R000010" }).hint({ rollNo: 1 })// force a specific index (for experiments)
db.perf.dropIndex("rollNo_1")                          // remove an index by name
```

### Step 10 — measure the **write cost** of indexes
```js
db.cost.drop()
let t0 = Date.now()
for (let i = 0; i < 20000; i++) db.cost.insertOne({ a: i, b: i % 7, c: "x" + i })
print("no extra indexes:", Date.now() - t0, "ms")

db.cost.drop()
db.cost.createIndex({ a: 1 }); db.cost.createIndex({ b: 1 }); db.cost.createIndex({ c: 1 })
t0 = Date.now()
for (let i = 0; i < 20000; i++) db.cost.insertOne({ a: i, b: i % 7, c: "x" + i })
print("3 extra indexes:", Date.now() - t0, "ms")
```
You should see the second run is **slower** — the visible proof that indexes cost write performance. (Exact times depend on your machine.)

### Step 11 — pagination and indexes
```js
db.perf.find().sort({ _id: 1 }).skip(90000).limit(10)         // skip must walk 90,000 entries → slow for deep pages
const last = db.perf.find().sort({ _id: 1 }).skip(49).limit(1).toArray()[0]._id
db.perf.find({ _id: { $gt: last } }).sort({ _id: 1 }).limit(10) // keyset pagination → constant-time
```

## 7.4 Summary table for your record

| Query | Index | Before | After |
|---|---|---|---|
| `rollNo = X` | `{rollNo:1}` | COLLSCAN, 100,000 docs examined | IXSCAN, 1 doc examined |
| `branch,year,cgpa>9` sort `name` | `{branch,year,name,cgpa}` | COLLSCAN + in-memory SORT | IXSCAN, no SORT |
| projection-only report | `{branch:1,cgpa:-1}` | FETCH needed | **0 docs examined** (covered) |

## 7.5 Common mistakes
```
❌ Indexing every field "just in case"        → slow writes, wasted RAM
❌ Wrong compound-index order (range before sort) → in-memory sort
❌ Forgetting _id:0 and wondering why it isn't covered
❌ Using regex /abc/ (unanchored) and expecting index use → anchored /^abc/ can use it
❌ Judging by time alone — use docsExamined vs nReturned (time varies with cache/machine)
❌ Not indexing the foreign key used by $lookup
```

## 7.6 RESULT
Indexes were created and `explain("executionStats")` showed the change from `COLLSCAN` (100,000 documents examined) to `IXSCAN` (1 document examined); compound, covered, text and TTL indexes were demonstrated.

## 7.7 🎤 VIVA — Practical 7

**Q1. What is an index and how does it work?**
> An index is a separate, ordered B-tree structure holding the values of selected fields plus pointers to documents. When a query's filter or sort matches an index, MongoDB walks the tree in logarithmic time instead of scanning the whole collection, and reads results in index order — which can also satisfy `sort` without extra work. Every collection has an automatic unique index on `_id`.

**Q2. What are the disadvantages of too many indexes?**
> Each write must update every index on the collection, so inserts, updates and deletes get slower; indexes consume disk and, more importantly, **RAM** (the working set of hot index pages should fit in memory); and they add maintenance and build time. So I create indexes for real, frequent queries and drop unused ones, identified with `$indexStats`.

**Q3. Explain the ESR rule with an example.**
> For compound indexes, put **Equality** fields first, then the **Sort** field, then **Range** fields. For `find({branch:"CSE", year:3, cgpa:{$gt:9}}).sort({name:1})` the index `{branch:1, year:1, name:1, cgpa:1}` lets MongoDB jump to the exact `branch+year` region, read names already in sorted order, and filter `cgpa` within the index — no in-memory sort and few documents examined.

**Q4. What is the prefix rule?**
> A compound index on `{a, b, c}` can serve queries that use a **leading subset** of its fields: `a`, `a+b`, or `a+b+c`. A query on `b` alone or `c` alone cannot use it efficiently, because the index is sorted first by `a`.

**Q5. What is a covered query and why is it fast?**
> A query is covered when every field in the filter **and** in the projection is part of one index (and `_id` is excluded unless indexed). MongoDB can answer purely from the index, so `totalDocsExamined` is 0, avoiding disk reads of documents. Use it for hot, narrow queries such as dashboards listing two fields.

**Q6. How do you read `explain("executionStats")` to decide if a query is efficient?**
> I look at the **stage** (`COLLSCAN` is a warning for big collections; `IXSCAN` is good), then compare `totalDocsExamined` and `totalKeysExamined` with `nReturned` — a large gap means wasted work — then check for a blocking in-memory `SORT`, and finally `executionTimeMillis`, which is the least reliable because of caching. If it's inefficient I adjust the index, rerun explain, and verify the improvement.

**Q7. Difference between `queryPlanner`, `executionStats` and `allPlansExecution` modes?**
> `queryPlanner` shows the chosen (winning) plan **without running** the query; `executionStats` runs the winning plan and reports real counters; `allPlansExecution` also reports stats for the rejected candidate plans the optimiser tried — useful for understanding why MongoDB chose an index.

**Q8. When would you use a TTL index, partial index, text index?**
> TTL for data that should expire automatically — sessions, OTPs, temporary logs. Partial when only a subset of documents is queried (e.g., `status: "active"`), giving a smaller, cheaper index. Text for natural-language word search over string fields with relevance scoring; for advanced search at scale use a dedicated search engine or Atlas Search.

**Q9. Why is `skip(90000)` slow and what is the alternative?**
> `skip` has to walk past all skipped entries each time, so deep pages cost O(skip). Use **keyset (range) pagination**: remember the last seen sort key (e.g., `_id`) and query `{ _id: { $gt: lastId } }` with `limit` — constant time with an index.

**Q10. Does creating an index block the database?**
> Modern MongoDB builds indexes with a process that holds exclusive locks only briefly at the start and end, so reads and writes can continue during the build (older versions had foreground/background modes). Still, on large collections building indexes uses significant I/O and CPU, so do it off-peak, monitor, and for replica sets prefer rolling builds. Never build blindly on a busy primary.

**🧑‍💼 Interview angle:** *"A query that used to be fast is now slow. Walk me through your debugging."* → Check recent deploys/data growth; get the actual query shape (profiler/slow log); run `explain("executionStats")`; look for COLLSCAN, high examined/returned ratio, in-memory sort; check `$indexStats` and whether the planner picked a different index; fix with the right compound index (ESR) or query rewrite; validate with explain and load test; add alerting on slow queries. Close by saying you'd verify the index's **write overhead** and RAM footprint.

---

# 🧪 PRACTICAL 8 — BUILD AGGREGATION PIPELINES USING STAGES

## 8.1 Aim
To analyse data with the **aggregation framework** using the stages `$match`, `$group`, `$project`, `$sort`, `$limit`, `$unwind`, `$lookup`, `$addFields`, `$bucket` and `$count`.

## 8.2 Theory — the assembly line 🏭

```
 collection ─► [ $match ] ─► [ $group ] ─► [ $sort ] ─► [ $project ] ─► result
              filter rows    summarise    order          reshape
```

An **aggregation pipeline** is an **ordered array of stages**; documents flow through the stages, and each stage's output is the next stage's input.

| Stage | Role | SQL cousin |
|---|---|---|
| `$match` | filter documents | `WHERE` / `HAVING` |
| `$group` | group by key and compute `$sum`, `$avg`, `$min`, `$max`, `$push`, `$addToSet`, `$first`, `$last` | `GROUP BY` |
| `$project` | include/exclude/compute fields | `SELECT` list |
| `$addFields` / `$set` | add computed fields, keep the rest | computed columns |
| `$sort` | order | `ORDER BY` |
| `$limit` / `$skip` | restrict | `LIMIT` / `OFFSET` |
| `$unwind` | one output document per array element | lateral join / unnest |
| `$lookup` | left outer join to another collection | `LEFT JOIN` |
| `$bucket` | group numeric values into ranges | `CASE WHEN` + group |
| `$count` / `$sortByCount` | count / group-count-sort | `COUNT(*)` |
| `$facet` | several sub-pipelines on the same input | multiple queries at once |
| `$out` / `$merge` | **write** the result to a collection | `INSERT … SELECT` / upsert |

**Optimisation rules:** `$match` (and `$limit`) as early as possible; `$match` first can use an index; `$project` early to shrink documents; avoid unnecessary `$unwind`/`$lookup`; index the foreign field of `$lookup`; remember blocking stages (`$group`, `$sort` without an index) use memory and may spill to disk.

## 8.3 Procedure — data setup (uses the 8 students of Section 0.3 plus a tiny enrollment set)

```js
use college
// FIRST re-run Section 0.3 (it drops and re-creates the 8 original students) so your outputs match the ones below. Then:

db.courses.drop();  db.enrollments.drop()
db.courses.insertMany([
  { code: "CSE494", title: "NoSQL Databases" },
  { code: "CSE310", title: "Operating Systems" }
])
db.enrollments.insertMany([
  { rollNo: "CSE2301", course: "CSE494", marks: 88 },
  { rollNo: "CSE2301", course: "CSE310", marks: 79 },
  { rollNo: "CSE2302", course: "CSE494", marks: 92 },
  { rollNo: "ECE2301", course: "CSE494", marks: 65 }
])
```

### A1 — students per branch (`$group` + `$sort`)
```js
db.students.aggregate([
  { $group: { _id: "$branch", total: { $sum: 1 } } },
  { $sort: { total: -1, _id: 1 } }
])
```
```js
{ _id: 'CSE', total: 4 }
{ _id: 'ECE', total: 2 }
{ _id: 'EEE', total: 1 }
{ _id: 'ME',  total: 1 }
```
(`_id: 1` is a tie-breaker so equal totals have a fixed order.)

### A2 — average, maximum per branch, rounded
```js
db.students.aggregate([
  { $group: { _id: "$branch", avgCgpa: { $avg: "$cgpa" }, maxCgpa: { $max: "$cgpa" }, n: { $sum: 1 } } },
  { $project: { _id: 0, branch: "$_id", n: 1, maxCgpa: 1, avgCgpa: { $round: ["$avgCgpa", 1] } } },
  { $sort: { avgCgpa: -1 } }
])
```
```js
{ n: 4, maxCgpa: 9.4, branch: 'CSE', avgCgpa: 8.9 }
{ n: 2, maxCgpa: 8,   branch: 'ECE', avgCgpa: 7.6 }
{ n: 1, maxCgpa: 7.5, branch: 'EEE', avgCgpa: 7.5 }
{ n: 1, maxCgpa: 6.9, branch: 'ME',  avgCgpa: 6.9 }
```
**Dry run of what flows through the pipeline:**
```
Input (8 docs) → $group builds 4 buckets:
   CSE → cgpas [8.4, 9.1, 8.8, 9.4]  → avg 8.925, max 9.4, n 4
   ECE → [7.2, 8.0]                  → avg 7.6,   max 8.0, n 2
   EEE → [7.5]                       → 7.5
   ME  → [6.9]                       → 6.9
→ $project renames _id→branch and rounds avg to 1 decimal (CSE 8.925 → 8.9)
→ $sort by avgCgpa descending
```

### A3 — top 3 students (`$sort` + `$limit` + `$project`)
```js
db.students.aggregate([
  { $sort: { cgpa: -1 } },
  { $limit: 3 },
  { $project: { _id: 0, name: 1, branch: 1, cgpa: 1 } }
])
```
```js
{ name: 'Isha', branch: 'CSE', cgpa: 9.4 }
{ name: 'Diya', branch: 'CSE', cgpa: 9.1 }
{ name: 'Sana', branch: 'CSE', cgpa: 8.8 }
```

### A4 — filter **before** grouping (`$match` first)
*Year-3 students: count and top CGPA per branch.*
```js
db.students.aggregate([
  { $match: { year: 3 } },
  { $group: { _id: "$branch", n: { $sum: 1 }, top: { $max: "$cgpa" } } },
  { $sort: { _id: 1 } }
])
```
```js
{ _id: 'CSE', n: 2, top: 9.1 }
{ _id: 'ECE', n: 1, top: 8 }
```

### A5 — most common skills (`$unwind`)
```js
db.students.aggregate([
  { $unwind: "$skills" },
  { $group: { _id: "$skills", n: { $sum: 1 } } },
  { $sort: { n: -1, _id: 1 } },
  { $limit: 3 }
])
```
```js
{ _id: 'python', n: 4 }
{ _id: 'c',      n: 1 }
{ _id: 'cad',    n: 1 }
```
`$unwind` turns a student with 2 skills into 2 documents (8 students → 14 skill-documents). A student with an **empty or missing** array disappears unless you use `{ $unwind: { path: "$skills", preserveNullAndEmptyArrays: true } }`.

### A6 — computed field with conditions (`$addFields` + `$switch`) then group
```js
db.students.aggregate([
  { $addFields: { grade: {
      $switch: { branches: [
        { case: { $gte: ["$cgpa", 9] }, then: "O" },
        { case: { $gte: ["$cgpa", 8] }, then: "A" },
        { case: { $gte: ["$cgpa", 7] }, then: "B" } ],
        default: "C" } } } },
  { $group: { _id: "$grade", students: { $push: "$name" }, n: { $sum: 1 } } },
  { $sort: { _id: 1 } }
])
```
```js
{ _id: 'A', students: [ 'Aarav', 'Meera', 'Sana' ], n: 3 }
{ _id: 'B', students: [ 'Kabir', 'Tarun' ],         n: 2 }
{ _id: 'C', students: [ 'Rohan' ],                   n: 1 }
{ _id: 'O', students: [ 'Diya', 'Isha' ],            n: 2 }
```
(The order of names inside each array follows the order documents arrived.)

### A7 — `$lookup` join + group: average marks per course with title
```js
db.enrollments.aggregate([
  { $group: { _id: "$course", avgMarks: { $avg: "$marks" }, takers: { $sum: 1 } } },
  { $lookup: { from: "courses", localField: "_id", foreignField: "code", as: "c" } },
  { $unwind: "$c" },
  { $project: { _id: 0, course: "$_id", title: "$c.title", takers: 1, avgMarks: { $round: ["$avgMarks", 1] } } },
  { $sort: { course: 1 } }
])
```
```js
{ takers: 1, course: 'CSE310', title: 'Operating Systems', avgMarks: 79 }
{ takers: 3, course: 'CSE494', title: 'NoSQL Databases',   avgMarks: 81.7 }
```
(CSE494 average = (88 + 92 + 65) / 3 = 81.666… → 81.7.)

### A8 — join students with enrollments (who scored ≥ 80?)
```js
db.enrollments.aggregate([
  { $match: { marks: { $gte: 80 } } },
  { $lookup: { from: "students", localField: "rollNo", foreignField: "rollNo", as: "s" } },
  { $unwind: "$s" },
  { $project: { _id: 0, name: "$s.name", course: 1, marks: 1 } },
  { $sort: { marks: -1 } }
])
```
```js
{ course: 'CSE494', marks: 92, name: 'Diya' }
{ course: 'CSE494', marks: 88, name: 'Aarav' }
```

### A9 — `$bucket`: CGPA ranges
```js
db.students.aggregate([
  { $bucket: { groupBy: "$cgpa", boundaries: [6, 7, 8, 9, 10], default: "other", output: { n: { $sum: 1 } } } }
])
```
```js
{ _id: 6, n: 1 }     // 6.0 ≤ cgpa < 7  → Rohan 6.9
{ _id: 7, n: 2 }     // 7 ≤ cgpa < 8    → Kabir 7.2, Tarun 7.5
{ _id: 8, n: 3 }     // 8 ≤ cgpa < 9    → Aarav 8.4, Meera 8.0, Sana 8.8
{ _id: 9, n: 2 }     // 9 ≤ cgpa < 10   → Diya 9.1, Isha 9.4
```
(Each bucket's lower bound is inclusive, upper bound exclusive.)

### A10 — `$count`, `$sortByCount`, `$facet`
```js
db.students.aggregate([ { $match: { cgpa: { $gte: 8 } } }, { $count: "highPerformers" } ])
// { highPerformers: 5 }

db.students.aggregate([ { $sortByCount: "$city" } ])      // city frequency, descending (ties unordered)

db.students.aggregate([
  { $facet: {
      byBranch: [ { $group: { _id: "$branch", n: { $sum: 1 } } } ],
      topThree: [ { $sort: { cgpa: -1 } }, { $limit: 3 }, { $project: { _id: 0, name: 1 } } ]
  } }
])
```
`$facet` computes several results from **one pass** of the input (dashboards!).

### A11 — explain an aggregation
```js
db.students.aggregate([ { $match: { branch: "CSE" } }, { $group: { _id: "$year", n: { $sum: 1 } } } ])
        .explain("executionStats")
```
With an index on `branch`, the first `$match` uses it (look for `IXSCAN`). A `$match` placed **after** `$group` cannot use that index.

### A12 — writing results (awareness)
```js
db.students.aggregate([
  { $group: { _id: "$branch", avgCgpa: { $avg: "$cgpa" } } },
  { $merge: { into: "branch_stats", whenMatched: "replace", whenNotMatched: "insert" } }
])
```
`$out` / `$merge` **write** to a collection (a materialised summary). Treat them as dangerous in user-supplied pipelines.

## 8.4 Common mistakes
```
❌ $match placed after $group on a field that no longer exists (e.g. "branch" after grouping by it → now "_id")
❌ Forgetting the "$" in field paths:  "$cgpa"  (field value)  vs  "cgpa"  (a literal string)
❌ $group without _id → syntax error (use _id: null for "everything")
❌ $project mixing inclusion and exclusion (except _id)
❌ $lookup without an index on foreignField → slow
❌ Expecting $group output to be sorted → it is NOT; add $sort
❌ Averaging nulls/missing silently → $avg ignores non-numeric/missing values (check counts!)
```

## 8.5 RESULT
Aggregation pipelines using `$match`, `$group`, `$project`, `$sort`, `$limit`, `$unwind`, `$addFields`, `$lookup`, `$bucket`, `$count` and `$facet` were built and verified.

## 8.6 🎤 VIVA — Practical 8

**Q1. What is the aggregation pipeline?**
> It is MongoDB's framework for data processing: an **ordered list of stages**, each transforming the stream of documents it receives and passing the result on. It supports filtering, grouping, reshaping, joining, bucketing and more, executing inside the database engine so that only results travel to the application — much more efficient than fetching documents and processing them in code.

**Q2. Why should `$match` be early in a pipeline?**
> It reduces the number of documents flowing through every later stage and — when it is the first stage — it can use an **index**. Putting `$match` after `$group` or `$project` often forces a full scan and wastes work. MongoDB's optimiser can sometimes reorder stages, but you shouldn't rely on it.

**Q3. Difference between `$project` and `$group`?**
> `$project` works **per document**: it chooses and computes fields, renames, and reshapes, producing one output per input. `$group` works **across documents**: it combines many inputs into one output per key using accumulators like `$sum`, `$avg`, `$max`. `$group` changes the document shape completely (the key becomes `_id`).

**Q4. What does `$unwind` do and what happens to empty arrays?**
> It deconstructs an array field, emitting one document per element with the array replaced by that element. Documents where the array is empty, null or missing are **dropped** unless `preserveNullAndEmptyArrays: true` is specified.

**Q5. How does `$lookup` compare with SQL joins?**
> It is a **left outer join**: every input document is kept, with an array of matching documents from the other collection (possibly empty). It works on collections in the same database, matches on equality of `localField` and `foreignField` (or via a sub-pipeline with `let/pipeline` for complex conditions), and needs an index on the foreign field. It is deliberately not the first tool in MongoDB — good data modelling reduces the need for joins.

**Q6. `$group` vs `$bucket`?**
> `$group` groups by **exact values** of an expression; `$bucket` groups numeric/date values into **ranges** you define with `boundaries`, with an optional `default` bucket for out-of-range values. It's perfect for histograms like CGPA bands.

**Q7. How do you debug or optimise a slow pipeline?**
> Run `explain("executionStats")` on the aggregation; ensure the leading `$match`/`$sort` uses an index; move `$match`/`$limit` early; trim documents with `$project` before heavy stages; index `$lookup` foreign fields; avoid `$unwind` on huge arrays when a `$filter`/`$size` would do; consider pre-aggregated summary collections (`$merge`); and use `allowDiskUse` for memory-heavy stages when necessary (memory limits apply per stage; newer versions allow spilling by default — check your version).

**Q8. What is `$facet` useful for?**
> Running **multiple independent sub-pipelines over the same input** in one command — e.g., one dashboard request returning counts-by-category, top items and price histogram — without scanning the data several times. Note that sub-pipelines can't use indexes themselves, and the combined output is subject to the 16 MB document limit.

**Q9. How is an aggregation different from `find()`?**
> `find` selects and projects existing documents (with sort/limit). Aggregation can **compute new data**: group, summarise, join, reshape, bucket, write results. Every `find` can be written as an aggregation (`$match` + `$project` + `$sort`), but not vice versa.

**Q10. What does `$out`/`$merge` do and what is the risk?**
> They write the pipeline's results into a collection — `$out` replaces the target, `$merge` can insert/update/merge into it. They turn a read into a **write**. In an app that lets users or an AI supply pipelines, these stages must be blocked, and the executing user should have only read privileges.

**🧑‍💼 Interview angle:** *"Write a pipeline for top 3 products by revenue per month."* → `$match` date range (indexed) → `$unwind` items → `$group` by `{month, productId}` summing `price*qty` → `$sort` → `$group` by month with `$push` of products → `$project` with `$slice: [..., 3]`. Mention the **index** on the date field, `$lookup` for product names only **after** reducing the data, and caching/pre-aggregation (`$merge` summary collection) if the report is hot.

---

# 🧪 PRACTICAL 9 — INTEGRATE MONGODB WITH Node.js

## 9.1 Aim
To connect a Node.js application to MongoDB, perform CRUD from code using (a) the **official `mongodb` driver** and (b) **Mongoose**, handle errors, and expose a small REST API with Express.

## 9.2 Theory

### Why a driver?
Your program and `mongod` speak different languages. A **driver** is a library that (1) opens and **pools** TCP connections, (2) converts JavaScript objects ⇄ BSON, (3) sends commands using MongoDB's wire protocol, (4) handles retries, server discovery and failover.

```
Your Node app ──► Driver (mongodb)  ──TCP 27017──►  mongod
   JS objects      BSON encode/decode, pooling          stores BSON
                   ▲
          Mongoose sits ON TOP: schemas, validation, middleware, populate
```

| | **Native driver (`mongodb`)** | **Mongoose** |
|---|---|---|
| Nature | Thin official client | **ODM** (Object-Document Mapper) built on the driver |
| Schema | None — any shape | Defined in code (types, required, min/max, enum, default) |
| Validation | Manual / `$jsonSchema` | Built-in validators |
| Extras | Maximum control, minimal overhead | Models, virtuals, hooks (middleware), `populate()`, query helpers |
| Best for | Scripts, performance-critical thin layers | Structured web apps with many models |

### Golden rules (viva + production)
```
1. ONE MongoClient / one mongoose connection per application (it owns the connection POOL).
2. Everything is asynchronous → use async/await; forgetting await is bug #1.
3. Credentials come from environment variables (.env), never hard-coded, never committed.
4. Handle errors: duplicate key (11000), validation errors, connection errors.
5. Validate/sanitise client input (NoSQL injection: {"$ne": null}).
```

## 9.3 Procedure

### Step 1 — project setup
```bash
node -v                     # need a current LTS version (18+; check the driver's docs for its minimum)
mkdir nodelab && cd nodelab
npm init -y
npm install mongodb mongoose dotenv express
```
Open `package.json` and add the line `"type": "module",` (so you can use `import`):
```json
{
  "name": "nodelab",
  "version": "1.0.0",
  "type": "module",
  "scripts": { "start": "node server.js" }
}
```

`.env` (secret — **never commit**):
```
MONGODB_URI=mongodb://127.0.0.1:27017
DB_NAME=college_node
```
`.gitignore`:
```
node_modules
.env
```
(For Atlas use `mongodb+srv://user:pass@cluster0.xxxxx.mongodb.net/` as the URI.)

### Step 2 — `ping.js`: prove the connection
```js
// ping.js
import "dotenv/config";
import { MongoClient } from "mongodb";

const client = new MongoClient(process.env.MONGODB_URI);

try {
  await client.connect();
  const res = await client.db("admin").command({ ping: 1 });
  console.log("✅ Connected. ping ->", res);
} catch (err) {
  console.error("❌ Connection failed:", err.message);
  process.exitCode = 1;
} finally {
  await client.close();
}
```
Run `node ping.js`:
```
✅ Connected. ping -> { ok: 1 }
```
*(Top-level `await` works because of `"type": "module"`.)*

### Step 3 — `crud_driver.js`: full CRUD with the native driver
```js
// crud_driver.js
import "dotenv/config";
import { MongoClient, ObjectId } from "mongodb";

const client = new MongoClient(process.env.MONGODB_URI);

try {
  await client.connect();
  const col = client.db(process.env.DB_NAME).collection("students");

  // 0. clean start + a unique index (constraint)
  await col.deleteMany({});
  await col.createIndex({ rollNo: 1 }, { unique: true });

  // 1. CREATE
  const r1 = await col.insertOne({ rollNo: "CSE2301", name: "Aarav", branch: "CSE", cgpa: 8.4 });
  console.log("1. insertOne ->", r1.acknowledged, "| id is", r1.insertedId.constructor.name);

  const r2 = await col.insertMany([
    { rollNo: "CSE2302", name: "Diya",  branch: "CSE", cgpa: 9.1 },
    { rollNo: "ECE2301", name: "Kabir", branch: "ECE", cgpa: 7.2 },
    { rollNo: "ECE2302", name: "Meera", branch: "ECE", cgpa: 8.0 }
  ]);
  console.log("2. insertMany -> inserted", r2.insertedCount);

  // 2. READ
  const cse = await col.find({ branch: "CSE" }).sort({ cgpa: -1 }).project({ _id: 0, name: 1, cgpa: 1 }).toArray();
  console.log("3. CSE by cgpa desc:", cse);

  const one = await col.findOne({ rollNo: "ECE2302" });
  console.log("4. findOne ->", one.name);

  const byId = await col.findOne({ _id: new ObjectId(String(r1.insertedId)) });   // string → ObjectId
  console.log("5. findOne by _id ->", byId.name);

  // 3. UPDATE
  const u1 = await col.updateOne({ rollNo: "CSE2301" }, { $set: { cgpa: 8.6 } });
  console.log("6. updateOne -> matched", u1.matchedCount, "modified", u1.modifiedCount);

  const f1 = await col.findOneAndUpdate(
    { rollNo: "CSE2302" }, { $set: { cgpa: 9.3 } }, { returnDocument: "after" });
  console.log("7. findOneAndUpdate ->", f1.name, f1.cgpa);

  const u2 = await col.updateMany({ branch: "ECE" }, { $set: { dept: "Electronics" } });
  console.log("8. updateMany -> matched", u2.matchedCount, "modified", u2.modifiedCount);

  // 4. ERROR HANDLING: duplicate key
  try {
    await col.insertOne({ rollNo: "CSE2301", name: "Duplicate", branch: "CSE", cgpa: 5 });
  } catch (err) {
    console.log("9. duplicate ->", err.code === 11000 ? "E11000 duplicate key (code 11000)" : err.message);
  }

  // 5. DELETE
  const d1 = await col.deleteOne({ rollNo: "ECE2301" });
  console.log("10. deleteOne -> deleted", d1.deletedCount);

  // 6. COUNT + AGGREGATE
  console.log("11. countDocuments ->", await col.countDocuments());
  const agg = await col.aggregate([
    { $group: { _id: "$branch", n: { $sum: 1 }, top: { $max: "$cgpa" } } },
    { $sort: { _id: 1 } }
  ]).toArray();
  console.log("12. aggregate ->", agg);
} catch (err) {
  console.error("❌ Error:", err.message);
  process.exitCode = 1;
} finally {
  await client.close();        // always release the connection
}
```
Run `node crud_driver.js`. Expected output (ObjectIds are not printed, so this is deterministic):
```
1. insertOne -> true | id is ObjectId
2. insertMany -> inserted 3
3. CSE by cgpa desc: [ { name: 'Diya', cgpa: 9.1 }, { name: 'Aarav', cgpa: 8.4 } ]
4. findOne -> Meera
5. findOne by _id -> Aarav
6. updateOne -> matched 1 modified 1
7. findOneAndUpdate -> Diya 9.3
8. updateMany -> matched 2 modified 2
9. duplicate -> E11000 duplicate key (code 11000)
10. deleteOne -> deleted 1
11. countDocuments -> 3
12. aggregate -> [ { _id: 'CSE', n: 2, top: 9.3 }, { _id: 'ECE', n: 1, top: 8 } ]
```

**Line-by-line ideas to explain:**
```
import "dotenv/config"        loads .env into process.env
new MongoClient(uri)          creates the client + POOL (no network call yet)
await client.connect()        opens connections / discovers the server
client.db(name).collection()  lightweight handles — no network I/O
.find() returns a CURSOR      lazily fetches; .toArray() pulls all results into memory
new ObjectId(string)          needed to match _id (string ≠ ObjectId)
returnDocument: "after"       get the NEW document (default returns the old one)
finally { client.close() }    scripts should close; long-running servers keep ONE client open
```
> Version note: in current driver versions `findOneAndUpdate` returns the document directly; older (v5-and-earlier) versions returned a wrapper `{ value: ... }`. Check the version you installed.

### Step 4 — Mongoose: `student.model.js`
```js
// student.model.js
import mongoose from "mongoose";

const studentSchema = new mongoose.Schema(
  {
    rollNo: { type: String, required: [true, "rollNo is required"], unique: true, trim: true,
              match: [/^[A-Z]{2,4}[0-9]{4}$/, "rollNo must look like CSE2301"] },
    name:   { type: String, required: true, trim: true, minlength: 2, maxlength: 50 },
    branch: { type: String, enum: ["CSE", "ECE", "EEE", "ME", "CE"], required: true },
    year:   { type: Number, min: 1, max: 4, default: 1 },
    cgpa:   { type: Number, min: 0, max: 10, required: true },
    skills: { type: [String], default: [] }
  },
  { timestamps: true }          // adds createdAt, updatedAt automatically
);

// virtual: computed on the fly, not stored
studentSchema.virtual("grade").get(function () {
  if (this.cgpa >= 9) return "O";
  if (this.cgpa >= 8) return "A";
  if (this.cgpa >= 7) return "B";
  return "C";
});

// index for a common list query
studentSchema.index({ branch: 1, cgpa: -1 });

export default mongoose.model("Student", studentSchema);   // collection name: "students"
```

### Step 5 — `crud_mongoose.js`
```js
// crud_mongoose.js
import "dotenv/config";
import mongoose from "mongoose";
import Student from "./student.model.js";

try {
  await mongoose.connect(process.env.MONGODB_URI, { dbName: "college_mongoose" });
  await Student.deleteMany({});
  await Student.init();                       // make sure indexes (incl. unique) are built before testing

  // CREATE
  const a = await Student.create({ rollNo: "CSE2301", name: "Aarav", branch: "CSE", year: 2, cgpa: 8.4 });
  await Student.insertMany([
    { rollNo: "CSE2302", name: "Diya",  branch: "CSE", year: 3, cgpa: 9.1 },
    { rollNo: "ECE2301", name: "Kabir", branch: "ECE", year: 2, cgpa: 7.2 }
  ]);
  console.log("1. created:", a.name, "| grade (virtual):", a.grade);

  // READ
  const list = await Student.find({ branch: "CSE" }).sort({ cgpa: -1 }).select("name cgpa -_id").lean();
  console.log("2. CSE list:", list);

  // UPDATE  (new: true → return updated doc; runValidators → enforce schema rules on updates!)
  const up = await Student.findOneAndUpdate(
    { rollNo: "ECE2301" }, { cgpa: 7.9 }, { new: true, runValidators: true });
  console.log("3. updated:", up.name, up.cgpa, "→ grade", up.grade);

  // VALIDATION ERROR
  try {
    await Student.create({ rollNo: "CSE2303", name: "Sana", branch: "CSE", cgpa: 11 });
  } catch (err) {
    console.log("4. validation:", err.name, "→", err.errors.cgpa.message);
  }

  // DUPLICATE KEY ERROR (unique is an INDEX, not a validator)
  try {
    await Student.create({ rollNo: "CSE2301", name: "Dup", branch: "CSE", cgpa: 5 });
  } catch (err) {
    console.log("5. duplicate → code", err.code);
  }

  // DELETE
  const del = await Student.deleteOne({ rollNo: "ECE2301" });
  console.log("6. deleted:", del.deletedCount, "| remaining:", await Student.countDocuments());
} catch (err) {
  console.error("❌", err.message);
  process.exitCode = 1;
} finally {
  await mongoose.disconnect();
}
```
Expected output:
```
1. created: Aarav | grade (virtual): A
2. CSE list: [ { name: 'Diya', cgpa: 9.1 }, { name: 'Aarav', cgpa: 8.4 } ]
3. updated: Kabir 7.9 → grade B
4. validation: ValidationError → Path `cgpa` (11) is more than maximum allowed value (10).
5. duplicate → code 11000
6. deleted: 1 | remaining: 2
```

**What to be able to explain:**
```
Schema → Model → Document        blueprint → collection-bound class → one record
timestamps: true                  createdAt / updatedAt maintained automatically
virtual                           computed property, not stored
runValidators: true               validators DON'T run on updates by default!
.lean()                           returns plain JS objects: faster, but no virtuals/methods
Student.init()                    waits for index creation — otherwise a duplicate test may run before the unique index exists
unique: true                      creates an INDEX → duplicate gives code 11000, not ValidationError
```

### Step 6 — a tiny REST API: `server.js`
```js
// server.js
import "dotenv/config";
import express from "express";
import mongoose from "mongoose";
import Student from "./student.model.js";

const app = express();
app.use(express.json());

// GET /api/students?branch=CSE
app.get("/api/students", async (req, res, next) => {
  try {
    // String() blocks query-operator injection like ?branch[$ne]=x
    const filter = req.query.branch ? { branch: String(req.query.branch) } : {};
    const students = await Student.find(filter).sort({ cgpa: -1 }).limit(50).lean();
    res.json(students);
  } catch (err) { next(err); }
});

// POST /api/students
app.post("/api/students", async (req, res, next) => {
  try {
    const { rollNo, name, branch, year, cgpa } = req.body;       // whitelist fields
    const created = await Student.create({ rollNo, name, branch, year, cgpa });
    res.status(201).json(created);
  } catch (err) { next(err); }
});

// central error handler → correct HTTP status codes
app.use((err, req, res, next) => {
  if (err.name === "ValidationError") return res.status(400).json({ error: err.message });
  if (err.code === 11000)            return res.status(409).json({ error: "Duplicate value" });
  console.error(err);
  res.status(500).json({ error: "Server error" });
});

await mongoose.connect(process.env.MONGODB_URI, { dbName: "college_mongoose" });
const port = process.env.PORT || 3000;
app.listen(port, () => console.log(`API running on http://localhost:${port}`));
```
Test with a second terminal:
```bash
curl -X POST http://localhost:3000/api/students \
  -H "Content-Type: application/json" \
  -d '{"rollNo":"CSE2310","name":"Neha","branch":"CSE","year":2,"cgpa":8.7}'
# → 201 with the created document

curl http://localhost:3000/api/students?branch=CSE
# → JSON array

curl -X POST http://localhost:3000/api/students -H "Content-Type: application/json" \
  -d '{"rollNo":"CSE2310","name":"Neha","branch":"CSE","year":2,"cgpa":8.7}'
# → 409 {"error":"Duplicate value"}

curl -X POST http://localhost:3000/api/students -H "Content-Type: application/json" \
  -d '{"rollNo":"bad","name":"X","branch":"CSE","cgpa":99}'
# → 400 {"error":"Student validation failed: ..."}
```

## 9.4 Status codes you must know (REST + MongoDB)
| Situation | Code |
|---|---|
| Read/update OK | 200 |
| Created | 201 |
| Deleted (no body) | 204 |
| Validation failed / malformed id | 400 |
| Not found | 404 |
| Duplicate key / business-rule conflict | 409 |
| Unexpected failure | 500 |

## 9.5 Common errors
| Error | Cause | Fix |
|---|---|---|
| `Cannot use import statement outside a module` | no `"type": "module"` | add it, or use `.mjs` |
| `MongoServerSelectionError` | server/URI/IP problem | check Practical 1 |
| `Cannot read properties of undefined` after `findOne` | document not found (`null`) | check for `null` before using |
| Result is a `Promise {<pending>}` | missing `await` | add `await` |
| `E11000` | unique index violated | return 409 |
| `CastError` | malformed ObjectId | validate with `mongoose.isValidObjectId` → 400 |
| App hangs after script ends | client not closed | `finally { await client.close() }` |
| Update ignores schema rules | `runValidators` missing | add `{ runValidators: true }` |

## 9.6 RESULT
A Node.js application connected to MongoDB, performed CRUD with the native driver and with Mongoose, handled validation and duplicate-key errors, and served a REST API.

## 9.7 🎤 VIVA — Practical 9

**Q1. Why do we use a driver, and what does it do for us?**
> A driver is the language-specific library that lets application code talk to MongoDB. It manages **connection pooling**, **serialises JavaScript objects to BSON** and back, implements the wire protocol, performs **server discovery and monitoring** (so it can follow a replica-set primary after failover), and supports **retryable reads/writes** and timeouts. Without it you'd have to implement the protocol yourself.

**Q2. Why should an application create only one `MongoClient`?**
> A `MongoClient` owns a **connection pool** and background monitoring threads. Creating a new client per request defeats pooling, floods the server with connections, adds handshake latency and can exhaust server connection limits under load. The correct pattern is one shared client created at startup (and closed at shutdown) and reused everywhere.

**Q3. What is Mongoose, and why use it over the native driver?**
> Mongoose is an **ODM** that adds a schema layer: typed fields, defaults, validators, indexes, virtuals, instance/static methods, middleware hooks, and `populate()` for references. It reduces boilerplate and prevents malformed data in structured applications. The cost is an abstraction layer with some overhead and less direct access to every server feature, so for thin or performance-critical code, or for scripts, the native driver can be a better fit.

**Q4. Explain Schema, Model and Document in Mongoose.**
> A **Schema** defines the structure, types, defaults, validators and indexes of documents. A **Model** is a compiled class bound to a collection that provides the query/CRUD interface (`Student.find`, `Student.create`). A **Document** is an instance of a model — one record, with its own methods like `save()` and virtuals.

**Q5. Why do Mongoose updates sometimes bypass validation?**
> By default, validators run on `save()`/`create()`, but query-based updates like `findOneAndUpdate`/`updateOne` skip them for performance. You must pass `{ runValidators: true }` (and be aware of `context` limitations for some validators) to enforce schema rules on updates.

**Q6. What does `lean()` do?**
> It returns **plain JavaScript objects** instead of full Mongoose documents. That skips hydration, getters, virtuals and change tracking, making reads faster and lighter on memory — ideal for read-only API responses. You lose document methods and virtuals.

**Q7. How do you prevent NoSQL injection in a Node API?**
> Never pass raw `req.body`/`req.query` objects into a filter. Whitelist fields; coerce to expected types (`String(value)`); validate with a schema library (Joi/Zod/express-validator); disable or sanitise query parsing that creates operator objects (`?email[$ne]=1`); avoid `$where` and server-side JavaScript; use least-privilege DB users; and hash passwords so even a bypass doesn't reveal secrets.

**Q8. How do you handle a duplicate-key error in an API?**
> Catch the error, check `err.code === 11000`, and respond with **409 Conflict** and a friendly message (optionally naming the field from `err.keyValue`). The unique index guarantees correctness even under concurrent requests, which an application-level "find then insert" check cannot.

**Q9. What is the difference between `find().toArray()` and iterating a cursor?**
> `find()` returns a **cursor** that fetches documents in batches lazily. `toArray()` loads **all** results into memory — fine for small results, dangerous for millions of documents. For large sets iterate with `for await (const doc of cursor)` (or use pagination/limits) so memory stays flat.

**Q10. What happens if MongoDB is down when the app starts, or goes down later?**
> At start, `connect()` fails after the server-selection timeout (default about 30 seconds) with a `MongoServerSelectionError` — handle it and exit or retry with backoff. Later, the driver keeps monitoring; operations during an outage fail after the timeout, while retryable reads/writes can transparently retry once across a failover. Add health-check endpoints, sensible timeouts and logging.

**🧑‍💼 Interview angle:** *"How would you structure a production Node + MongoDB service?"* → config from env/secret manager; single connection with tuned pool/timeouts; models + validation; repository/service layer; central error handling mapping Mongo errors → HTTP codes; input validation; pagination with indexed sorts; structured logging; health/readiness endpoints; graceful shutdown (stop HTTP, close client); tests with an in-memory or containerised MongoDB; Dockerfile; CI.

---

# 🧪 PRACTICAL 10 — ESTABLISH A MONGODB CONNECTION USING PYTHON (PyMongo)

## 10.1 Aim
To connect Python to MongoDB using **PyMongo**, verify the connection, and perform CRUD, aggregation and error handling.

## 10.2 Theory

**PyMongo** is the official synchronous Python driver. Key points:

```
• Documents are plain Python dicts; arrays are lists; dates are datetime objects.
• Connection is LAZY: MongoClient(...) returns instantly — the first real operation (or an explicit ping)
  triggers server selection. So always ping to test.
• Method names are snake_case: insert_one, find_one, update_many, delete_one.
• Cursors are one-pass: once iterated they are exhausted.
• ObjectId lives in the bson package that ships WITH PyMongo → from bson import ObjectId
  (do NOT "pip install bson" — that is a different, conflicting package).
```

| Node.js | Python |
|---|---|
| `insertOne` | `insert_one` |
| `find(...).toArray()` | `list(col.find(...))` |
| `.sort({ cgpa: -1 })` | `.sort("cgpa", -1)` |
| `{ returnDocument: "after" }` | `return_document=ReturnDocument.AFTER` |
| `err.code === 11000` | `except DuplicateKeyError` |
| `await` everywhere | synchronous calls (an async API exists in recent versions) |

## 10.3 Procedure

### Step 1 — environment
```bash
python --version                        # 3.9+ recommended; check the PyMongo docs for the minimum
mkdir pylab && cd pylab
python -m venv venv
# Windows:   venv\Scripts\activate
# macOS/Linux: source venv/bin/activate
pip install "pymongo[srv]" python-dotenv pandas scikit-learn   # srv extra needed for mongodb+srv:// (Atlas)
```
`.env`
```
MONGODB_URI=mongodb://127.0.0.1:27017
```

### Step 2 — `ping.py`
```python
# ping.py
import os
from dotenv import load_dotenv
from pymongo import MongoClient
from pymongo.errors import ServerSelectionTimeoutError

load_dotenv()
client = MongoClient(os.environ["MONGODB_URI"], serverSelectionTimeoutMS=5000)

try:
    print("✅ ping ->", client.admin.command("ping"))
except ServerSelectionTimeoutError as e:
    print("❌ cannot reach MongoDB:", e)
finally:
    client.close()
```
Run `python ping.py`:
```
✅ ping -> {'ok': 1.0}
```
`serverSelectionTimeoutMS=5000` makes failures appear in 5 seconds instead of the default 30.

### Step 3 — `crud_pymongo.py`
```python
# crud_pymongo.py
import os
from bson import ObjectId
from dotenv import load_dotenv
from pymongo import MongoClient, ASCENDING, ReturnDocument
from pymongo.errors import DuplicateKeyError, PyMongoError

load_dotenv()
client = MongoClient(os.environ["MONGODB_URI"], serverSelectionTimeoutMS=5000)

try:
    col = client["college_py"]["students"]

    # 0. clean start + unique index
    col.delete_many({})
    col.create_index([("rollNo", ASCENDING)], unique=True)

    # 1. CREATE
    r1 = col.insert_one({"rollNo": "CSE2301", "name": "Aarav", "branch": "CSE", "cgpa": 8.4})
    print("1. insert_one ->", r1.acknowledged, "| id type:", type(r1.inserted_id).__name__)

    r2 = col.insert_many([
        {"rollNo": "CSE2302", "name": "Diya",  "branch": "CSE", "cgpa": 9.1},
        {"rollNo": "ECE2301", "name": "Kabir", "branch": "ECE", "cgpa": 7.2},
        {"rollNo": "ECE2302", "name": "Meera", "branch": "ECE", "cgpa": 8.0},
    ])
    print("2. insert_many -> inserted", len(r2.inserted_ids))

    # 2. READ
    cse = list(col.find({"branch": "CSE"}, {"_id": 0, "name": 1, "cgpa": 1}).sort("cgpa", -1))
    print("3. CSE by cgpa desc:", cse)

    print("4. find_one ->", col.find_one({"rollNo": "ECE2302"})["name"])
    print("5. by _id ->", col.find_one({"_id": ObjectId(str(r1.inserted_id))})["name"])

    # 3. UPDATE
    u1 = col.update_one({"rollNo": "CSE2301"}, {"$set": {"cgpa": 8.6}})
    print("6. update_one -> matched", u1.matched_count, "modified", u1.modified_count)

    f1 = col.find_one_and_update({"rollNo": "CSE2302"}, {"$set": {"cgpa": 9.3}},
                                 return_document=ReturnDocument.AFTER)
    print("7. find_one_and_update ->", f1["name"], f1["cgpa"])

    u2 = col.update_many({"branch": "ECE"}, {"$set": {"dept": "Electronics"}})
    print("8. update_many -> matched", u2.matched_count, "modified", u2.modified_count)

    # 4. ERROR HANDLING
    try:
        col.insert_one({"rollNo": "CSE2301", "name": "Duplicate", "branch": "CSE", "cgpa": 5})
    except DuplicateKeyError:
        print("9. duplicate -> DuplicateKeyError (E11000)")

    # 5. DELETE
    print("10. delete_one -> deleted", col.delete_one({"rollNo": "ECE2301"}).deleted_count)

    # 6. COUNT + AGGREGATE
    print("11. count_documents ->", col.count_documents({}))
    agg = list(col.aggregate([
        {"$group": {"_id": "$branch", "n": {"$sum": 1}, "top": {"$max": "$cgpa"}}},
        {"$sort": {"_id": 1}},
    ]))
    print("12. aggregate ->", agg)

    # 7. Cursor is one-pass
    cur = col.find({})
    first_pass = list(cur)
    second_pass = list(cur)
    print("13. cursor reuse ->", len(first_pass), "then", len(second_pass))

except PyMongoError as e:
    print("❌ MongoDB error:", e)
finally:
    client.close()
```
Expected output:
```
1. insert_one -> True | id type: ObjectId
2. insert_many -> inserted 3
3. CSE by cgpa desc: [{'name': 'Diya', 'cgpa': 9.1}, {'name': 'Aarav', 'cgpa': 8.4}]
4. find_one -> Meera
5. by _id -> Aarav
6. update_one -> matched 1 modified 1
7. find_one_and_update -> Diya 9.3
8. update_many -> matched 2 modified 2
9. duplicate -> DuplicateKeyError (E11000)
10. delete_one -> deleted 1
11. count_documents -> 3
12. aggregate -> [{'_id': 'CSE', 'n': 2, 'top': 9.3}, {'_id': 'ECE', 'n': 1, 'top': 8.0}]
13. cursor reuse -> 3 then 0
```

### Step 4 — dates and time zones (a PyMongo-specific trap)
```python
from datetime import datetime, timezone
col.insert_one({"event": "login", "at": datetime.now(timezone.utc)})   # store timezone-aware UTC
doc = col.find_one({"event": "login"})
print(doc["at"])          # PyMongo returns naive UTC datetimes by default unless tz_aware=True on the client
```
`MongoClient(uri, tz_aware=True)` returns aware datetimes. Storing naive local times causes silent offset bugs.

### Step 5 — Atlas connection
```python
uri = "mongodb+srv://<user>:<password>@cluster0.xxxxx.mongodb.net/?retryWrites=true&w=majority"
client = MongoClient(uri)
print(client.admin.command("ping"))
```
Checklist: IP in Network Access, DB user exists, password URL-encoded (`urllib.parse.quote_plus`), `pymongo[srv]` installed.

## 10.4 Common errors
| Error | Cause | Fix |
|---|---|---|
| `ServerSelectionTimeoutError` | server down / wrong host / Atlas IP not allowed | check server + URI + allow-list |
| `ModuleNotFoundError: pymongo` | wrong interpreter / venv not active | activate venv; `pip install pymongo` |
| `ImportError` involving `bson` | standalone `bson` package installed | `pip uninstall bson` then reinstall `pymongo` |
| `InvalidURI` / SRV error | missing `srv` extra / DNS blocked | `pip install "pymongo[srv]"`; check network |
| Second loop over cursor is empty | cursor exhausted | `list(cursor)` once and reuse |
| `bson.errors.InvalidId` | malformed id string | validate with `ObjectId.is_valid(s)` |
| `TypeError: 'Collection' object is not callable` | calling `col.find` without `()` or wrong attribute | check names |
| Naive vs aware datetime confusion | tz handling | use UTC-aware datetimes |

## 10.5 RESULT
Python connected to MongoDB using PyMongo; `ping` succeeded; CRUD, aggregation and error handling were demonstrated.

## 10.6 🎤 VIVA — Practical 10

**Q1. What is PyMongo and how is the connection established?**
> PyMongo is MongoDB's official Python driver. You create a `MongoClient` with a connection string; it builds a connection pool and starts background server monitoring but does **not** connect immediately (lazy). The first operation, or an explicit `client.admin.command("ping")`, triggers server selection. You then access `client[db_name][collection_name]` (lightweight handles) and call methods like `insert_one` and `find`.

**Q2. Why is the PyMongo connection called "lazy" and how do you test it?**
> Constructing `MongoClient` returns immediately even if the server is down, because connection and discovery happen in the background. To validate the setup I run `ping` (or any command) inside a `try/except ServerSelectionTimeoutError`, optionally with a short `serverSelectionTimeoutMS` so failures are quick.

**Q3. Compare PyMongo with the Node.js driver.**
> Same MongoDB features and similar semantics, different language idioms. Node's API is **async** (promises/`await`), uses camelCase and returns cursors with `toArray()`; PyMongo is classically **synchronous** (an async API exists in recent releases, and Motor was the earlier async option), uses snake_case, dicts and `list(cursor)`. In both you should use one shared client, handle duplicate-key errors, and keep credentials in environment variables.

**Q4. Where does `ObjectId` come from in Python and what is the common mistake?**
> `from bson import ObjectId` — the `bson` package is bundled with PyMongo. The common mistake is running `pip install bson`, which installs an unrelated library that conflicts with PyMongo's `bson` and causes import errors. Fix: uninstall it and reinstall PyMongo.

**Q5. A cursor returned data in the first loop and nothing in the second. Why?**
> A PyMongo cursor is a one-pass iterator over server batches. Once exhausted it yields nothing. Convert to a list (`results = list(cursor)`) if you need to iterate multiple times, or re-run `find()`.

**Q6. How do you handle errors in PyMongo?**
> Catch specific exceptions from `pymongo.errors`: `DuplicateKeyError` for unique-index violations, `ServerSelectionTimeoutError` for connectivity, `OperationFailure` for server-side command failures (auth, validation), `BulkWriteError` for bulk operation failures, with `PyMongoError` as the base class. Log context, return friendly messages and avoid blanket `except Exception` that hides bugs.

**Q7. When would you choose Python over Node for MongoDB work?**
> For **data analysis, scripting, ETL and machine learning** — the pandas/scikit-learn ecosystem pairs naturally with PyMongo and aggregation pipelines. Node is typically chosen for real-time web APIs sharing a JavaScript stack with the front end. Many systems use both against the same database: Node for the API, Python for analytics.

**Q8. How do you store and read dates correctly from Python?**
> Store **timezone-aware UTC** datetimes (`datetime.now(timezone.utc)`). MongoDB stores UTC milliseconds; PyMongo returns naive UTC datetimes unless the client is created with `tz_aware=True`. Convert to local time only for display.

**🧑‍💼 Interview angle:** *"Load collection data into pandas for analysis — how, and what are the pitfalls?"* → Use an **aggregation pipeline** to filter/shape inside MongoDB, project only needed fields, stream or batch large results rather than `list(find())` on millions of docs, flatten nested documents with `pd.json_normalize`, handle missing fields/NaN and type drift, avoid leaking the target into features, and write predictions back with `bulk_write`.

---

# 🧪 BONUS PRACTICALS — UNIT V & VI IN ACTION

*(Not in the numbered lab list, but your syllabus covers them and viva questions come from them. Do these if time permits.)*

## B1 — MongoDB for AI-based predictive analytics

**Aim:** read student data from MongoDB, train a model to predict pass/fail, evaluate it honestly, and log the run back to MongoDB.

**Pipeline:**
```
MongoDB ─► find()/aggregate() ─► pandas DataFrame ─► train/test split ─► Pipeline(scale+model)
        ◄─ model_runs log (metrics, date)                               ─► metrics on UNSEEN test data
```

`predict_pass.py`:
```python
# predict_pass.py — MongoDB -> pandas -> scikit-learn -> back to MongoDB
import os
import random
from datetime import datetime, timezone

import pandas as pd
from sklearn.linear_model import LogisticRegression
from sklearn.metrics import accuracy_score, classification_report
from sklearn.model_selection import train_test_split
from sklearn.pipeline import Pipeline
from sklearn.preprocessing import StandardScaler

random.seed(42)

def seed_students(col, n=300):
    """Insert synthetic students (skip if data already exists)."""
    if col.count_documents({}) >= n:
        return
    col.delete_many({})
    docs = []
    for i in range(n):
        attendance = random.randint(40, 100)
        internal = random.randint(10, 40)
        assignments = random.randint(0, 10)
        score = 0.5 * attendance + 1.2 * internal + 3 * assignments + random.gauss(0, 8)
        docs.append({"rollNo": f"S{i:04d}", "attendance": attendance, "internal": internal,
                     "assignments": assignments, "passed": 1 if score > 85 else 0})
    col.insert_many(docs)

def load_dataframe(col):
    rows = list(col.find({}, {"_id": 0, "attendance": 1, "internal": 1, "assignments": 1, "passed": 1}))
    return pd.DataFrame(rows)

def run(db):
    col = db["students_ml"]
    seed_students(col)
    df = load_dataframe(col)
    X = df[["attendance", "internal", "assignments"]]
    y = df["passed"]
    X_train, X_test, y_train, y_test = train_test_split(
        X, y, test_size=0.25, stratify=y, random_state=42)

    model = Pipeline([("scale", StandardScaler()),
                      ("clf", LogisticRegression(max_iter=1000))])
    model.fit(X_train, y_train)
    pred = model.predict(X_test)
    acc = accuracy_score(y_test, pred)
    print("rows:", len(df), "| pass rate:", round(float(y.mean()), 2))
    print("test accuracy:", round(float(acc), 3))
    print(classification_report(y_test, pred, digits=3))

    db["model_runs"].insert_one({
        "model": "logreg", "features": list(X.columns),
        "accuracy": float(acc), "trainedAt": datetime.now(timezone.utc)})
    return model

if __name__ == "__main__":
    from pymongo import MongoClient
    from dotenv import load_dotenv
    load_dotenv()
    client = MongoClient(os.environ["MONGODB_URI"])
    run(client["college_ai"])
```
Output (I ran the same ML code on this synthetic data with the database calls replaced by an in-memory stand-in; scikit-learn version differences may change the last digits):
```
rows: 300 | pass rate: 0.4
test accuracy: 0.84
              precision    recall  f1-score   support

           0      0.923     0.800     0.857        45
           1      0.750     0.900     0.818        30

    accuracy                          0.840        75
   macro avg      0.837     0.850     0.838        75
weighted avg      0.854     0.840     0.842        75
```
**Reading it:** 300 students, 40% pass. The model is right on 84% of the 75 held-out students; for class 1 (pass) recall 0.90 means it finds 90% of the students who truly pass, but precision 0.75 means one in four "pass" predictions is wrong. **Always report metrics on data the model never saw** (test split) — and note the data here is synthetic.

**Viva points:** features vs label; train/test split; **data leakage** (don't let future/target info into features); overfitting (cross-validation); class imbalance (use precision/recall/F1, not just accuracy); bias and privacy for student data; store `model_runs` for reproducibility.

## B2 — AI-driven query optimisation: profiler → analysis → fix

**Aim:** find slow, unindexed queries from MongoDB's own telemetry, then fix them with a verified index.

```js
use perflab
// 1. turn on the profiler: log operations slower than 20 ms into system.profile
db.setProfilingLevel(1, { slowms: 20 })

// 2. run some queries (e.g., the Practical 7 queries WITHOUT the index)
db.perf.find({ name: "Student77777" }).toArray()
db.perf.find({ cgpa: { $gt: 9.9 } }).toArray()

// 3. read the slowest operations and their plans
db.system.profile.find(
  { ns: "perflab.perf", millis: { $gt: 20 } },
  { _id: 0, op: 1, millis: 1, planSummary: 1, docsExamined: 1, nreturned: 1, keysExamined: 1 }
).sort({ millis: -1 }).limit(5)

// 4. find COLLSCAN offenders and their "waste" (docsExamined / nreturned)
db.system.profile.aggregate([
  { $match: { ns: "perflab.perf", planSummary: "COLLSCAN" } },
  { $project: { _id: 0, millis: 1, docsExamined: 1, nreturned: 1,
                waste: { $divide: ["$docsExamined", { $max: ["$nreturned", 1] }] } } },
  { $sort: { waste: -1 } }, { $limit: 5 }
])

// 5. fix, verify, measure again
db.perf.createIndex({ name: 1 })
db.perf.find({ name: "Student77777" }).explain("executionStats")   // IXSCAN, 1 doc examined

// 6. unused-index hygiene
db.perf.aggregate([ { $indexStats: {} } ])

// 7. safe experiment: HIDE an index (planner ignores it, but it is still maintained) before dropping
db.perf.hideIndex("name_1");  db.perf.unhideIndex("name_1")

// 8. turn the profiler off when done (it adds overhead)
db.setProfilingLevel(0)
```
**What "AI-driven" adds on top of this manual loop:** tools (Atlas Performance Advisor/Query Profiler on supported tiers, or your own scripts and ML) read thousands of such profile entries, group them by **query shape**, rank by impact, **suggest indexes** (ESR), detect latency anomalies and forecast workload — but a **human validates with `explain()`** before anything reaches production. *(Availability of Atlas features depends on the cluster tier — check current docs.)*

**Viva points:** profiler levels 0/1/2; `slowms`; `planSummary`; waste ratio; hidden indexes; human-in-the-loop; why LLM-generated queries need allow-lists, read-only users and `explain` guards.

## B3 — Mini project (CourseHub) in 12 commands

```bash
mkdir coursehub && cd coursehub && npm init -y
npm i express mongoose dotenv
# add "type":"module"; create .env (MONGODB_URI, DB_NAME) and .gitignore
# models: Student (rollNo*, email*), Course (code*, maxSeats, enrolledCount), Enrollment (student, course, marks, status)
# index: Enrollment { student:1, course:1 } UNIQUE
# routes: /api/students, /api/courses, /api/enrollments, /api/reports/*
# rule: atomic seat booking
#   Course.findOneAndUpdate({_id, $expr:{$lt:["$enrolledCount","$maxSeats"]}}, {$inc:{enrolledCount:1}}, {new:true})
node scripts/seed.js        # sample data
npm start                   # run API
node scripts/race.js        # fire simultaneous enrollments; enrolledCount must never exceed maxSeats
python analytics/risk_report.py   # at-risk students via PyMongo + aggregation
```
The full design, code, tests, report outline and demo script are in **Unit VI Parts 1–3**.

---

# 🎤 SECTION A — MASTER VIVA BANK (UNITS I → VI) WITH FULL "BEST-FIT" ANSWERS

> **How to answer (the 4-part shape examiners reward):**
> **① Definition → ② How it works → ③ Example (use students / CourseHub) → ④ Trade-off or when to use.**
> You don't need all four every time; two or three crisp parts beat a ramble. Each answer below is written in that shape so you can lift the structure, not memorise the words.

---

## 🅰️ UNIT I — Introduction to NoSQL and MongoDB

**V-I.1 Why did NoSQL databases emerge? What problems do they solve?**
> Web-scale applications produced data that was **huge (volume), fast-arriving (velocity) and varied (variety)**, and relational systems struggled in three ways: scaling a single big server up becomes very expensive, rigid schemas slow down iteration, and mapping objects to normalised tables adds friction. NoSQL systems were designed to **scale out** across commodity servers, accept **flexible structures**, and optimise for particular access patterns (key lookups, wide rows, documents, graph traversal). They do this by relaxing some relational guarantees — which is the trade-off: you gain scale and flexibility, and you take on more responsibility for consistency and integrity design.

**V-I.2 ACID vs BASE?**
> **ACID** — Atomicity, Consistency, Isolation, Durability — is the classic transactional guarantee of relational databases. **BASE** — Basically Available, Soft state, Eventually consistent — describes systems that prioritise availability and scale and let replicas converge over time. It's a spectrum, not a binary: MongoDB gives **ACID for a single document** always and supports **multi-document ACID transactions**, while also letting you choose weaker or stronger read/write guarantees per operation (read/write concerns). So I'd say MongoDB is "ACID where you need it, tunable elsewhere."

**V-I.3 Describe MongoDB's architecture.**
> A client application uses a **driver** to connect to **`mongod`**, the database server, which uses the **WiredTiger** storage engine (document-level concurrency control, compression, a journal for durability and periodic checkpoints). For **high availability** several `mongod` processes form a **replica set**: one **primary** accepts writes and records them in the **oplog**; **secondaries** replicate the oplog and can take over via an **election** if the primary fails. For **horizontal scale** a **sharded cluster** adds **`mongos`** routers, **config servers** (cluster metadata) and multiple **shards** (each a replica set), distributing data by a **shard key**. Atlas runs and manages all this for you.

**V-I.4 Explain replica sets and elections.**
> A replica set is a group of `mongod` instances holding the same data. The primary applies writes and logs them to the oplog; secondaries replay it. Members send heartbeats; if the primary becomes unreachable and a **majority** of voting members agree, a secondary is elected primary (typically within seconds), and drivers with retryable writes continue. An odd number of voting members (3 or 5) prevents ties; an **arbiter** votes but holds no data. Writes acknowledged with `w: "majority"` survive failover because a majority already has them.

**V-I.5 What is sharding and what makes a good shard key?**
> Sharding splits a collection across shards so no single server holds all data or takes all traffic. The **shard key** decides placement, so a good one has **high cardinality** (many distinct values), **even distribution** (no hot values), is **not monotonically increasing** (or inserts all hit one shard — use hashed or compound keys), and appears in most queries so they can be **targeted** to one shard instead of scatter-gathered. A bad key is hard to change later (resharding exists in newer versions but is a heavy operation), so I only shard when data/throughput truly exceeds one replica set.

**V-I.6 Compare MongoDB with Redis, Cassandra and Neo4j.**
> **Redis** is an in-memory key-value store — extremely fast caching/sessions, limited querying. **Cassandra** is a wide-column store built for massive, always-on, write-heavy workloads with tunable consistency and query-first table design. **Neo4j** is a graph database where relationship traversal is the main operation. **MongoDB** is a general-purpose document database with rich querying, secondary indexes, aggregation and flexible schemas. The right choice depends on the access pattern; real systems often combine them (e.g., MongoDB as system of record + Redis cache).

**V-I.7 Community vs Enterprise vs Atlas?**
> Community is the free, source-available server for self-hosting. Enterprise adds commercial features and support for self-managed deployments (e.g., advanced security/auditing integrations). Atlas is the fully managed cloud service (on major cloud providers) including backups, monitoring, scaling and security controls. Exact feature sets change, so I'd confirm in the current documentation.

**V-I.8 What are mongosh and Compass used for?**
> `mongosh` is the command-line JavaScript shell for administration, scripting and quick queries; Compass is the GUI to explore schemas, build queries and aggregation pipelines visually, inspect indexes and view `explain` plans. Both are **clients** of `mongod`; they don't store data.

---

## 🅱️ UNIT II — MongoDB Basics and CRUD

**V-II.1 What are namespaces, databases, collections, documents?**
> A **database** is a container of collections; a **collection** is a group of documents (comparable to a table, with no enforced schema unless validation is set); a **document** is a BSON record of field–value pairs. The full name `database.collection` is the **namespace**. Databases and collections are created lazily on first write.

**V-II.2 What is a capped collection?**
> A fixed-size collection that preserves insertion order and **overwrites the oldest documents** when full — like a circular buffer. Good for logs/recent-events with bounded storage. Limitations: you can't delete individual documents and growth-changing updates are restricted. For time-based expiry, a TTL index (or a time-series collection) is often the more flexible choice.

**V-II.3 How do you update a nested field or one array element?**
> Use dot notation: `{ $set: { "address.city": "Pune" } }`. For arrays: the positional operator `$` updates the first matched element (`{ "marks.$.score": 90 }` with a filter on the array), `$[]` updates **all** elements, and `$[<id>]` with `arrayFilters` updates the elements matching a condition. Example: `updateOne({_id:1},{ $set: {"marks.$[m].score": 90} },{ arrayFilters:[{ "m.subject":"DBMS" }] })`.

**V-II.4 How would you do a case-insensitive search efficiently?**
> A case-insensitive regex (`/pune/i`) cannot use a normal index well. Better: create the index with a **collation** (`{ locale: "en", strength: 2 }`) and query with the same collation, or store a normalised lowercase copy of the field and index that. For true text search use a text index or Atlas Search.

**V-II.5 `$in` vs `$or`?**
> `$in` tests whether **one field** equals any value in a list — compact and index-friendly. `$or` combines **different conditions** (possibly on different fields); each clause can use its own index, but every field in every clause needs an appropriate index to avoid a collection scan. For the same field prefer `$in`.

**V-II.6 What is a write concern and why does it matter at the basic level?**
> It defines **how many nodes must acknowledge a write** before the client gets success: `w:1` (primary only), `w:"majority"` (a majority of replica-set members), plus `j: true` for journal durability. It balances **durability vs latency**: `majority` survives a failover; `w:1` is faster but a write acknowledged only by the primary can be lost if the primary fails before replication. Recent MongoDB versions default to majority in many configurations — check your version.

---

## 🅲 UNIT III — Schema Design and Data Modeling

**V-III.1 What are the principles of MongoDB schema design?**
> (1) **Design for your queries and update patterns**, not for abstract entities. (2) **Data that is accessed together should be stored together** (embed). (3) Keep documents **bounded** — avoid unbounded arrays. (4) **Reference** when data is large, shared, or changes independently. (5) Accept **controlled duplication** where it speeds reads, and plan how it stays consistent. (6) Use **indexes** that match the queries. (7) Use **validation** and a `schemaVersion` so the model can evolve safely. I validate a design by listing the top queries and checking each is served by one read/one index.

**V-III.2 How do you handle one-to-squillions?**
> Never embed the children in the parent. Each child stores a **reference to the parent** (`deviceId`), with an index such as `{ deviceId: 1, ts: -1 }`. For time-series data use the **bucket pattern** (one document per device per hour containing an array of readings) or a native **time-series collection**, which handles bucketing and compression for you.

**V-III.3 When is denormalisation (duplication) acceptable, and how do you keep it consistent?**
> When the duplicated field is read very frequently, **changes rarely**, and the cost of an occasional multi-document update is acceptable — e.g., copying `studentName` into an enrollment for list pages. To keep it consistent: update copies in the same operation or a background job, tolerate brief staleness where the business allows, use **change streams** to propagate changes, and for critical data prefer references or transactions. Always document which fields are copies.

**V-III.4 Explain the polymorphic and schema-versioning patterns.**
> **Polymorphic** — different shapes live in the same collection because they're queried together (e.g., payments: card, UPI, bank, each with different fields plus a `type` field) so you avoid many collections and unions. **Schema versioning** — a `schemaVersion` field lets old and new document shapes coexist; application code handles both while a background migration upgrades old documents gradually, enabling zero-downtime changes.

**V-III.5 Design a schema for a college placement portal.**
> Collections: `students` (profile, skills[], embedded `address`, resume metadata), `companies`, `jobs` (reference `companyId`, embedded `requirements`, `status`), and `applications` as a **junction** (`studentId`, `jobId`, `status`, `rounds[]` embedded and bounded, timestamps). Indexes: `applications {studentId:1, jobId:1}` unique; `jobs {status:1, deadline:1}`; `students {branch:1, cgpa:-1}`. Reports with aggregation (placement rate per branch). Reasoning: applications are many-to-many with their own data and lifecycle; rounds are bounded and read with the application so they're embedded.

**V-III.6 What are the trade-offs of storing a counter like `enrolledCount` in the parent?**
> **Pro:** O(1) read for "is it full?" and for list views. **Con:** a derived value that can drift if updates fail midway or code forgets to maintain it. Mitigate with **atomic `$inc`** conditioned on capacity, compensating updates on failure, a periodic **reconciliation** job that recomputes from source data, and monitoring for mismatches — exactly what CourseHub does.

---

## 🅳 UNIT IV — Indexing and Aggregation

**V-IV.1 How is an index structured and how does MongoDB choose which one to use?**
> Indexes are **B-trees** of key values (in the WiredTiger engine) with pointers (record identifiers) to documents. For a query, the **query planner** identifies candidate indexes, **trial-runs** the candidate plans briefly in parallel, and picks the one that returns results most efficiently; the winner is stored in the **plan cache** keyed by the query *shape* and re-evaluated when conditions change (e.g., index added/dropped, cache evictions, performance degradation). `hint()` overrides the choice; `explain("allPlansExecution")` shows the competition.

**V-IV.2 What is a multikey index and what are its limits?**
> An index on an **array field** — MongoDB creates an entry per element so queries on elements are indexed. Limits: a compound index can include **at most one array field** per document (indexing two parallel arrays would create a Cartesian explosion), multikey indexes can't be covered for the array field in some cases, and they grow with array size.

**V-IV.3 Why is selectivity/cardinality important when choosing index fields?**
> An index is only helpful if it **narrows** the search. A field with two values (e.g., `isActive`) selects half the collection — poor selectivity on its own. High-cardinality fields (`rollNo`, `email`) are very selective. In compound indexes, lead with equality fields that are **used in most queries**, keeping ESR in mind; low-cardinality fields can still be useful as leading *equality* fields when always present in queries, or when they help sorted reads.

**V-IV.4 Explain the 100 MB memory limit in sorts and groups.**
> Blocking operations — an in-memory `SORT` (no index) or `$group` — have a per-stage RAM limit (**100 MB** historically). Beyond that the operation fails unless it can **spill to disk** (`allowDiskUse`; newer versions enable disk use by default for aggregation, but check your version). The best fix is an **index that provides the sort order** or earlier filtering/projection to reduce data.

**V-IV.5 How can a pipeline use an index?**
> Only the **leading** stages can: a first-stage `$match` (and `$sort`, and certain `$match`+`$sort` combinations, `$geoNear`, `$text`) can use indexes. Once documents pass through a stage like `$group`, `$project` that reshapes them, later stages work on in-memory results with no indexes. This is why the order of stages matters so much.

**V-IV.6 Compare `$lookup` with embedding and with `populate()`.**
> Embedding avoids joins entirely (best read speed, atomic updates, but duplication/size risks). `$lookup` is a **server-side** left join within one aggregation round-trip (needs an index on the foreign field). Mongoose `populate()` runs **additional queries from the application** and merges results client-side — convenient, but more round-trips. For heavy analytics use `$lookup` after reducing data; for simple API responses `populate` is fine; for hot read paths prefer embedding or the extended-reference pattern.

**V-IV.7 What are window functions in MongoDB?**
> `$setWindowFields` (MongoDB 5.0+) computes values over a **window of related documents** — running totals, moving averages, ranks — without collapsing documents like `$group`. Example: cumulative marks per student ordered by exam date.

**V-IV.8 Why might a query not use an index you created?**
> The query doesn't contain the **leading** field of a compound index; types differ (string vs number); an unanchored or case-insensitive regex; the planner estimated a scan is cheaper (low selectivity); the index is **hidden**; the collation differs; or the query uses operators that can't use it (`$where`, `$ne`/`$nin` are usually weak). Always confirm with `explain()` rather than guessing.

---

## 🅴 UNIT V — Integration, Predictive Analytics, AI-driven Optimisation

**V-V.1 Explain connection pooling parameters in a driver.**
> The driver keeps a pool of reusable TCP connections per server: `maxPoolSize` (default 100) caps concurrent connections, `minPoolSize` keeps some warm, `maxIdleTimeMS` closes idle ones, `waitQueueTimeoutMS`/`serverSelectionTimeoutMS` bound waiting. Size the pool from real concurrency — multiplying app instances × pool size must stay under the server's connection capacity — and **never create clients per request**.

**V-V.2 What are retryable writes and how do transactions look in code?**
> **Retryable writes** let the driver automatically retry certain writes **once** after transient network errors or a primary election, safely, using transaction numbers so a write isn't applied twice. **Transactions** group multiple operations atomically through a **session**:
```js
const session = client.startSession();
try {
  await session.withTransaction(async () => {
    await accounts.updateOne({ _id: A }, { $inc: { balance: -100 } }, { session });
    await accounts.updateOne({ _id: B }, { $inc: { balance: +100 } }, { session });
  });
} finally { await session.endSession(); }
```
> They require a replica set (or sharded cluster), should be short, and are unnecessary when a single-document atomic update can do the job.

**V-V.3 What are change streams?**
> A real-time feed of data changes (insert/update/delete/replace) on a collection, database or cluster, built on the oplog and available on replica sets/sharded clusters. Use `collection.watch()` to react to events — invalidate caches, trigger notifications, sync a search index, feed analytics. They're resumable with a **resume token**, so a consumer that restarts can continue where it stopped.

**V-V.4 How would you build a prediction system on MongoDB data end-to-end?**
> (1) Define the question and label (e.g., "will fail the course"). (2) Build features with an **aggregation pipeline** inside MongoDB (attendance %, average internal marks). (3) Load into pandas; handle missing values. (4) **Split** train/test *before* any fitting; avoid **leakage**. (5) Train a `Pipeline` (imputer/scaler + model), cross-validate, evaluate with metrics suited to class balance (precision/recall/F1/ROC-AUC). (6) Save the model (joblib) and **log a `model_runs` document** (data version, metrics, params, date). (7) Write predictions back (`bulk_write`) or serve on demand. (8) Monitor for **drift** and retrain; review fairness and privacy.

**V-V.5 What is data leakage? Give a MongoDB example.**
> Leakage is when information that wouldn't be available at prediction time (or from the test set) influences training, making metrics unrealistically good. Example: predicting whether a student will fail by using a feature built from **final exam marks**, or computing a normalisation mean over the **whole** dataset before splitting. Prevent it by splitting first, building features only from data available at the prediction time, and fitting preprocessing inside a Pipeline on the training fold only.

**V-V.6 How does MongoDB support AI workloads?**
> Flexible documents hold raw data, features and predictions together; aggregation builds features near the data; **change streams** trigger real-time scoring; **time-series collections** store sensor/metric streams; and **vector search** (e.g., Atlas Vector Search) stores embeddings and finds semantically similar documents, enabling **RAG** — retrieving relevant documents to ground an LLM's answer. The database is the **data layer**; the models live in Python/ML services.

**V-V.7 What does "AI-driven query optimisation" really mean, and what are its risks?**
> Using analytics, statistics or ML on workload telemetry (profiler/slow logs) to **recommend or automate** tuning — index suggestions, anomaly detection, workload forecasting, even LLM-assisted query writing. Risks: unnecessary or conflicting indexes that slow writes and consume RAM, recommendations based on **stale or unrepresentative** workloads, hallucinated or destructive LLM-generated queries, and opaque automation that is hard to audit. The professional practice is **recommend → validate with `explain()` / staging / hidden index → human approval → monitor**.

**V-V.8 How do you secure LLM-generated MongoDB queries?**
> Treat them as **untrusted input**: restrict to a **read-only** database user; **allow-list** aggregation stages and operators (block `$out`, `$merge`, `$where`, `$function`); force a `$limit` and `maxTimeMS`; reject plans that show `COLLSCAN` on large collections using `explain`; show the generated query to the user for approval; log everything; and guard against prompt injection from document content that gets fed back to the model.

---

## 🅵 UNIT VI — Project, Production, Future

**V-VI.1 Walk me through your mini project.**
> *(Use this skeleton and swap in your own details.)* "**CourseHub** manages students, courses, enrollments and results on **Node/Express + Mongoose + MongoDB**, with a **Python** analytics script. I started from requirements and queries, then modelled `students` and `courses` as separate collections and `enrollments` as a **junction collection** because the relationship has marks and status and is queried from both sides. I embedded small bounded data (address, skills). I derived the **index plan from the query list**, e.g. a unique compound index on `{student, course}` that prevents duplicates and serves 'my courses'. The key challenge was **overbooking under concurrency**; I solved it with a single atomic `findOneAndUpdate` that checks capacity and increments `enrolledCount`, and proved it with a race script. Reports use aggregation pipelines with `$match` first, and I verified the indexes with `explain`. For production I would add authentication, TLS, a replica set, backups, monitoring and automated tests."

**V-VI.2 What are the most important production best practices?**
> **Security in layers** (network restrictions, TLS, authentication, role-based least privilege, input validation, secrets management, auditing); **reliability** (replica set of 3+, `w: "majority"` for critical writes, retryable writes, tested backups with defined RPO/RTO); **performance** (indexes matched to queries, `explain` reviews, projection/limits/pagination, connection pooling, `maxTimeMS`); **operations** (monitoring and few actionable alerts, separate environments, safe migrations, graceful shutdown, health checks, runbooks); and **data design** (bounded arrays, validation, schema versioning).

**V-VI.3 RPO and RTO — and how do backups satisfy them?**
> **RPO** (Recovery Point Objective) is the maximum acceptable **data loss** measured in time; **RTO** (Recovery Time Objective) is the maximum acceptable **downtime**. Periodic snapshots give an RPO equal to the snapshot interval; **continuous/point-in-time backup** (oplog-based) shrinks RPO to seconds-minutes. RTO depends on restore speed, so I run **restore drills** and follow **3-2-1** (3 copies, 2 media, 1 off-site). `mongodump/mongorestore` suit small datasets; filesystem/cloud snapshots or managed continuous backup suit large ones. A backup you have never restored is only a hope.

**V-VI.4 What would you monitor?**
> Availability and replication (member state, **replication lag**, oplog window), resources (CPU, RAM, disk space/IOPS, **WiredTiger cache** pressure), connections (current vs available), operations (op counters, queues), **slow queries** and COLLSCANs, index usage, page faults, error rates and application latency (p95/p99). Alerts should be few, actionable and tied to runbooks (disk > 80%, lag > threshold, connections near limit, primary change).

**V-VI.5 How do you migrate a schema safely?**
> Prefer **additive, backward-compatible** changes. Add a `schemaVersion`, deploy code that reads both shapes, run a **throttled background migration** in batches (idempotent and resumable), validate, then tighten validation (`warn` → `error`) and finally remove the old code path. Take a backup first, rehearse on staging with production-like data, and keep a rollback plan.

**V-VI.6 What are the security basics for a MongoDB deployment?**
> Enable **authentication** and use **least-privilege roles**; never expose the database to the public internet (private networking/IP allow-list); use **TLS** in transit and **encryption at rest**; keep credentials in a secrets manager and rotate them; sanitise inputs against NoSQL injection; hash passwords with a strong algorithm; enable **auditing** where available; patch regularly; and avoid logging sensitive data. A default install with no auth bound to all interfaces is the classic breach.

**V-VI.7 What are AI-augmented databases and AutoML? Give benefits and limitations.**
> *AI-augmented databases* add AI features — semantic search via embeddings and vector indexes, natural-language querying, self-tuning/advisors, anomaly detection — on top of a normal database. *AutoML* automates the repetitive parts of ML: trying multiple algorithms, hyper-parameter search and cross-validated evaluation to deliver a strong baseline model. **Benefits:** speed, accessibility, consistent evaluation, reduced toil. **Limitations:** they don't define the problem, fix poor data, prevent leakage, choose the right business metric or ensure fairness and privacy; they can be costly and opaque. Therefore: **AI assists, humans own validation and decisions.**

**V-VI.8 Where do you see MongoDB (and databases in general) going?**
> Toward **converged platforms** (documents + search + vectors + time-series), **managed and serverless** operation, stronger built-in **security** (e.g., encryption that supports queries), **AI-assisted** development and tuning, and tighter **operational–analytical** integration. The fundamentals — modelling for access patterns, indexing, measuring with `explain`, securing and backing up data — remain the core skills.

---

# 💼 SECTION B — INDUSTRY INTERVIEW ROUND

## B.1 The professional answer framework (use for any design/debug question)

```
1. CLARIFY    ask 2–3 questions: data size? read/write ratio? consistency needs? latency target?
2. MODEL      entities, access patterns, embed/reference decisions (say WHY)
3. INDEX      the indexes those queries need (ESR), plus write-cost awareness
4. SCALE      replica set → read scaling → sharding (shard key reasoning)
5. SAFETY     validation, security, backups, monitoring
6. TRADE-OFFS what you gave up, and what you'd revisit with more data
7. VERIFY     how you'd measure it (explain, load test, metrics)
```
Interviewers reward **structured reasoning and honest trade-offs** more than a "perfect" answer.

## B.2 Design questions (with model answers)

**D1. Design the schema for a chat application.**
> *Clarify:* group chats? message volume? history length? Likely **messages are append-heavy and read by conversation, newest first**. **Model:** `conversations` (participants[] bounded, lastMessage embedded snapshot, updatedAt), `messages` (conversationId, senderId, text, createdAt, readBy[] bounded or a separate receipts collection if groups are big). **Do not** embed all messages in the conversation (unbounded). **Index:** `messages {conversationId:1, createdAt:-1}` (equality + sort), `conversations {participants:1, updatedAt:-1}` (multikey). **Scale:** shard `messages` by hashed or compound `{conversationId, createdAt}` — conversationId gives targeted queries, and the time component prevents a single huge hot chunk; TTL or archival for old messages. **Real-time:** change streams or a pub/sub layer. **Trade-offs:** `lastMessage` snapshot is duplicated for fast inbox lists and must be updated with each message.

**D2. Design an e-commerce product catalogue and orders.**
> Products vary widely in attributes → flexible documents with an `attributes` object or a polymorphic pattern; index on `{category:1, price:1}`, text/Atlas Search for search. **Orders** embed line items (bounded; price **snapshot** at purchase time) and reference customer; store an extended reference for product name. Inventory decrements use an atomic conditional `findOneAndUpdate({sku, stock:{$gte:qty}}, {$inc:{stock:-qty}})`; payments and order creation across collections use a **transaction** if they must succeed together. Reports use aggregation/pre-aggregated summaries.

**D3. Design a logging/metrics (time-series) system.**
> Use a **time-series collection** (or the bucket pattern) with `timeField`, `metaField` (device/service), and a granularity; set a **TTL/retention**; index on meta+time; downsample old data into hourly/daily summaries with `$merge`; shard by `{meta, time}` if needed. Avoid one document per reading in a plain collection at huge scale and avoid unbounded arrays.

**D4. Choose a shard key for an `orders` collection.**
> Candidates: `_id` (monotonic → hot shard ❌), `customerId` (good cardinality, most queries filter by customer → targeted ✔, but a huge customer can become a hot spot), **hashed `customerId`** (even distribution, loses range queries on that key), or **compound `{customerId, createdAt}`** (targeted queries + range for customer history + better distribution). I'd pick the compound key if "orders of a customer" is the dominant query, and confirm with the real query mix. Reasoning matters more than the specific answer.

**D5. MongoDB vs PostgreSQL (with JSONB) — how do you choose?**
> Both can store JSON. PostgreSQL is ideal when the data is **strongly relational**, needs complex joins, strict constraints/foreign keys and mature SQL tooling. MongoDB fits when the **document is the natural unit of access**, schemas evolve quickly, horizontal scaling and built-in replication/sharding are needed, or data is hierarchical. For mixed needs, I'd evaluate team skill, query patterns, consistency requirements and operational model rather than picking by trend — and sometimes use both.

## B.3 Performance and debugging

**P1. "Our API suddenly got slow after a release." What do you do?**
> Establish **what changed** (code, data growth, traffic). Identify the slow operations through the **profiler/slow query log or APM**; capture the exact query shapes. Run `explain("executionStats")`: COLLSCAN? high `docsExamined/nReturned`? in-memory SORT? wrong index chosen? Check **connection saturation** and pool waits, **replication lag** if reading from secondaries, **cache pressure**/page faults, and lock/queue metrics. Fix with the right index (ESR), query rewrite (projection, limit, keyset pagination), or data-model change; **validate on staging**, deploy off-peak, and add an alert so regressions are caught earlier.

**P2. "This aggregation takes 30 seconds." How do you speed it up?**
> `explain` it. Ensure the first `$match`/`$sort` uses an index; push `$match` and `$limit` earlier; `$project` away large fields before `$group`/`$lookup`; index the `$lookup` foreign field and `$lookup` **after** reducing rows; replace `$unwind` + `$group` with `$filter`/`$size` when possible; consider **pre-aggregation** into a summary collection with `$merge`, caching, or running heavy analytics on a secondary/analytics node; set `maxTimeMS`.

**P3. "We have too many indexes." What now?**
> Check `$indexStats` for indexes with no recent use, find redundant ones (an index on `{a}` is redundant if `{a,b}` exists), **hide** candidate indexes to test impact safely, then drop; watch write latency and memory afterwards. Keep indexes tied to documented queries.

**P4. "Writes are slow." Where do you look?**
> Number and size of indexes per write; document growth causing rewrites; write concern (`majority` + journal) and replication lag/network latency; hot shard from a monotonic key; disk IOPS; large multi-document transactions; lock/queue and cache eviction metrics; and the client — unbatched single inserts instead of `insertMany`/`bulkWrite`.

## B.4 Production and reliability

**R1. "The primary crashed at 2 a.m." What happens, and what did you design for?**
> The remaining members detect missing heartbeats and **elect a new primary** (seconds). Drivers rediscover the topology and, with **retryable writes**, retry the in-flight operation; writes acknowledged with `w:"majority"` are not lost. We'd get alerts (primary change, member down), investigate the failed node's logs/hardware, re-sync it, and review capacity. Design choices that made this safe: 3-node replica set across failure zones, majority write concern for critical data, sensible driver timeouts, monitoring/alerts, and rehearsed runbooks.

**R2. "Someone ran `deleteMany({})` on production." Recovery?**
> Stop further damage (restrict the user), assess the scope and time of the incident, then restore from **point-in-time backup** to a separate environment, extract the lost data and merge it back (or switch over if the whole database was affected). Afterwards: enforce **least-privilege roles** (no broad delete rights for humans), require reviews/approvals for destructive operations, enable auditing, and run regular **restore drills**. Replication does **not** protect against this — it replicates the delete.

**R3. "How do you secure a MongoDB deployment on the cloud?"** → see V-VI.6; add private networking/VPC peering, no public IPs, secrets manager, per-service DB users, TLS, encryption at rest with KMS, audit logs, patching and regular access reviews.

## B.5 Behavioural / project questions

**B1. "Tell me about a technical problem you solved in your project."** *(STAR format)*
> **S**ituation: students could overbook the last seat when enrolling simultaneously. **T**ask: guarantee capacity limits without slowing the system. **A**ction: replaced read-then-write logic with a single atomic `findOneAndUpdate` (capacity check inside the filter, `$inc` in the update), added a unique compound index to block duplicates, and a compensation step if the later insert failed; wrote a script firing parallel requests. **R**esult: the stress test never exceeded capacity, and I documented the reasoning and a reconciliation script to detect drift.

**B2. "What would you do differently?"**
> Add authentication/roles and `$jsonSchema` validation from the start, automate tests in CI, and measure with `explain` earlier instead of after the fact. (Honest, specific reflection beats claiming perfection.)

**B3. "How do you keep learning a fast-moving technology?"**
> Read release notes and official docs (not just blogs), build small experiments, verify claims with `explain`/benchmarks, and treat vendor features as options to evaluate against trade-offs.

## B.6 Coding round — "write the query"

Use the 8 students from Section 0.3 and the enrollments from Practical 8.

**C1. Top 2 students per branch.**
```js
db.students.aggregate([
  { $sort: { cgpa: -1 } },
  { $group: { _id: "$branch", top: { $push: { name: "$name", cgpa: "$cgpa" } } } },
  { $project: { top: { $slice: ["$top", 2] } } },
  { $sort: { _id: 1 } }
])
// CSE → Isha 9.4, Diya 9.1 · ECE → Meera 8, Kabir 7.2 · EEE → Tarun · ME → Rohan
```
(Newer versions also offer the `$topN` accumulator; the sort-then-push-then-slice approach works broadly.)

**C2. Find duplicate emails.**
```js
db.users.aggregate([
  { $group: { _id: "$email", n: { $sum: 1 }, ids: { $push: "$_id" } } },
  { $match: { n: { $gt: 1 } } }
])
```

**C3. Students with no enrollments.**
```js
db.students.aggregate([
  { $lookup: { from: "enrollments", localField: "rollNo", foreignField: "rollNo", as: "e" } },
  { $match: { e: { $size: 0 } } },
  { $project: { _id: 0, name: 1 } }
])
// Meera, Rohan, Sana, Tarun, Isha  (those with no enrollment documents)
```
Index `enrollments.rollNo` for this at scale.

**C4. Running total of marks per student ordered by date (window function, 5.0+).**
```js
db.exams.aggregate([
  { $setWindowFields: {
      partitionBy: "$rollNo", sortBy: { date: 1 },
      output: { running: { $sum: "$marks", window: { documents: ["unbounded", "current"] } } }
  } }
])
```

**C5. Update only the matching element of an array.**
```js
db.marksdemo.updateOne(
  { roll: 1 },
  { $set: { "marks.$[m].s": 90 } },
  { arrayFilters: [ { "m.sub": "DBMS" } ] }
)
```

**C6. Paginate (keyset).**
```js
const page1 = db.students.find().sort({ _id: 1 }).limit(10).toArray()
const lastId = page1[page1.length - 1]._id
const page2 = db.students.find({ _id: { $gt: lastId } }).sort({ _id: 1 }).limit(10).toArray()
```

**C7. Atomic inventory decrement.**
```js
const item = db.products.findOneAndUpdate(
  { sku: "PEN-01", stock: { $gte: 3 } },
  { $inc: { stock: -3 } },
  { returnDocument: "after" }
)
// item === null → insufficient stock (no negative inventory, no race)
```

**C8. Count students per city, only cities with ≥ 2, sorted.**
```js
db.students.aggregate([
  { $group: { _id: "$city", n: { $sum: 1 } } },
  { $match: { n: { $gte: 2 } } },
  { $sort: { n: -1, _id: 1 } }
])
// Bengaluru 2 · Chennai 2 · Hyderabad 2 · Pune 2
```

---

# 🧾 SECTION C — LAB RECORD TEMPLATE (copy per practical)

```
EXPERIMENT NO.   :  __          DATE : __/__/____
TITLE            :  (from the syllabus list)
AIM              :  To ...
REQUIREMENTS     :  MongoDB __ (Community/Atlas/Docker), mongosh __, Node __ / Python __, Compass (optional)
THEORY           :  definition · diagram · key terms (half page)
PROCEDURE        :  1. ... 2. ... 3. ...
COMMANDS / CODE  :  (paste, with comments)
OUTPUT           :  (screenshots / pasted output)
OBSERVATIONS     :  e.g. "docsExamined fell from 100000 to 1"
RESULT           :  Thus ... was successfully executed.
VIVA QUESTIONS   :  3–5 with answers
```
**Screenshot checklist:** server running (`ping`), each command + output, `explain` before/after, Compass view, error cases (validation / duplicate), final directory listing for code.

---

# 🛠️ SECTION D — TROUBLESHOOTING TABLE (lab-day lifesaver)

| Symptom | First check | Fix |
|---|---|---|
| Cannot connect locally | Is `mongod` running? port 27017? | start service/container; check firewall |
| Cannot connect to Atlas | IP allow-list? user? URI? internet/DNS? | add IP; recreate user; URL-encode password |
| `Authentication failed` | username/password/auth DB | re-enter; create user in the right DB |
| `command not found: mongosh` | PATH / not installed | install mongosh, reopen terminal |
| Insert works but data "disappears" | wrong database (`db` ≠ expected) | `db` to check; `use <name>` |
| `find()` returns nothing | wrong collection name, type mismatch, typo, ObjectId vs string | check `show collections`, `$type`, ids |
| `E11000 duplicate key` | unique index/`_id` | change value / use upsert / handle error |
| `Document failed validation` | validator rule | read `errInfo`; fix data or rule |
| `explain` looks different from notes | MongoDB version | find `COLLSCAN`/`IXSCAN`/`SORT` inside nested plan |
| Aggregation returns empty | `$match` field name/type, stage order, `$unwind` dropped docs | test stage by stage (add stages one at a time) |
| Node: `Promise { <pending> }` | missing `await` | add `await` |
| Node: `Cannot use import` | module type | `"type":"module"` |
| Python: `ServerSelectionTimeoutError` | server/URI | `ping`; fix URI; short timeout for fast feedback |
| Python: import bson error | wrong `bson` package | `pip uninstall bson; pip install pymongo` |
| Slow loop inserts | one insert per request | `insertMany` / `insert_many` in batches |

**Debugging mantra:** *Is the server up? Right database? Right collection? Right type? Test one stage/step at a time.*

---

# 🧠 SECTION E — THE FINAL ONE-PAGE CHEAT SHEET

```js
// ---------- NAVIGATION ----------
show dbs; use college; show collections; db.dropDatabase(); db.students.drop()

// ---------- CRUD ----------
db.c.insertOne({..})                     db.c.insertMany([..])
db.c.find(filter, projection).sort({f:-1}).skip(n).limit(n)
db.c.findOne(filter)                     db.c.countDocuments(filter)
db.c.updateOne(filter,{$set:{..}})       db.c.updateMany(..)    db.c.replaceOne(..)
db.c.updateOne(filter, upd, {upsert:true})
db.c.findOneAndUpdate(filter, upd, {returnDocument:"after"})
db.c.deleteOne(filter)                   db.c.deleteMany(filter)

// ---------- OPERATORS ----------
$eq $ne $gt $gte $lt $lte $in $nin  $and $or $nor $not  $exists $type  $regex $expr
$all $elemMatch $size   |   $set $unset $inc $rename $push $addToSet $pull $pop

// ---------- BSON ----------
NumberInt() NumberLong("..") NumberDecimal("..") ISODate("..") new Date() ObjectId() UUID("..")
{ field: { $type: "int" } }   db.c.aggregate([{ $project:{ t:{ $type:"$field" } } }])

// ---------- VALIDATION ----------
db.createCollection("c",{ validator:{ $jsonSchema:{ bsonType:"object", required:[..], properties:{..} } },
                          validationLevel:"strict", validationAction:"error" })
db.runCommand({ collMod:"c", validator:{..}, validationAction:"warn" })
db.c.createIndex({k:1},{unique:true})

// ---------- INDEX + EXPLAIN ----------
db.c.createIndex({a:1,b:-1})   db.c.getIndexes()   db.c.dropIndex("a_1")
db.c.find(q).explain("executionStats")      // COLLSCAN/IXSCAN · docsExamined vs nReturned · SORT?
ESR: Equality → Sort → Range      PREFIX: {a,b,c} serves a · a+b · a+b+c      COVERED: all fields in index, _id:0

// ---------- AGGREGATION ----------
$match → $group → $sort → $project      $unwind  $lookup  $addFields  $bucket  $facet  $count  $merge
db.c.aggregate([ {$match:{..}}, {$group:{_id:"$f", n:{$sum:1}}}, {$sort:{n:-1}} ])
```
```
// ---------- NODE ----------   import {MongoClient} from "mongodb";  one client; await; finally close
// ---------- MONGOOSE ----------   Schema → Model → Document; runValidators:true; unique = index (11000); lean()
// ---------- PYMONGO ----------   MongoClient(uri) lazy → ping; from bson import ObjectId; cursors one-pass; DuplicateKeyError
```

**Ten sentences to remember for any viva**
```
1. NoSQL = flexible models + scale-out, trading some relational guarantees.
2. CAP: during a partition choose consistency or availability; MongoDB leans CP.
3. Document model: embed what's read together; reference what's large/unbounded/shared.
4. Single-document operations are atomic; multi-document transactions exist but cost.
5. Validation = $jsonSchema in DB + validators in app; unique is an INDEX.
6. Index for queries (ESR), prove with explain (docsExamined ≈ nReturned), mind write cost.
7. Aggregation = assembly line; $match first; only leading stages use indexes.
8. One client per app; await everything; env-var secrets; handle 11000.
9. Production = security layers + replica set + tested backups + monitoring.
10. AI assists (vectors, advisors, AutoML) — humans validate and decide.
```

---

# 🏁 PRACTICAL MANUAL COMPLETE

You now hold, in a single file:

```
✔ All 10 syllabus practicals, end to end — Aim · Theory · Procedure · Code · Output · Explanation · Result
✔ Install (local / Docker / Atlas), SQL-vs-NoSQL, CRUD, BSON, modelling, validation,
  indexing + explain, aggregation, Node.js (driver + Mongoose + REST), Python PyMongo
✔ Bonus: AI predictive analytics, profiler-based query optimisation, mini-project commands
✔ 60+ practical-based viva answers + 45 master viva answers across Units I–VI
✔ Industry interview round: design, performance, production, behavioural, coding
✔ Lab-record template, troubleshooting table, final cheat sheet
```

**Last advice, mate:** don't try to memorise this — **type it**. Run every practical once on your own machine, break one thing on purpose (a duplicate key, a failed validation, a missing index) and read the error. The person who has *seen* the error message answers the viva question without blinking. 😤🔥

*— end of the CSE494 Practical Manual —*
