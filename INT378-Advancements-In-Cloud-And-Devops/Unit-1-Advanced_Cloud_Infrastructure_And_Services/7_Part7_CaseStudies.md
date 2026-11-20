# 🌍 UNIT I — PART 7: CASE STUDIES ON SCALABLE CLOUD SOLUTIONS (NETFLIX, AMAZON, UBER)

> **Covered in this document:** how to read and answer a case study → what "scalable" really means (the 8 scalability patterns) → Netflix (monolith to microservices, AWS migration, Open Connect CDN, resilience and Chaos Engineering, containers, data stores) → Amazon (service-oriented architecture, two-pizza teams, birth of AWS, DynamoDB lineage, Prime Day scale) → Uber (monolith to microservices, real-time matching, geospatial indexing, Kafka and data platform, multi-cloud move) → side-by-side comparison → common patterns across all three → mapping case-study ideas to AWS/Azure/Docker services from Parts 2-6 → exam answer templates → viva → master sheet.

---

Absolutely mate. 😤🔥 Parts 1-6 gave tools. Part 7 shows **how real giants combine them** to serve millions of users.

⚠️ **Honesty note for exams:** these companies change their architecture constantly. Learn the **ideas and patterns** (which are stable and widely published in their engineering blogs and talks) rather than exact numbers. Where a figure is approximate, say "approximately" or "reported".

---

# 1️⃣ HOW TO ATTACK ANY CASE STUDY 🧭

Use this 6-part skeleton for any company:

```
1. BACKGROUND  → what the business does
2. CHALLENGE   → why the old setup failed (scale, speed, reliability)
3. SOLUTION    → architecture + cloud services chosen
4. TECHNIQUES  → scaling, provisioning, resilience, caching, data
5. RESULT      → benefits (availability, speed, cost, global reach)
6. LESSONS     → what we learn
```

## 1.1 What "scalable" means (recap with patterns)

| Pattern | Idea | Seen in |
|---|---|---|
| **Horizontal scaling** | Add more machines | All three |
| **Microservices** | Many small independent services | All three |
| **Stateless services** | Any instance serves any request | All three |
| **Caching / CDN** | Serve from memory/edge | Netflix (Open Connect), Amazon (CloudFront) |
| **Data partitioning (sharding)** | Split data across nodes | Amazon (Dynamo), Uber |
| **Asynchronous messaging** | Queues/streams decouple parts | Uber (Kafka), Amazon (SQS) |
| **Multi-region / multi-AZ redundancy** | Survive failures | Netflix, Amazon |
| **Automation + autoscaling** | No manual capacity work | All three |

---

# 2️⃣ CASE STUDY 1: NETFLIX 🎬

## 2.1 Background

Global video streaming service, with a very large subscriber base across more than 190 countries. Traffic is huge, spiky (evenings, new-release nights) and latency sensitive: viewers expect instant playback.

## 2.2 The turning point (the "why")

In 2008 Netflix suffered a major database corruption that disrupted DVD shipping for days. Lesson: a **single vertically-scaled data center with a monolith** is a single point of failure. Netflix decided to move to the cloud and to **design for failure**.

```
BEFORE: own data centers + large monolithic app + big relational DB
AFTER : AWS cloud + hundreds of microservices + distributed data stores
        (migration took years, finishing around 2016 for streaming workloads)
```

## 2.3 Architecture in two halves ⭐

```
                    ┌──────────────────────────────────────────┐
 User device ──1──▶ │  CONTROL PLANE on AWS                    │
 (TV, phone)        │  login, browse, recommendations,         │
                    │  billing, "what to play" decisions       │
                    └──────────────────────────────────────────┘
        └────2──▶ ┌──────────────────────────────────────────┐
                  │  DATA PLANE: Open Connect (Netflix CDN)   │
                  │  video bytes streamed from caching servers │
                  │  placed inside ISPs / near users           │
                  └──────────────────────────────────────────┘
```

| Half | What | Where |
|---|---|---|
| **Control plane** | Everything except the video bits: sign-in, catalog, personalization, API | AWS (EC2, S3, load balancers, many managed services) |
| **Data plane (video delivery)** | Streaming the actual video files | **Open Connect**, Netflix's own CDN with appliances placed at ISPs and internet exchange points |

Why this split is smart: video is the heaviest traffic; serving it from edge boxes near users cuts latency and network cost, while AWS gives elasticity for everything else.

## 2.4 Key techniques

| Technique | Detail |
|---|---|
| **Microservices** | Hundreds of independent services (user profile, recommendations, playback, billing), each owned by a small team, deployed independently |
| **API gateway layer** | Zuul-style edge gateway routes/filters device requests |
| **Service discovery** | Eureka-style registry so services find each other as instances come and go |
| **Elastic autoscaling** | Instances scale with viewing patterns (evening peaks) |
| **Multi-region active-active** | Traffic can be shifted between AWS regions if one has trouble (regional evacuation) |
| **Caching** | EVCache (memcached-based) in front of data stores; pre-computed recommendations |
| **Polyglot data stores** | Apache Cassandra (distributed NoSQL), MySQL, others, chosen per need |
| **Encoding pipeline** | Each title is encoded into many bitrates/formats in parallel in the cloud (a burst-compute, elasticity win) |
| **Containers** | Internal container platform (Titus) runs large numbers of containers for batch and service workloads |
| **Continuous delivery** | Spinnaker (open-sourced by Netflix) for multi-cloud automated deployments, canary and red/black style rollouts |
| **Fault tolerance** | Circuit breaker pattern (Hystrix, now in maintenance mode, with ideas carried on in newer libraries), timeouts, fallbacks (show generic list if personalization fails) |

## 2.5 Chaos Engineering 🐒 (famous!)

Netflix **deliberately breaks things in production** to prove resilience.

```
Chaos Monkey    → randomly terminates instances
Simian Army     → family of tools (latency, region failure simulations; Chaos Monkey is the one still widely referenced)
Principle       → "everything fails eventually; build so users don't notice"
```

## 2.6 Results and lessons

```
✔ High availability even when instances/AZs fail
✔ Rapid global expansion without building data centers everywhere
✔ Fast, independent deployments (many per day)
✔ Elastic cost model for encoding/analytics
Lessons: design for failure • microservices need strong automation • split control and data planes •
         test resilience continuously
```

## 2.7 Map to your syllabus 🔗

| Netflix idea | Unit I concept |
|---|---|
| Autoscaling by demand | Dynamic provisioning, ASG (Part 4) |
| Open Connect | CDN, edge locations (Parts 2, 4) |
| Microservices + Titus | Containers (Part 5) |
| Multi-region | Region/AZ (Part 1) |
| Cassandra + EVCache | NoSQL + caching (Part 4) |

---

# 3️⃣ CASE STUDY 2: AMAZON 🛒

## 3.1 Background

World's largest online retailer: millions of products, huge traffic, extreme spikes (Prime Day, Black Friday). Also the creator of AWS.

## 3.2 The challenge

In the early 2000s the retail site was a large tightly-coupled application. As teams grew, every change needed coordination, deployments were slow and scaling one feature meant scaling everything.

## 3.3 The solution: Service-Oriented Architecture (SOA) → microservices ⭐

Amazon restructured into many small services that talk **only through well-defined APIs**. A widely told mandate from that era: teams must expose functionality via service interfaces, no back-door database access.

```
Product page request fans out to many services:
 ┌─ pricing service        ┌─ inventory service
 ├─ recommendation service ├─ reviews service
 ├─ images/media           ├─ ads / promotions
 └─ delivery estimates     └─ ... (dozens of calls assembled into one page)
```

### Two-pizza teams 🍕🍕

Small teams (small enough to be fed by two pizzas) **own** a service end to end ("you build it, you run it"). Result: autonomy, speed, clear accountability.

## 3.4 Birth of AWS

Building internal infrastructure services (compute, storage, databases) taught Amazon how to run infrastructure as APIs. In 2006 it offered them externally: **S3 and EC2 launched**, starting AWS. The retail business then became both a pioneer and a heavy user of the same cloud.

## 3.5 Key techniques

| Technique | Detail |
|---|---|
| **Decoupling via APIs and messaging** | Services call each other through APIs; queues (SQS) and events absorb spikes |
| **Horizontal scaling + autoscaling** | Fleets scale for peak shopping events |
| **Multi-AZ/region redundancy** | Failure of one data center does not take down shopping |
| **Purpose-built databases** | Amazon's "Dynamo" paper (2007) inspired DynamoDB: highly available key-value store, partitioning by key, replication, tunable consistency; they use different data stores for different jobs (relational, key-value, search, cache) |
| **Caching/CDN** | CloudFront and in-memory caches for product pages and static assets |
| **Automation and CI/CD** | Very frequent deployments with automated testing and rollbacks (internal tooling that inspired AWS CodeDeploy/CodePipeline ideas) |
| **Load shedding and graceful degradation** | If some non-critical service (recommendations) is slow, the page still loads without it |
| **Capacity planning for events** | Pre-scaling and load testing before Prime Day, using autoscaling for the unexpected |

## 3.6 Results and lessons

```
✔ Teams ship independently and fast
✔ Elastic capacity for seasonal peaks
✔ New business (AWS) born from internal capabilities
Lessons: API-first decoupling • small autonomous teams • choose the right database per workload •
         build for failure and peaks • turn internal platforms into products
```

## 3.7 Consistency trade-off (nice exam point)

For a shopping cart, availability matters more than perfect immediate consistency: better to always let the customer add an item (**eventual consistency**) than to block. This is the **CAP theorem** thinking behind Dynamo: during a network partition, choose availability.

## 3.8 Map to your syllabus 🔗

| Amazon idea | Unit I concept |
|---|---|
| EC2/S3 launch | IaaS, object storage (Parts 1, 2) |
| SQS/SNS decoupling | Integration services (Part 2) |
| Prime Day autoscaling | ASG + capacity planning (Part 4) |
| DynamoDB | NoSQL managed service (Part 2) |

---

# 4️⃣ CASE STUDY 3: UBER 🚗

## 4.1 Background

Ride-hailing, food delivery and freight across many countries and cities. Workload is **real-time, location-heavy, and highly variable** (rush hours, events, weather, New Year's Eve surge).

## 4.2 The challenge

Early Uber was a monolithic application (often described as a Python monolith with a single database). As cities and features grew:

```
Slow deployments, one bug could bring down everything,
difficult to scale individual features (dispatch vs payments), global complexity
```

## 4.3 The solution: microservices at massive scale

Uber broke the monolith into a very large number of microservices (often described as thousands today). Core domains:

```
Rider app / Driver app → API gateway
  → Dispatch / Matching       (match riders and drivers in real time)
  → Maps and ETA              (routing, travel time estimates)
  → Pricing / Surge           (dynamic pricing from supply & demand)
  → Payments / Fraud
  → Trip management, Notifications
```

## 4.4 Key technical ideas ⭐

| Idea | Explanation |
|---|---|
| **Real-time matching** | Dispatch system continuously matches nearby riders and drivers; low latency essential |
| **Geospatial indexing (H3)** | Uber created **H3**, a hexagonal grid system that divides the map into cells so "find drivers near this rider" and "demand in this area" become fast cell lookups; used for surge pricing zones and analytics |
| **Event streaming (Apache Kafka)** | Trip events, locations and logs flow through Kafka at huge volume, enabling async processing, real-time analytics, and decoupling of services |
| **Sharded/distributed datastores** | Uber built and used sharded storage on MySQL/other systems (and Cassandra in places) to scale data horizontally; later evolved their own datastore layers |
| **Caching** | Heavy use of caches (Redis-style) for hot data such as locations, ETAs |
| **Dynamic (surge) pricing** | Pricing reacts to real-time supply-demand per area (H3 cell) |
| **Machine learning platform (Michelangelo)** | ML for ETA prediction, fraud detection, pricing, recommendations |
| **Containerization and cluster scheduling** | Services run in containers on cluster managers; Uber has moved toward Kubernetes-based infrastructure over time |
| **Observability** | Large-scale metrics, tracing (they created **Jaeger**, a CNCF distributed tracing project), logging for microservices (links to Unit IV) |
| **Resilience / multi-data-center** | Services replicated across data centers/zones so city operations continue during failures |

## 4.5 Cloud strategy

Historically Uber ran mostly on its own data centers. In recent years it has publicly announced moving significant workloads to public cloud providers (Google Cloud and Oracle Cloud were announced as strategic partners around 2022-2023) while continuing to operate hybrid capacity. Exact current split may change: present this as **a shift from on-prem towards hybrid/multi-cloud** for elasticity and efficiency.

## 4.6 Results and lessons

```
✔ Independent team deployments and faster feature rollout
✔ Real-time operation at city and global scale
✔ Elastic handling of surges
Lessons: microservices need mature tooling (tracing, service mesh/discovery, CI/CD) •
         geospatial + streaming are core to real-time systems •
         hybrid/multi-cloud can reduce risk and cost • ML belongs inside the platform
```

## 4.7 Map to your syllabus 🔗

| Uber idea | Unit I concept |
|---|---|
| Surge scaling | Dynamic provisioning/autoscaling (Part 4) |
| Kafka decoupling | Messaging pattern (Part 2) |
| Containers/Kubernetes | Docker, orchestration (Part 5) |
| Hybrid/multi-cloud | Deployment models (Part 1) |

---

# 5️⃣ SIDE-BY-SIDE COMPARISON ⚖️

| Aspect | Netflix | Amazon | Uber |
|---|---|---|---|
| Domain | Video streaming | E-commerce + cloud provider | Mobility / delivery |
| Main scaling pressure | Evening peaks, global video bandwidth | Seasonal events (Prime Day) | Real-time, location based surges |
| Starting point | Monolith + own DC + DB failure (2008) | Large tightly-coupled retail app | Monolith + single DB |
| Architecture | Microservices on AWS + own CDN | SOA / microservices, own infra (AWS) | Microservices, many languages |
| Signature idea | Open Connect CDN, Chaos Monkey | Two-pizza teams, Dynamo, API mandate | H3 geo-index, Kafka streaming, Jaeger |
| Data | Cassandra, MySQL, EVCache | DynamoDB lineage, RDS, many specialized | Sharded MySQL, Cassandra, Kafka, caches |
| Cloud model | Public cloud (AWS) + own CDN | Owns the cloud (AWS) | On-prem → hybrid/multi-cloud |
| Resilience | Chaos engineering, multi-region | Multi-AZ, graceful degradation | Multi-DC replication, retries |
| Delivery | Spinnaker CD | Highly automated CI/CD | CI/CD + containers |

---

# 6️⃣ COMMON PATTERNS ACROSS ALL THREE 🔑

```
1. Break the monolith → microservices (independent scaling + deployment)
2. Horizontal scaling with automation (autoscaling, containers)
3. Stateless services + external data/caches
4. Caching and CDN to cut latency
5. Asynchronous messaging/streaming to decouple and absorb spikes
6. Purpose-built, distributed (NoSQL) data stores with partitioning/replication
7. Redundancy across AZs/regions/data centers; design for failure
8. Strong observability and CI/CD to manage complexity
9. Small autonomous teams (organization mirrors architecture: Conway's Law)
```

### Costs/challenges of microservices (balanced view, impresses examiners)

```
Operational complexity • network latency between services • distributed debugging •
data consistency across services • need for mature DevOps and monitoring
```

---

# 7️⃣ DESIGN YOUR OWN SCALABLE SYSTEM (PRACTICE QUESTION) ✍️

**Q: Design a scalable video-sharing website using cloud services.**

```
DNS:         Route 53 / Azure DNS
CDN:         CloudFront / Front Door for videos and static assets
Web/API:     containers (ECS/EKS/AKS) or EC2/VMSS behind ALB/App Gateway, autoscaling, multi-AZ
Storage:     S3/Blob for uploaded videos; lifecycle rules to cheaper tiers
Processing:  upload event → queue (SQS) → worker fleet (containers/Spot) encodes versions
Database:    RDS/Azure SQL for users; DynamoDB/Cosmos for activity feeds; Redis cache
Monitoring:  CloudWatch/Azure Monitor alarms and dashboards
Cost:        Spot for encoding, Savings Plan for base fleet, tagging + budgets
Resilience:  multi-AZ, health checks, backups, optional second region
```

---

# 8️⃣ EXAM ANSWER TEMPLATE (5-10 MARKS) ✍️

```
Title: Case study of <Company>

1. Introduction (2 lines)
2. Challenges faced before cloud/microservices
3. Architecture adopted (diagram: users → edge/CDN → gateway → microservices → data stores)
4. Cloud/scalability techniques used (bullets: autoscaling, caching, CDN, messaging, multi-AZ, containers)
5. Benefits achieved
6. Conclusion/lessons
```

Always add a small diagram and one real keyword unique to the company (Open Connect / Chaos Monkey for Netflix; two-pizza teams / Dynamo for Amazon; H3 / Kafka / surge for Uber).

---

# 9️⃣ VIVA QUESTIONS (PART 7) 🎯

**Q1. Why did Netflix move to AWS?** A 2008 database failure showed the risk of a monolith in its own data center; the cloud offered elasticity, redundancy and faster growth.

**Q2. What is Open Connect?** Netflix's own CDN with caching appliances placed near or inside ISPs to stream video efficiently.

**Q3. What is Chaos Monkey?** Tool that randomly terminates production instances to test resilience.

**Q4. What is a control plane vs data plane in Netflix?** Control plane (AWS) handles sign-in, catalog, recommendations; data plane (Open Connect) delivers video bytes.

**Q5. What is a two-pizza team?** A small team owning a service end to end at Amazon.

**Q6. How did AWS originate?** From Amazon's internal infrastructure expertise; S3 and EC2 launched in 2006.

**Q7. What inspired DynamoDB?** Amazon's Dynamo paper on a highly available, partitioned, replicated key-value store.

**Q8. What is H3 in Uber?** Hexagonal hierarchical geospatial indexing system used for location analytics, matching and surge pricing.

**Q9. Why does Uber use Kafka?** To stream huge volumes of events asynchronously and decouple services.

**Q10. Name challenges of microservices.** Complexity, latency, distributed debugging, consistency, need for strong automation and monitoring.

**Q11. How do these companies handle peak traffic?** Horizontal autoscaling, caching/CDN, queues, pre-scaling, load shedding.

**Q12. Common lesson across the three?** Design for failure, decouple services, automate, and scale horizontally.

---

# 🧠 PART 7 MASTER SHEET

```
NETFLIX → 2008 DB failure → AWS + microservices ; Control plane (AWS) vs Data plane (Open Connect CDN)
          Chaos Monkey, Spinnaker, Eureka/Zuul, Cassandra + EVCache, Titus containers
AMAZON  → monolith → SOA/microservices via APIs ; two-pizza teams ; AWS born 2006 (S3, EC2)
          Dynamo → DynamoDB ; SQS decoupling ; Prime Day scaling ; eventual consistency for cart
UBER    → monolith → thousands of microservices ; real-time dispatch ; H3 geo-index ; Kafka ;
          surge pricing ; Michelangelo ML ; Jaeger tracing ; on-prem → hybrid/multi-cloud
COMMON  → microservices, horizontal scaling, stateless, cache/CDN, async messaging,
          NoSQL partitioning, multi-AZ redundancy, observability, CI/CD
```

---

## 🫡 PART 7 STATUS

You can now write a structured case study for Netflix, Amazon and Uber, and connect each idea back to AWS, Azure and Docker concepts.

**STOP here.** Next up:

> **PART 8 — Practical Lab Guide (EC2 + VPC + Nginx, and Docker build/push/pull), step by step 🧪**
