# 🟣 UNIT I — PART 4: RESOURCE PROVISIONING AND OPTIMIZATION TECHNIQUES

> **Covered in this document:** what provisioning means → why it is hard (over vs under provisioning) → types (static, dynamic, on-demand, self-service, advance reservation) → manual vs automated provisioning → provisioning workflow → scaling (vertical vs horizontal) → auto scaling policies (target tracking, step, scheduled, predictive) with a dry run → load balancing algorithms → caching and CDN → capacity planning → optimization techniques (rightsizing, pricing models, spot, storage tiering, serverless, scheduling, tagging, monitoring) → performance vs cost trade-off → worked cost example → Erl's cloud mechanisms → hands-on snippets (AWS CLI, Azure CLI, ASG config) → viva → master sheet.

---

Absolutely mate. 😤🔥 Parts 2 and 3 told you **what services exist**. Part 4 answers:

> **How do we get the right amount of resources, at the right time, at the lowest cost?**

---

# 1️⃣ WHAT IS RESOURCE PROVISIONING?

> **Provisioning** = allocating and preparing cloud resources (VMs, storage, network, databases) so applications can use them.

Hotel analogy 🏨: guests arrive, the hotel allots rooms. Too few rooms → guests turned away. Too many empty rooms → money wasted.

## 1.1 The central problem

```
DEMAND (users, load)  vs  CAPACITY (resources you provisioned)

Under-provisioning:  capacity < demand → slow site, crashes, lost customers
Over-provisioning:   capacity > demand → paying for idle resources
Goal:                capacity ≈ demand at every moment
```

```
Load
 │        ╱╲            ← real demand (spiky)
 │   ╱╲  ╱  ╲   ╱╲
 │  ╱  ╲╱    ╲_╱  ╲
 │ ═══════════════════  ← static capacity (wasteful most of the time)
 │  ┌┐ ┌─┐ ┌┐ ┌─┐       ← elastic capacity following demand
 └────────────────────── time
```

---

# 2️⃣ TYPES OF PROVISIONING ⭐

| Type | How it works | Example | Pros | Cons |
|---|---|---|---|---|
| **Static (advance) provisioning** | Allocate fixed resources up front based on forecast | Reserve 10 servers for a year | Predictable, cheaper if forecast is right | Waste or shortage if forecast wrong |
| **Dynamic provisioning** | Resources added/removed automatically as load changes | Auto Scaling Group | Efficient, elastic | Needs good rules and monitoring |
| **On-demand / self-service** | Consumer requests resources instantly via console/API | Click "Launch instance" | Fast, flexible | Higher unit price |
| **User self-provisioning** | Consumer acquires via a form/portal with no admin approval | Developer spins up a test VM | Agility | Needs governance (budgets, policies) |
| **Reserved / advance reservation** | Commit for 1-3 years for a discount | Reserved Instances, Savings Plans | Big savings | Lock-in |

Erl groups the common ones as **static**, **dynamic**, and **on-demand (self-service)** provisioning.

## 2.1 Provisioning by resource type

```
Compute   → VM / container / function
Storage   → disks, buckets, file shares
Network   → VPC/VNet, subnets, gateways
Database  → RDS, Cosmos DB
Platform  → app service, runtimes
```

---

# 3️⃣ MANUAL vs AUTOMATED PROVISIONING

| | Manual (console clicking) | Automated (CLI / SDK / IaC) |
|---|---|---|
| Speed | Slow for many resources | Fast, repeatable |
| Errors | Human mistakes, config drift | Consistent |
| Documentation | None | The code is the documentation |
| Scale | Hard | Easy |
| Tools | Portal/console | AWS CLI, Azure CLI, Terraform, CloudFormation, Ansible |

💡 This is why **Unit II is Infrastructure as Code**: manual provisioning does not scale.

## 3.1 The provisioning workflow

```
1. Request       → user/pipeline asks for resource (API call)
2. Authenticate  → IAM / RBAC checks permission
3. Validate      → quota limits, policy, budget
4. Select host   → scheduler finds capacity (region/AZ/host)
5. Allocate      → virtualization carves CPU/RAM/disk/network
6. Configure     → image, networking, security groups, user data / bootstrap
7. Start & register → add to load balancer, DNS, monitoring
8. Meter         → billing starts
9. De-provision  → release when done (and billing stops)
```

---

# 4️⃣ SCALING — THE HEART OF DYNAMIC PROVISIONING ⭐⭐⭐

## 4.1 Vertical vs Horizontal

| | Vertical scaling (scale up/down) | Horizontal scaling (scale out/in) |
|---|---|---|
| Meaning | Make one machine bigger/smaller | Add/remove machines |
| Example | t3.small → t3.xlarge | 2 servers → 10 servers |
| Downtime | Often needs restart | Usually none |
| Limit | Hardware ceiling | Practically unlimited |
| Failure impact | Single point of failure | Redundant |
| App needs | Nothing special | Must be **stateless** or share state externally |
| Best for | Databases (traditionally), quick fixes | Web/app tiers, cloud-native design |

```
Vertical:   [small]  →  [ BIG ]
Horizontal: [vm] → [vm][vm][vm][vm]
```

### Stateless idea (important for horizontal scaling)

Don't store user sessions on a single server's memory. Store sessions in Redis/DB so **any** server can serve **any** request.

## 4.2 Auto Scaling policy types (AWS names, Azure has equivalents in VMSS autoscale)

| Policy | How it decides | Example |
|---|---|---|
| **Target tracking** | Keep a metric near a target | Keep average CPU at 50% |
| **Step scaling** | Different actions for different alarm sizes | CPU 70-85% → +1, >85% → +3 |
| **Simple scaling** | One alarm, one action, cooldown | CPU > 70% → +1 instance |
| **Scheduled scaling** | At known times | 9 AM scale to 10, 11 PM scale to 2 |
| **Predictive scaling** | ML forecasts from history | Pre-scale before daily peak |

Key settings in an Auto Scaling Group:

```
Min size       = never go below (availability floor)
Desired size   = current target
Max size       = never go above (cost ceiling)
Cooldown       = wait time before next scaling action (avoid flapping)
Health checks  = replace unhealthy instances automatically
```

## 4.3 DRY RUN: AUTO SCALING IN ACTION

```
Setup: Min=2, Desired=2, Max=6, target CPU = 50%

09:00  traffic normal   → 2 instances at 40% CPU        → no action
12:00  flash sale       → CPU average hits 85%          → CloudWatch alarm fires
12:01  ASG calculates   → need ~ 2 × 85/50 ≈ 3.4 → add instances (rounded up)
12:03  new instances boot, join load balancer (health check passes)
12:05  4 instances at ~43% CPU                          → stable
15:00  traffic drops    → CPU 20%                       → scale in after cooldown
15:10  back to 2 instances (Min)                        → cost drops
```

---

# 5️⃣ LOAD BALANCING TECHNIQUES ⚖️

## 5.1 Purpose

Spread incoming requests over many servers so none is overloaded; detect and skip unhealthy servers.

## 5.2 Algorithms

| Algorithm | Idea | Good when |
|---|---|---|
| **Round robin** | S1, S2, S3, S1, S2, S3… | Servers equal, requests similar |
| **Weighted round robin** | Stronger servers get more | Mixed server sizes |
| **Least connections** | Send to server with fewest active connections | Long-lived/varied requests |
| **Least response time** | Fastest responding server | Latency sensitive |
| **IP hash** | Same client IP → same server | Session stickiness |
| **Sticky sessions** | Cookie keeps user on one server | Legacy stateful apps |

## 5.3 Layers

```
Layer 4 (transport): TCP/UDP only      → AWS NLB, Azure Load Balancer
Layer 7 (application): HTTP aware      → AWS ALB, Azure Application Gateway
                       path routing /api → API servers, /img → image servers
```

Health check example: LB calls `GET /health` every 10 s; fails 3 times → stop routing to that server.

---

# 6️⃣ CACHING AND CDN (PERFORMANCE OPTIMIZATION) 🚀

| Technique | What | Tools |
|---|---|---|
| **CDN** | Copy static content (images, JS, video) to edge locations near users | CloudFront, Azure Front Door/CDN |
| **In-memory cache** | Keep hot data in RAM, skip DB | Redis/Memcached (ElastiCache, Azure Cache for Redis) |
| **Read replicas** | Extra DB copies for reads | RDS read replicas |
| **Database indexing** | Faster queries (Unit IV of Mongo course 🔗) | — |
| **Compression, HTTP caching** | Smaller, reusable responses | — |

Latency math idea: user in Delhi, origin in Virginia ≈ 250 ms RTT; CDN edge in Delhi ≈ 20 ms.

---

# 7️⃣ CAPACITY PLANNING

| Strategy | Meaning |
|---|---|
| **Lead strategy** | Add capacity ahead of expected demand (before sale) |
| **Lag strategy** | Add capacity after demand appears (risk of slowness) |
| **Match strategy** | Add small increments continuously (auto scaling) |

Cloud's gift: you can use **match strategy** because capacity is elastic.

Load testing tools to verify limits before launch: Apache JMeter, k6, Locust.

---

# 8️⃣ OPTIMIZATION TECHNIQUES (THE BIG TOOLBOX) 🧰

Optimization has **three goals**: **performance**, **cost**, **reliability**. We focus on the cost and efficiency side now (Unit VI goes deeper into billing and FinOps).

## 8.1 Rightsizing ⭐

> Choose instance size that fits actual usage.

```
Observed: m5.2xlarge averaging 8% CPU, 20% RAM
Action:   move to m5.large  → roughly 75% less compute cost
Tools:    AWS Compute Optimizer, Azure Advisor, CloudWatch/Azure Monitor metrics
```

Rule of thumb: sustained CPU < 20-30% and RAM low → downsize candidate.

## 8.2 Pricing model optimization ⭐⭐

| Model | Meaning | Discount idea | Best for | Risk |
|---|---|---|---|---|
| **On-Demand** | Pay per second/hour, no commitment | Baseline (0%) | Unpredictable, short, dev/test | Most expensive |
| **Reserved Instances / Reservations** | Commit 1 or 3 years to specific config | Up to roughly 70%+ off | Steady 24x7 workloads | Lock-in |
| **Savings Plans** | Commit to $/hour spend, flexible across instances | Similar to RI, more flexible | Steady but evolving workloads | Over-commit |
| **Spot (AWS) / Spot VMs (Azure)** | Spare capacity at deep discount (up to ~90%); can be reclaimed with short notice | Largest savings | Batch jobs, CI, big data, fault-tolerant | Interruption |
| **Dedicated Hosts** | Whole physical server for you | Higher cost | Compliance, bring-your-own-license | Cost |
| **Free tier / credits** | Limited free usage | — | Learning | Expiry |

### Smart combination (what real companies do)

```
Base load (always on)      → Reserved / Savings Plan
Variable load              → On-Demand with Auto Scaling
Fault-tolerant batch/CI    → Spot
Dev/Test                   → Scheduled shutdown + small sizes
```

## 8.3 Storage optimization

```
Hot data      → S3 Standard / Blob Hot
Warm          → Infrequent Access / Cool
Cold archive  → Glacier / Archive
Lifecycle policy: after 30 days → IA, after 90 → Glacier, after 7 years → delete
Delete unattached EBS volumes, old snapshots, orphaned disks
Choose right disk type (gp3 vs io2; Standard HDD vs Premium SSD)
```

## 8.4 Serverless for spiky workloads

Pay only when code runs. A function invoked 1 million times/month costs very little compared to an always-on VM waiting for rare requests.

## 8.5 Scheduling (turn things off!)

```
Dev/test VMs: run 9 AM - 7 PM on weekdays → 50 of 168 hours/week ≈ 70% saving
Tools: AWS Instance Scheduler, EventBridge + Lambda, Azure Automation / Auto-shutdown
```

## 8.6 Eliminate waste checklist ✅

```
☐ Idle/unused VMs and load balancers
☐ Unattached disks, old snapshots
☐ Unassociated Elastic IPs (charged when idle)
☐ Oversized instances
☐ Dev environments running at night
☐ Data transfer between regions that could be avoided
☐ Missing tags (no one knows who owns the cost)
```

## 8.7 Governance for optimization

| Practice | Why |
|---|---|
| **Tagging** (Project, Owner, Env) | Allocate cost and find owners |
| **Budgets + alerts** | Early warning on overspend |
| **Quotas / policies** | Prevent giant instance types by mistake |
| **Regular reviews** | Optimization is continuous, not one-time |

## 8.8 Architectural optimization

```
Decouple with queues (SQS)         → absorb spikes, smaller fleets
Use managed services               → less ops work
Stateless design                   → easy horizontal scaling
Multi-AZ, not multi-everything     → right level of redundancy
Graviton / ARM instances           → better price-performance for many workloads
Containers                         → pack more workloads per VM (Part 5)
```

---

# 9️⃣ PERFORMANCE vs COST TRADE-OFF

```
More resources → better performance, higher cost
Fewer resources → lower cost, risk of slowness

Sweet spot = enough capacity to meet SLA (e.g., response < 300 ms)
             at minimum cost
```

Define **SLOs/SLAs** first (availability 99.9%, latency target); then optimize cost without breaking them.

Availability math (useful for answers):

```
99%     → ~3.65 days downtime/year
99.9%   → ~8.76 hours/year
99.99%  → ~52.6 minutes/year
99.999% → ~5.26 minutes/year
```

---

# 🔟 WORKED COST EXAMPLE 🧮

Workload: web app needs **4 instances running 24x7 for a month (730 hours)**. Assume a hypothetical price of **$0.10/hour On-Demand** (illustrative only, real prices vary by region and type).

```
On-Demand:       4 × 730 × 0.10 = $292.00 / month

Reserved (assume ~40% cheaper effective rate = $0.06/hr)
                 4 × 730 × 0.06 = $175.20 / month  → saves $116.80

Mixed approach: 2 reserved always-on + 2 on-demand for peaks (peaks 6 hrs/day ≈ 180 hrs)
                 2 × 730 × 0.06 = $87.60
                 2 × 180 × 0.10 = $36.00
                 Total            = $123.60 / month  → saves $168.40 (≈ 58%)
```

Lesson: **commit to the base, flex for the peaks**.

---

# 1️⃣1️⃣ ERL'S CLOUD MECHANISMS (EXAM THEORY) 📖

Your textbook describes technology mechanisms that enable provisioning and scaling:

| Mechanism | What it does |
|---|---|
| **Hypervisor** | Creates and runs virtual servers |
| **Virtual server** | VM that emulates a physical server |
| **Resource replication** | Creates multiple instances of an IT resource (supports scaling) |
| **Automated scaling listener** | Monitors requests and triggers scaling actions |
| **Load balancer** | Distributes workload across multiple resources |
| **Pay-per-use monitor** | Measures usage for billing |
| **Audit monitor** | Records data for compliance |
| **SLA monitor** | Tracks service level performance (availability, response time) |
| **Failover system** | Switches to redundant instance on failure |
| **Resource cluster** | Group of IT resources acting as one |
| **Multi-device broker / Cloud storage device / Cloud usage monitor** | Support and track usage |

Erl scaling architectures to recall: **dynamic horizontal scaling**, **dynamic vertical scaling**, **dynamic relocation** (move to bigger host), **elastic resource capacity**, **service load balancing**, **cloud bursting** (overflow from on-prem to cloud when demand spikes).

---

# 1️⃣2️⃣ HANDS-ON SNIPPETS 💻

## 12.1 AWS: Launch template + Auto Scaling Group (CLI skeleton)

```bash
# Launch template: the "recipe" for each instance
aws ec2 create-launch-template --launch-template-name web-lt \
  --launch-template-data '{
    "ImageId":"ami-xxxxxxxx",
    "InstanceType":"t3.micro",
    "SecurityGroupIds":["sg-xxxxxxxx"],
    "UserData":"'"$(base64 -w0 userdata.sh)"'"
  }'

# Auto Scaling Group across two subnets (two AZs)
aws autoscaling create-auto-scaling-group --auto-scaling-group-name web-asg \
  --launch-template LaunchTemplateName=web-lt,Version='$Latest' \
  --min-size 2 --max-size 6 --desired-capacity 2 \
  --vpc-zone-identifier "subnet-aaa,subnet-bbb" \
  --target-group-arns arn:aws:elasticloadbalancing:...:targetgroup/web-tg/...

# Target tracking: keep CPU near 50%
aws autoscaling put-scaling-policy --auto-scaling-group-name web-asg \
  --policy-name cpu50 --policy-type TargetTrackingScaling \
  --target-tracking-configuration '{
    "PredefinedMetricSpecification":{"PredefinedMetricType":"ASGAverageCPUUtilization"},
    "TargetValue":50.0}'
```

`userdata.sh`:

```bash
#!/bin/bash
dnf install -y nginx          # Amazon Linux 2023 (use apt on Ubuntu)
systemctl enable --now nginx
echo "Hello from $(hostname)" > /usr/share/nginx/html/index.html
```

## 12.2 Azure: VM Scale Set with autoscale

```bash
az group create -n rg-scale -l centralindia

az vmss create -g rg-scale -n vmss-web --image Ubuntu2204 \
  --vm-sku Standard_B1s --instance-count 2 \
  --admin-username azureuser --generate-ssh-keys

az monitor autoscale create -g rg-scale --resource vmss-web \
  --resource-type Microsoft.Compute/virtualMachineScaleSets \
  --name auto-web --min-count 2 --max-count 6 --count 2

az monitor autoscale rule create -g rg-scale --autoscale-name auto-web \
  --condition "Percentage CPU > 70 avg 5m" --scale out 1
az monitor autoscale rule create -g rg-scale --autoscale-name auto-web \
  --condition "Percentage CPU < 30 avg 5m" --scale in 1
```

## 12.3 Find waste with CLI

```bash
# stopped or idle? list unattached EBS volumes
aws ec2 describe-volumes --filters Name=status,Values=available \
  --query "Volumes[].[VolumeId,Size]" --output table
```

---

# 1️⃣3️⃣ COMMON MISTAKES 🚨

| Mistake | Effect | Fix |
|---|---|---|
| Max size not set | Runaway bill | Always set sensible Max |
| No cooldown | Flapping (up-down-up) | Use cooldown/warm-up |
| Stateful app behind ASG | Users lose sessions | Externalize state |
| Reserved capacity for variable load | Paying for unused commitment | Reserve only the base |
| Forgetting to delete lab resources | Bill | Tags + budgets + cleanup |
| Only CPU metric | Memory/IO bottleneck unseen | Use multiple metrics |

---

# 1️⃣4️⃣ VIVA QUESTIONS (PART 4) 🎯

**Q1. What is resource provisioning?** Allocating and configuring cloud resources for applications.

**Q2. Static vs dynamic provisioning?** Static allocates fixed capacity ahead using forecasts; dynamic adjusts automatically with demand.

**Q3. Over- vs under-provisioning?** Over = idle resources and wasted money; under = poor performance and outages.

**Q4. Vertical vs horizontal scaling?** Vertical makes a machine bigger; horizontal adds more machines.

**Q5. Why must apps be stateless for horizontal scaling?** Any instance must handle any request; shared state lives outside the instance.

**Q6. Name auto scaling policy types.** Target tracking, step, simple, scheduled, predictive.

**Q7. Load balancing algorithms?** Round robin, weighted RR, least connections, IP hash, least response time.

**Q8. What is rightsizing?** Matching instance size to actual usage to cut waste.

**Q9. Reserved vs Spot?** Reserved = commitment for steady load with discount; Spot = spare capacity at deep discount but can be interrupted.

**Q10. What is cloud bursting?** Using cloud capacity for overflow when on-premises capacity is exhausted.

**Q11. Role of automated scaling listener?** Monitors requests/load and triggers scaling actions.

**Q12. How can you reduce dev/test cost?** Scheduled shutdowns, smaller sizes, spot instances, budgets.

---

# 🧠 PART 4 MASTER SHEET

```
Goal: capacity ≈ demand at minimum cost
Provisioning: static | dynamic | on-demand/self-service | reserved
Scale: vertical (bigger) vs horizontal (more) → stateless needed
ASG: min / desired / max + cooldown + health check
Policies: target tracking | step | simple | scheduled | predictive
LB: round robin | weighted | least conn | IP hash   (L4 vs L7)
Performance: CDN + cache + read replicas
Cost: rightsize + (On-Demand / Reserved / Savings Plan / Spot) + storage tiers
      + serverless + schedule + delete waste + tags + budgets
Pattern: commit to base load, flex for peaks
Erl: hypervisor, replication, scaling listener, load balancer, pay-per-use monitor, SLA monitor, failover
```

---

## 🫡 PART 4 STATUS

You can now explain how resources are provisioned, how systems scale automatically, and how to cut costs without hurting performance.

**STOP here.** Next up:

> **PART 5 — Containerization + Docker Architecture + Life Cycle 🐳**
