# 🏆 UNIT I — GRAND CONSOLIDATION

## Advanced Cloud Infrastructure and Services — From Scratch → End-to-End

> **Covered in this document:** the whole unit in one sentence → the complete story → part-by-part revision (Parts 1-8) → AWS vs Azure vs Docker side-by-side → decision tables → common errors table → security checklist → lab ideas mapped to practicals → 3 real-world scenarios → 30 best viva questions with answers → common traps → what to write in an exam (and what not to) → master sheet → memory maps → how Unit I connects to Units II-VI.

---

# 🚀 THE ENTIRE UNIT I IN ONE SENTENCE

> **Cloud gives on-demand, pay-per-use infrastructure (AWS/Azure) that you provision and optimize dynamically, package applications into portable Docker containers shipped through registries like Docker Hub, and design for scale and failure the way Netflix, Amazon and Uber do.**

---

# 1️⃣ THE COMPLETE UNIT I STORY

```
PROBLEM   → Own servers: costly, slow, guesswork capacity, fragile
IDEA      → Rent computing as a utility (cloud): on-demand, elastic, measured    [Part 1]
ENGINE    → Virtualization + hypervisor; IaaS/PaaS/SaaS; public/private/hybrid     [Part 1]
AWS       → Compute, storage, network (VPC), database, serverless, monitoring      [Part 2]
AZURE     → Same ideas, different names + enterprise/hybrid strengths              [Part 3]
PROVISION → Right resources at right time: scaling, load balancing, rightsizing,
            pricing models, caching                                                [Part 4]
CONTAINER → "Works on my machine" fix: namespaces + cgroups + layers = Docker      [Part 5]
DOCKER    → Dockerfile → build → run → volumes/networks → push to Docker Hub       [Part 6]
PROOF     → Netflix, Amazon, Uber apply all of this at planetary scale             [Part 7]
PRACTICE  → EC2 + VPC + SG + Nginx ; Docker build/push/pull                       [Part 8]
```

---

# 2️⃣ PART-BY-PART REVISION

## Part 1: Cloud from scratch
```
Definition: on-demand network access to shared, rapidly provisioned resources (NIST)
5 traits: On-demand self-service, Broad access, Resource pooling, Rapid elasticity, Measured service
Virtualization: hypervisor Type 1 (bare-metal: KVM, ESXi, Hyper-V) vs Type 2 (hosted: VirtualBox)
Service models: IaaS (EC2/VM) | PaaS (Beanstalk/App Service) | SaaS (Gmail) | FaaS (Lambda/Functions)
Deployment: Public | Private | Hybrid (public+private) | Community | Multi-cloud (many public)
Geography: Region → AZs → Edge locations
Shared responsibility: provider = security OF cloud ; you = security IN cloud
CapEx → OpEx ; Erl: provider, consumer, IT resource, scaling, risks
```

## Part 2: AWS
```
Families: Compute, Storage, DB, Network, Security, Integration, Monitoring, Management
EC2 (AMI, type, key pair, SG, EBS, user data, Elastic IP) ; ASG + ELB ; Lambda ; Beanstalk
S3 object | EBS block | EFS file ; storage classes ; 11 nines durability
VPC: CIDR, subnets (per AZ), route table, IGW, NAT ; SG (stateful, instance) vs NACL (stateless, subnet)
RDS/Aurora, DynamoDB, ElastiCache, Redshift ; ECR/ECS/EKS/Fargate ; SQS/SNS/EventBridge/Kinesis
CloudWatch (metrics) vs CloudTrail (audit) ; IAM ; Well-Architected 6 pillars (O S R P C S)
```

## Part 3: Azure
```
Hierarchy: Tenant > Management Group > Subscription > Resource Group > Resource
VM, VMSS, App Service, Functions, ACI, AKS ; Blob, Managed Disks, Files ; VNet, NSG, App Gateway
Azure SQL, Cosmos DB, Cache for Redis ; Entra ID + RBAC ; Key Vault ; Monitor ; ARM/Bicep ; Arc
AWS ↔ Azure twin mapping ; Azure = enterprise/hybrid/Microsoft stack, AWS = breadth/maturity
```

## Part 4: Provisioning and optimization
```
Provisioning types: static | dynamic | on-demand/self-service | reserved
Over- vs under-provisioning ; workflow: request → auth → validate → allocate → configure → meter
Vertical (bigger) vs horizontal (more, needs stateless) scaling
ASG: min/desired/max, cooldown, health checks ; policies: target tracking, step, simple, scheduled, predictive
Load balancing algorithms: round robin, weighted, least connections, IP hash ; L4 vs L7
Cost: rightsizing, On-Demand/Reserved/Savings Plan/Spot, storage tiers, scheduling, tagging, budgets
Erl mechanisms: hypervisor, replication, automated scaling listener, load balancer, pay-per-use monitor, failover
```

## Part 5: Containers + Docker architecture + lifecycle
```
Container = app + deps sharing host kernel ; VM = full guest OS
Isolation: namespaces (visibility) + cgroups (limits) + union FS (layers)
Architecture: Client → REST → Daemon (dockerd) → containerd → runc ; Registry (Docker Hub)
Image (read-only template) vs Container (running instance + writable layer)
Life cycle: create → start → [pause/unpause] → stop (SIGTERM) / kill (SIGKILL) → rm ; run = pull+create+start
Container lives while PID 1 lives ; persistence only via volumes ; OCI standards ; Kubernetes orchestrates
```

## Part 6: Docker hands-on + Docker Hub
```
Dockerfile: FROM, WORKDIR, COPY, RUN, ENV, EXPOSE, USER, CMD, ENTRYPOINT, HEALTHCHECK
Layer order for cache ; .dockerignore ; bind 0.0.0.0 ; -p host:container
Volumes (named/bind) ; networks (bridge, user-defined DNS) ; Compose ; multi-stage
Docker Hub: account → token → login → tag user/repo:tag → push → pull anywhere ; official images ; rate limits
```

## Part 7: Case studies
```
Netflix: 2008 DB failure → AWS microservices ; control plane (AWS) vs data plane (Open Connect CDN) ; Chaos Monkey
Amazon: SOA/APIs, two-pizza teams, AWS (2006), Dynamo → DynamoDB, SQS decoupling, Prime Day scaling
Uber: monolith → microservices, real-time dispatch, H3 geo-index, Kafka, surge pricing, Jaeger, hybrid/multi-cloud
Patterns: microservices, horizontal scaling, stateless, cache/CDN, async messaging, NoSQL, multi-AZ, CI/CD, observability
```

## Part 8: Practical lab
```
P1: account+MFA+budget → VPC/IGW/route → SG(22 My IP, 80 all) → key pair → launch → SSH → Nginx → page → public IP test → cleanup
P4: install Docker → app → Dockerfile → build → run -p → lifecycle ops → login → tag → push → pull on 2nd system → verify → cleanup
```

---

# 3️⃣ MASTER COMPARISON TABLES

## 3.1 VM vs Container vs Serverless

| | VM (EC2) | Container (Docker) | Serverless (Lambda) |
|---|---|---|---|
| Unit of deployment | Whole OS + app | App + deps | Function |
| Startup | Minutes | Seconds | Milliseconds-seconds |
| You manage | OS, runtime, app | Image, app (host/orchestrator) | Code only |
| Billing | Per second/hour while running | Underlying host/Fargate usage | Per request + duration |
| Best for | Legacy, full control | Microservices, CI/CD | Event-driven, spiky |
| Isolation | Strong | Process-level | Provider-managed |

## 3.2 AWS vs Azure essentials

| AWS | Azure |
|---|---|
| EC2 | Virtual Machines |
| Auto Scaling Group | VM Scale Sets |
| Elastic Beanstalk | App Service |
| Lambda | Functions |
| EKS / ECR | AKS / ACR |
| S3 / EBS / EFS | Blob / Managed Disks / Files |
| VPC / SG+NACL | VNet / NSG |
| ALB / NLB | Application Gateway / Load Balancer |
| RDS / DynamoDB | Azure SQL / Cosmos DB |
| IAM | Entra ID + RBAC |
| CloudWatch / CloudTrail | Monitor / Activity Log |
| CloudFormation | ARM / Bicep |

## 3.3 Pricing models

| Model | Commitment | Discount | Risk | Use |
|---|---|---|---|---|
| On-Demand | None | 0% | Highest price | Unpredictable/short |
| Reserved / Savings Plan | 1-3 yr | Large | Over-commit | Steady base load |
| Spot | None | Largest (up to ~90%) | Interruption | Batch, CI, fault-tolerant |
| Dedicated Host | Optional | — | Cost | Compliance/licensing |

## 3.4 Scaling

| | Vertical | Horizontal |
|---|---|---|
| Action | Bigger machine | More machines |
| Downtime | Often | Rarely |
| Limit | Hardware | Near unlimited |
| Needs | — | Stateless design + LB |

## 3.5 Storage

| | Block (EBS) | File (EFS) | Object (S3) |
|---|---|---|---|
| Access | Attached disk | Shared NFS folder | HTTP API / keys |
| Scope | One AZ, one instance | Many instances | Region-wide, unlimited |
| Use | OS/DB disk | Shared content | Media, backups, static sites |

---

# 4️⃣ DECISION TABLES

## 4.1 Which compute?

| Situation | Choose |
|---|---|
| Need full OS control, legacy app | EC2 / Azure VM |
| Just deploy code, minimal ops | Beanstalk / App Service |
| Event-driven, spiky, short tasks | Lambda / Functions |
| Microservices, portability | Containers (ECS/EKS/AKS, Fargate) |
| Many containers, advanced orchestration | Kubernetes |
| Fault-tolerant batch at low cost | Spot instances |

## 4.2 Which load balancer?

| Need | Choose |
|---|---|
| HTTP routing by path/host, WAF | ALB / Application Gateway |
| Ultra-fast TCP/UDP | NLB / Azure Load Balancer |

## 4.3 Which Docker data option?

| Need | Choose |
|---|---|
| Production data persistence | Named volume |
| Live-edit code in dev | Bind mount |
| Temporary secrets/scratch | tmpfs |

---

# 5️⃣ COMMON ERRORS TABLE 🛠️

| Symptom | Cause | Fix |
|---|---|---|
| EC2 website not opening | SG missing port 80 / no public IP / no IGW route / web server stopped | Check SG, IP, route, `systemctl status nginx` |
| SSH timeout | SG port 22 not allowing your IP | Update to current My IP |
| SSH "Permission denied (publickey)" | Wrong username or key | `ec2-user`/`ubuntu`; correct `.pem` |
| "UNPROTECTED PRIVATE KEY" | Key permissions open | `chmod 400 key.pem` |
| Unexpected AWS bill | Forgotten instances/EIP/NAT/volumes | Budgets + cleanup + tags |
| ASG flapping | No cooldown/warm-up | Configure cooldown |
| Sessions lost after scaling | Stateful app | Externalize session to Redis/DB |
| Docker "daemon not running" | Service stopped | Start Docker service/Desktop |
| Docker permission denied | Not in docker group | `usermod -aG docker` + re-login |
| Container exits immediately | Main process ended | `docker logs`; run foreground process |
| Port published but unreachable | App on 127.0.0.1 | Bind 0.0.0.0 |
| `port already allocated` | Host port busy | Use different host port |
| Push "access denied" | Not logged in / wrong image name | `docker login`; `user/repo:tag` |
| `toomanyrequests` | Hub rate limit | Login / mirror |
| Data lost after container removal | Used writable layer | Use volumes |
| `exec format error` | ARM vs x86 mismatch | Build for target platform |

---

# 6️⃣ SECURITY CHECKLIST 🔐 (PREVIEW OF UNIT V)

```
☐ MFA on root and admin users; no daily root use
☐ Security groups least-open (SSH only from My IP; no 0.0.0.0/0 on 22/3389 in real systems)
☐ Never commit keys/secrets; no secrets in Docker images or user data in plain text
☐ Private subnets for databases; public only for load balancers/web tier
☐ S3 Block Public Access on unless intentionally public
☐ Use IAM roles, not long-lived access keys, on EC2
☐ Official/minimal Docker base images, non-root USER, scan images, pin versions
☐ Tag everything; set budgets and alerts; enable CloudTrail
```

---

# 7️⃣ LAB IDEAS (BEYOND THE SYLLABUS PRACTICALS)

| Lab | Idea |
|---|---|
| A | Launch two EC2 instances in different AZs behind an ALB; stop one and watch traffic continue |
| B | Create an ASG with target-tracking CPU 50%; generate load with `stress` or `ab` and watch scale-out |
| C | Host a static site on S3 + CloudFront |
| D | Write a Dockerfile with multi-stage build and compare image sizes |
| E | `docker compose` with web + Redis visit counter |
| F | Push image to both Docker Hub and AWS ECR |
| G | Rightsizing exercise: read CloudWatch CPU graphs and pick a smaller instance |
| H | Azure twin: VM + NSG + Nginx via `az` CLI, then delete the resource group |

---

# 8️⃣ THREE REAL-WORLD SCENARIOS

## Scenario A: College admission portal crashes on result day
```
Problem: fixed 2 servers; 50x traffic on result day
Fix: ALB + Auto Scaling (scheduled + target tracking), CDN for static pages, cache results,
     read replicas, queue heavy PDF generation, load test before event
Concepts: dynamic provisioning, horizontal scaling, caching, decoupling (Parts 2, 4)
```

## Scenario B: Startup ships "works on my machine" bugs
```
Problem: dev/test/prod differences
Fix: Dockerize app, build once in CI, push to registry, same image everywhere,
     compose for local dev, tags per commit, rollback by image tag
Concepts: containers, Dockerfile, Docker Hub/ECR (Parts 5, 6)
```

## Scenario C: Monthly cloud bill too high
```
Problem: many oversized always-on VMs, orphaned disks
Fix: rightsizing via Compute Optimizer, Savings Plan for steady base, Spot for batch,
     schedule dev shutdown, delete unattached volumes/snapshots, S3 lifecycle rules, tagging + budgets
Concepts: optimization techniques, pricing models (Part 4)
```

---

# 9️⃣ 30 BEST UNIT I VIVA QUESTIONS 🔥 (WITH ANSWERS)

**Cloud basics**

1. **Define cloud computing.** On-demand delivery of shared computing resources over the internet with pay-as-you-go pricing and minimal management effort.
2. **List NIST's five characteristics.** On-demand self-service, broad network access, resource pooling, rapid elasticity, measured service.
3. **IaaS vs PaaS vs SaaS?** Infrastructure (VMs) vs platform to run code vs finished software; control decreases from IaaS to SaaS.
4. **Hybrid vs multi-cloud?** Hybrid mixes public and private; multi-cloud uses multiple public providers.
5. **Shared responsibility model?** Provider secures cloud infrastructure; customer secures data, configuration, identities, OS (on IaaS).
6. **Region vs AZ?** Geographic area vs isolated data center(s) within it.

**AWS and Azure**

7. **What is EC2 and its key components?** Virtual servers: AMI, instance type, key pair, security group, EBS, user data.
8. **Security group vs NACL?** SG: instance, stateful, allow-only. NACL: subnet, stateless, allow+deny.
9. **What makes a subnet public?** Route to an Internet Gateway (plus public IP and SG rule for reachability).
10. **EBS vs EFS vs S3?** Block disk for one instance; shared file system; object storage.
11. **CloudWatch vs CloudTrail?** Performance monitoring vs API audit logging.
12. **Azure resource hierarchy?** Tenant → management group → subscription → resource group → resource.
13. **Azure equivalents of EC2, S3, VPC, Lambda?** Virtual Machines, Blob Storage, VNet, Functions.
14. **When prefer Azure over AWS?** Microsoft-centric enterprises, hybrid needs, license reuse (Hybrid Benefit).

**Provisioning and optimization**

15. **Static vs dynamic provisioning?** Fixed advance allocation vs automatic adjustment to demand.
16. **Vertical vs horizontal scaling?** Bigger machine vs more machines; horizontal needs stateless design.
17. **Name auto scaling policies.** Target tracking, step, simple, scheduled, predictive.
18. **Load balancing algorithms?** Round robin, weighted, least connections, IP hash, least response time.
19. **What is rightsizing?** Matching instance size to real utilization to cut waste.
20. **Reserved vs Spot vs On-Demand?** Commitment discount vs interruptible deep discount vs flexible full price.

**Docker**

21. **Container vs VM?** Shared-kernel, lightweight, fast vs full guest OS, heavier, stronger isolation.
22. **How does Docker isolate containers?** Namespaces, cgroups, layered union filesystem.
23. **Docker architecture?** Client, daemon (dockerd) with containerd/runc, registry, objects (images, containers, networks, volumes).
24. **Image vs container?** Read-only template vs running instance.
25. **Container life cycle?** Created → running → paused → stopped → removed; `run` = create + start.
26. **RUN vs CMD vs ENTRYPOINT?** Build-time command vs default run-time command vs fixed executable.
27. **What is Docker Hub and how do you push an image?** Public registry; login, tag as `user/repo:tag`, `docker push`.

**Case studies**

28. **Netflix architecture in brief?** AWS-hosted microservices (control plane) plus Open Connect CDN (data plane), resilience via Chaos Monkey.
29. **Amazon's scalability lessons?** API-first services, two-pizza teams, purpose-built databases, decoupling with queues, autoscaling.
30. **Uber's key technologies?** Microservices, H3 geospatial indexing, Kafka streaming, surge pricing, ML platform, move toward hybrid/multi-cloud.

---

# 🔟 COMMON UNIT I TRAPS 🚨

```
1. Saying "private cloud = hybrid" (hybrid = public + private)
2. Confusing scalability with elasticity (elasticity = automatic grow AND shrink)
3. Calling Lambda IaaS (it is FaaS/serverless); EC2 is IaaS; Beanstalk is PaaS
4. Saying subnets/AZ rules are identical in Azure (Azure subnets span the region)
5. "Security group is stateless" (SG is stateful; NACL is stateless)
6. Thinking EXPOSE publishes a port (only -p does)
7. Thinking containers have their own kernel (they share the host kernel)
8. Forgetting 0.0.0.0 binding in Dockerized apps
9. Using :latest in production, secrets in images
10. Forgetting cleanup → surprise bills
11. Quoting exact market-share/price numbers confidently (they change)
12. Treating stop and terminate as same (stop keeps disk; terminate deletes instance)
13. Mixing CloudWatch (metrics) and CloudTrail (audit)
14. Saying data persists in a container after rm (needs volumes)
15. Presenting case-study details as exact fact (use "approximately/reportedly")
```

---

# 1️⃣1️⃣ WHAT TO WRITE IN AN EXAM ✍️

| Question type | Structure |
|---|---|
| **Define/Explain (2-3 marks)** | Definition + 2 key points + one example |
| **Differentiate** | Table with 5-6 rows (aspect, A, B) + one-line conclusion |
| **Architecture/Diagram (5-8 marks)** | Neat labeled diagram + component table + working flow + benefits |
| **Docker architecture** | Draw Client → Daemon → containerd/runc → Registry; list objects; explain `docker run` flow |
| **Container life cycle** | State diagram with commands on arrows + short notes (stop vs kill, rm, volumes) |
| **Provisioning/optimization (8-10 marks)** | Types of provisioning → scaling → load balancing → cost techniques → example calculation |
| **Case study (10 marks)** | Template from Part 7: background, challenge, solution, techniques, results, lessons + diagram |
| **Practical/procedure** | Aim, requirements, numbered steps with commands, expected output, cleanup, conclusion |

Writing habits: underline keywords, give one real service name per concept (EC2, S3, VPC, Docker Hub), add one tiny example.

---

# 1️⃣2️⃣ WHAT NOT TO WRITE ❌

```
✘ Long storytelling without technical keywords
✘ Exact prices, market shares, user counts stated as certain facts
✘ Marketing language ("best", "revolutionary") instead of technical reasons
✘ Mixing AWS and Azure names in one answer without saying which is which
✘ Commands with wrong syntax or made-up flags
✘ Skipping diagrams in architecture questions
✘ Claiming containers are "more secure than VMs" (they're lighter, not inherently more isolated)
✘ Presenting one company's choices as the only right way
```

---

# 1️⃣3️⃣ UNIT I MASTER SHEET 🧠🔥

```
CLOUD      On-demand, elastic, measured, pooled, broadly accessible | CapEx→OpEx
MODELS     IaaS EC2 | PaaS App Service/Beanstalk | SaaS Gmail | FaaS Lambda
DEPLOY     Public | Private | Hybrid | Community | Multi-cloud
GLOBAL     Region > AZ > Edge ; multi-AZ = high availability
SECURITY   Shared responsibility ; IAM ; SG (stateful) vs NACL (stateless)
AWS        EC2, S3/EBS/EFS, VPC, RDS/DynamoDB, ELB/ASG, Lambda, ECS/EKS, CloudWatch/CloudTrail
AZURE      VM, Blob, VNet/NSG, SQL/Cosmos, VMSS, Functions, AKS, Entra ID, Monitor, ARM
PROVISION  static | dynamic | on-demand ; scale up/out ; ASG min/desired/max ; LB L4/L7
OPTIMIZE   rightsize, Reserved/Savings/Spot, tiering, schedule, delete waste, tag, budget
CONTAINER  shared kernel ; namespaces+cgroups+layers
DOCKER     Client → dockerd → containerd → runc ; Registry ; image vs container
LIFECYCLE  create → start → pause/unpause → stop/kill → rm
DOCKERFILE FROM WORKDIR COPY RUN ENV EXPOSE USER CMD ; cache order ; 0.0.0.0 ; -p ; volumes
HUB        login → tag user/repo:tag → push → pull → run
CASES      Netflix (Open Connect, Chaos Monkey) | Amazon (two-pizza, Dynamo, SOA) | Uber (H3, Kafka, surge)
LABS       P1 EC2+VPC+SG+SSH+Nginx | P4 Docker build+run+push+pull | ALWAYS CLEAN UP
```

---

# 1️⃣4️⃣ MEMORY MAPS 🗺️

## Cloud map
```
Cloud ─┬─ Traits (O-B-R-R-M)
       ├─ Models (IaaS/PaaS/SaaS/FaaS)
       ├─ Deployment (Pub/Priv/Hybrid/Community/Multi)
       ├─ Geography (Region/AZ/Edge)
       └─ Responsibility (OF vs IN)
```

## Provisioning map
```
Demand ─▶ Monitor ─▶ Scaling policy ─▶ ASG ─▶ Load balancer ─▶ Healthy instances
            │                                              
            └─▶ Cost levers: rightsize • commit • spot • tier • schedule • tag
```

## Docker map
```
Dockerfile ─build▶ Image ─run▶ Container ─▶ (volume, network, port)
                    │ push/pull
                    ▼
                 Registry (Docker Hub / ECR / ACR)
```

## Case-study map
```
Monolith ─▶ Microservices ─▶ Cloud + Containers ─▶ Autoscale + Cache/CDN + Queues ─▶ Observability + CI/CD
```

---

# 1️⃣5️⃣ HOW UNIT I CONNECTS TO THE REST OF INT378 🔗

| Next unit | Link from Unit I |
|---|---|
| **Unit II: IaC (Terraform, CloudFormation)** | Part 4's "automated provisioning" and Part 8's manual console steps become code; the EC2 and VPC you built by hand are declared in templates |
| **Unit III: Agile, CI/CD, Jenkins** | Docker images are the artifact CI builds and pushes; same image moves dev → prod |
| **Unit IV: Monitoring (Prometheus, Grafana)** | CloudWatch idea expands; metrics drive autoscaling; Uber/Netflix observability |
| **Unit V: DevSecOps and Cloud Security** | Shared responsibility, IAM, SG, container security, secrets, image scanning |
| **Unit VI: Cloud-native, serverless, cost** | Lambda, 12-factor (stateless, config in env), pricing models and budgets from Part 4 |

```
Unit I (what + how to run)
   → Unit II (automate it as code)
   → Unit III (build/test/deploy it automatically)
   → Unit IV (watch it)
   → Unit V (secure it)
   → Unit VI (optimize and govern it)
```

---

# 🏆 UNIT I — FINAL TAKEAWAY

```
1. Cloud = rent elastic, metered infrastructure instead of owning it.
2. AWS and Azure offer the same building blocks under different names.
3. Good provisioning = match capacity to demand automatically and cheaply.
4. Containers package apps portably; Docker builds, ships (Hub), and runs them.
5. Giants like Netflix, Amazon and Uber scale with microservices, automation,
   caching, messaging, redundancy, and a culture of designing for failure.
6. You can do it yourself: EC2 + VPC + SG + Nginx, and Docker build → push → pull.
```

---

## 🫡 UNIT I STATUS

✅ Part 1 Cloud from scratch
✅ Part 2 AWS services
✅ Part 3 Azure + comparison
✅ Part 4 Provisioning and optimization
✅ Part 5 Containers + Docker architecture + lifecycle
✅ Part 6 Docker hands-on + Docker Hub
✅ Part 7 Case studies
✅ Part 8 Practical lab guide
✅ Grand Consolidation

## Next up in your syllabus: UNIT II, Infrastructure as Code (IaC): Terraform and AWS CloudFormation 🚀

Send the go-ahead and I'll build Unit II the same way.
