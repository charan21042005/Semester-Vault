# 🟠 UNIT I — PART 2: AWS ADVANCED SERVICES

> **Covered in this document:** how to think about AWS (200+ services, 8 families) → account, IAM, Free Tier basics → compute (EC2, instance families, AMI, key pairs, user data, Elastic IP, Auto Scaling, ELB, Lambda, Elastic Beanstalk) → storage (S3, EBS, EFS, Glacier, storage classes) → networking (VPC, subnets, IGW, NAT, route tables, Security Group vs NACL, Route 53, CloudFront, Direct Connect) → databases (RDS, Aurora, DynamoDB, ElastiCache, Redshift) → containers (ECS, EKS, Fargate, ECR) → messaging/integration (SQS, SNS, EventBridge, Kinesis) → management/monitoring (CloudWatch, CloudTrail, CloudFormation, Systems Manager) → a 3-tier reference architecture → Well-Architected Framework → AWS CLI examples → viva → master sheet.

---

Absolutely mate. 😤🔥 Part 1 gave us what cloud is. Now we meet the biggest provider.

> AWS (Amazon Web Services) = the world's largest cloud platform, with 200+ services. You do **not** need all of them. You need the **core ~30** and how they fit together.

---

# 1️⃣ HOW TO THINK ABOUT AWS (DON'T MEMORIZE 200 SERVICES)

Every service belongs to a family:

```
COMPUTE      → run code            (EC2, Lambda, ECS, EKS)
STORAGE      → keep data           (S3, EBS, EFS)
DATABASE     → structured data     (RDS, DynamoDB, Aurora)
NETWORKING   → connect + secure    (VPC, ELB, Route 53, CloudFront)
SECURITY     → who can do what     (IAM, KMS, WAF, Shield)
INTEGRATION  → services talk       (SQS, SNS, EventBridge)
MONITORING   → watch + audit       (CloudWatch, CloudTrail)
MANAGEMENT   → automate infra      (CloudFormation, Systems Manager)
```

Kitchen analogy 🍳: compute = stove, storage = fridge/pantry, database = recipe register, networking = doors and hallways, security = locks and guards, monitoring = CCTV.

---

# 2️⃣ ACCOUNT BASICS YOU MUST KNOW

| Item | Meaning |
|---|---|
| **Root user** | The email you signed up with. All-powerful. Lock it with MFA, don't use daily |
| **IAM user / role** | Day-to-day identities with limited permissions (Unit V goes deep) |
| **Region selector** | Top-right of console. Resources are **regional** (EC2, VPC) except a few global ones (IAM, Route 53, CloudFront) |
| **Free Tier** | Limited free usage for new accounts. AWS changed its new-account model in 2025 to a credit-based free plan, so check the current terms on the AWS Free Tier page before starting a lab |
| **Billing alarm / Budgets** | Set one on day 1 to avoid surprise bills |

⚠️ Golden rule: **always terminate/delete lab resources after practicals.**

---

# 3️⃣ COMPUTE SERVICES 💻

## 3.1 Amazon EC2 (Elastic Compute Cloud) — IaaS ⭐⭐⭐

Rentable virtual servers. This is **Practical 1**.

### Key EC2 terms

| Term | Meaning |
|---|---|
| **Instance** | One virtual server |
| **AMI** (Amazon Machine Image) | Template: OS + preinstalled software (Amazon Linux, Ubuntu, Windows) |
| **Instance type** | CPU/RAM/network size, e.g., `t3.micro` |
| **Key pair** | Public/private key to SSH in (`.pem` file). AWS keeps public key, you keep private |
| **Security group** | Virtual firewall for the instance |
| **EBS volume** | The virtual disk |
| **User data** | Script that runs at first boot (auto-install Nginx!) |
| **Elastic IP** | Static public IPv4 that you keep |
| **Tags** | Key-value labels (Name=web-1, Env=dev) for organizing and cost tracking |

### Instance families (name decoding)

```
t3.micro
│ │ └── size (nano < micro < small < medium < large < xlarge ...)
│ └──── generation
└────── family
```

| Family | Optimized for | Example use |
|---|---|---|
| **T** (t3, t4g) | Burstable, general, cheap | Small websites, labs |
| **M** | General purpose balanced | App servers |
| **C** | Compute (CPU heavy) | Batch, encoding, gaming servers |
| **R / X** | Memory | In-memory DB, caches |
| **I / D** | Storage (fast disk) | NoSQL, data warehouses |
| **G / P** | GPU | ML training, graphics |

### Instance lifecycle states

```
pending → running → stopping → stopped → (start again) → running
                  ↘ shutting-down → terminated  (gone forever)
```

Stop = you keep EBS disk, stop paying compute. Terminate = instance deleted.

### Pricing options (preview, detailed in Part 4)

On-Demand • Reserved Instances • Savings Plans • Spot • Dedicated Hosts.

## 3.2 Auto Scaling + Elastic Load Balancing (ELB) ⭐

```
          Users
            │
     ┌──────▼───────┐
     │ Load Balancer│   spreads requests
     └───┬──────┬───┘
         ▼      ▼
      EC2-1   EC2-2  ... EC2-N   ← Auto Scaling Group adds/removes these
```

| Service | Job |
|---|---|
| **ELB** | Distributes traffic. Types: **ALB** (HTTP/HTTPS, Layer 7), **NLB** (TCP/UDP, Layer 4, very fast), **GLB** (3rd-party appliances) |
| **Auto Scaling Group (ASG)** | Keeps N instances healthy; adds/removes by rules |

## 3.3 AWS Lambda — serverless (FaaS)

```
Upload a function → define trigger → AWS runs it only when triggered
Billed per request + execution time (ms)
No servers to manage
```

Triggers: API Gateway, S3 upload, schedule (EventBridge), SQS message. (Practical on Lambda is in Unit VI.)

## 3.4 Elastic Beanstalk — PaaS

Upload your code (Java, Node, Python…), Beanstalk creates EC2, ELB, ASG for you. Developer-friendly.

---

# 4️⃣ STORAGE SERVICES 🗄️

## 4.1 Three kinds of storage (EXAM FAVORITE)

| Type | AWS service | Like | Use |
|---|---|---|---|
| **Block** | **EBS** | Hard disk plugged into one VM | OS disk, databases |
| **File** | **EFS** (Linux), FSx | Shared network folder | Many servers share files |
| **Object** | **S3** | Dropbox/Drive style bucket | Images, videos, backups, static sites, logs |

## 4.2 Amazon S3 (Simple Storage Service) ⭐⭐

```
Bucket (globally unique name)  →  Objects (files) identified by Key
s3://my-notes-bucket/images/cat.png
```

| Feature | Meaning |
|---|---|
| **Durability** | 99.999999999% (11 nines) |
| **Versioning** | Keep old versions of objects |
| **Lifecycle rules** | Auto-move old data to cheaper class or delete |
| **Encryption** | At rest and in transit |
| **Static website hosting** | Serve HTML directly |
| **Block Public Access** | Safety switch against accidental public buckets |

### S3 storage classes (cheaper as access gets rarer)

```
S3 Standard                → frequent access
S3 Intelligent-Tiering     → auto-moves based on usage
S3 Standard-IA / One Zone-IA → infrequent access
S3 Glacier (Instant / Flexible / Deep Archive) → archives, cheapest, slower retrieval
```

## 4.3 EBS essentials

EBS volume lives in **one AZ**, attaches to **one EC2** at a time (generally), supports **snapshots** (backups stored in S3).

---

# 5️⃣ NETWORKING — VPC (MOST IMPORTANT PRACTICAL CONCEPT) 🌐

## 5.1 What is a VPC?

**VPC (Virtual Private Cloud)** = your own private, isolated network inside AWS. You decide IP ranges, subnets, routes, firewalls.

Analogy: AWS = a huge apartment society; **VPC = your own gated compound inside it**.

## 5.2 VPC building blocks

| Block | What it is |
|---|---|
| **CIDR block** | IP range of the VPC, e.g., `10.0.0.0/16` (65,536 IPs) |
| **Subnet** | Slice of VPC in **one AZ**. **Public subnet** has route to Internet Gateway; **private subnet** doesn't |
| **Internet Gateway (IGW)** | Door between VPC and internet |
| **Route table** | Rules: where traffic for which destination goes |
| **NAT Gateway** | Lets private-subnet servers go **out** to internet without being reachable **in** |
| **Security Group (SG)** | Firewall at **instance** level. **Stateful**, allow rules only |
| **Network ACL (NACL)** | Firewall at **subnet** level. **Stateless**, allow and deny rules |
| **Elastic IP** | Static public IP |

## 5.3 Diagram

```
                    Internet
                       │
                 ┌─────▼─────┐
                 │    IGW    │
                 └─────┬─────┘
┌─────────────────── VPC 10.0.0.0/16 ──────────────────────┐
│  Public subnet 10.0.1.0/24 (AZ-a)                         │
│    └── EC2 web server (public IP, SG: allow 22, 80, 443)  │
│                                                            │
│  Private subnet 10.0.2.0/24 (AZ-a)                         │
│    └── Database (no public IP, SG: allow 3306 from web SG)│
│    └── outbound via NAT Gateway                            │
└────────────────────────────────────────────────────────────┘
```

## 5.4 Security Group vs NACL ⭐ (frequent viva)

| | Security Group | NACL |
|---|---|---|
| Level | Instance (ENI) | Subnet |
| State | **Stateful** (reply allowed automatically) | **Stateless** (must allow both directions) |
| Rules | Allow only | Allow + Deny |
| Evaluation | All rules together | In number order |
| Default | Deny all inbound, allow all outbound | Default NACL allows all |

## 5.5 A public subnet is public because…

```
Its route table has:   0.0.0.0/0  →  igw-xxxx
(+ instance has a public IP + SG allows the port)
```
All three must be true for a website to be reachable. This is the #1 reason "my EC2 isn't opening" in the practical.

## 5.6 Other networking services

| Service | Job |
|---|---|
| **Route 53** | DNS (domain → IP), health checks, routing policies |
| **CloudFront** | CDN, caches content at edge locations |
| **Direct Connect** | Dedicated private line from your office to AWS |
| **Site-to-Site VPN** | Encrypted tunnel over internet |
| **VPC Peering / Transit Gateway** | Connect VPCs together |

---

# 6️⃣ DATABASE SERVICES 🛢️

| Service | Type | Notes |
|---|---|---|
| **RDS** | Managed relational (MySQL, PostgreSQL, MariaDB, Oracle, SQL Server) | Backups, patching, Multi-AZ failover handled |
| **Aurora** | AWS-built MySQL/PostgreSQL-compatible, high performance | Auto-growing storage, replicas |
| **DynamoDB** | Serverless key-value / document NoSQL | Millisecond latency at huge scale |
| **ElastiCache** | In-memory (Redis/Memcached) | Caching to speed up apps |
| **Redshift** | Data warehouse | Analytics on big data |
| **DocumentDB** | MongoDB-compatible | Callback to your Unit V! 🔗 |

Pick rule: structured/transactions → RDS/Aurora. Massive scale simple access → DynamoDB. Speed → ElastiCache. Analytics → Redshift.

---

# 7️⃣ CONTAINER SERVICES 🐳 (links to Part 5)

| Service | Meaning |
|---|---|
| **ECR** | Elastic Container Registry, AWS's Docker Hub-like private registry |
| **ECS** | AWS's own container orchestrator |
| **EKS** | Managed Kubernetes |
| **Fargate** | Serverless engine for ECS/EKS: run containers, no servers to manage |

---

# 8️⃣ INTEGRATION AND MESSAGING

| Service | Style | Analogy |
|---|---|---|
| **SQS** | Queue (pull) | Token queue at a counter: one worker takes each message |
| **SNS** | Pub/Sub (push) | Loudspeaker: one message to many subscribers |
| **EventBridge** | Event bus, schedules | "When X happens, trigger Y" |
| **Kinesis** | Real-time data streams | Live stream of logs/clicks |
| **API Gateway** | Front door for APIs | Receptionist routing to Lambda/EC2 |

Why they matter: they **decouple** components so one slow part doesn't crash everything, key to scalable design (Part 7).

---

# 9️⃣ MONITORING, SECURITY AND MANAGEMENT

| Service | Job |
|---|---|
| **CloudWatch** | Metrics, logs, alarms, dashboards (CPU, network…) |
| **CloudTrail** | Audit log: who called which API when |
| **IAM** | Users, roles, policies (permissions) |
| **KMS** | Encryption key management |
| **WAF / Shield** | Web firewall / DDoS protection |
| **CloudFormation** | Infrastructure as Code (Unit II) |
| **Systems Manager** | Patch, run commands on fleets |
| **Cost Explorer / Budgets** | See and control spend (Unit VI) |

**CloudWatch vs CloudTrail:** CloudWatch = *performance* (what is happening); CloudTrail = *accountability* (who did what).

---

# 🔟 REFERENCE ARCHITECTURE: SCALABLE 3-TIER WEB APP

```
 Users → Route 53 (DNS) → CloudFront (CDN, caches static content)
                                │
                        Application Load Balancer  (public subnets, 2 AZs)
                         ┌──────┴───────┐
                       EC2 (AZ-a)     EC2 (AZ-b)     ← Auto Scaling Group
                         └──────┬───────┘
                          ElastiCache (Redis)       (private subnets)
                                │
                          RDS Multi-AZ (primary + standby)
 S3 → images / static files / backups
 CloudWatch → alarms (CPU > 70% → scale out)
```

Why this is "scalable and highly available":

```
Multiple AZs        → survive an AZ failure
Load balancer + ASG → handle traffic growth
Cache + CDN         → reduce load and latency
Managed DB Multi-AZ → automatic failover
```

---

# 1️⃣1️⃣ AWS WELL-ARCHITECTED FRAMEWORK (6 PILLARS) ⭐

| Pillar | Question it asks |
|---|---|
| **Operational Excellence** | Can we run and improve it smoothly? |
| **Security** | Is data and access protected? |
| **Reliability** | Does it recover from failure? |
| **Performance Efficiency** | Are we using resources efficiently as demand changes? |
| **Cost Optimization** | Are we avoiding unnecessary spend? |
| **Sustainability** | Is the environmental impact minimized? |

Mnemonic: **O-S-R-P-C-S**.

---

# 1️⃣2️⃣ AWS CLI QUICK TASTE 💻

```bash
# install & configure once
aws configure                       # asks Access Key, Secret, region (ap-south-1), output json

# who am I?
aws sts get-caller-identity

# list EC2 instances (table view)
aws ec2 describe-instances --query "Reservations[].Instances[].[InstanceId,State.Name,PublicIpAddress]" --output table

# launch one instance (replace IDs with yours)
aws ec2 run-instances --image-id ami-xxxxxxxx --instance-type t3.micro \
    --key-name my-key --security-group-ids sg-xxxxxxxx --subnet-id subnet-xxxxxxxx \
    --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=web-1}]'

# stop / terminate
aws ec2 stop-instances --instance-ids i-0123456789abcdef0
aws ec2 terminate-instances --instance-ids i-0123456789abcdef0

# S3
aws s3 mb s3://my-unique-bucket-123
aws s3 cp index.html s3://my-unique-bucket-123/
aws s3 ls
```

⚠️ Never put Access Keys in code or GitHub (Unit V covers secrets).

---

# 1️⃣3️⃣ SERVICE CHOOSER TABLE (QUICK DECISIONS)

| Need | Pick |
|---|---|
| Full control of a server | EC2 |
| Just upload code | Elastic Beanstalk |
| Run code on events, no servers | Lambda |
| Run Docker containers, no servers | ECS/EKS + Fargate |
| Store images/videos/backups | S3 |
| Disk for a VM | EBS |
| Shared folder for many VMs | EFS |
| SQL database managed | RDS / Aurora |
| Huge scale key-value | DynamoDB |
| Speed up reads | ElastiCache / CloudFront |
| Decouple parts | SQS / SNS |
| Watch metrics | CloudWatch |
| Audit actions | CloudTrail |

---

# 1️⃣4️⃣ VIVA QUESTIONS (PART 2) 🎯

**Q1. What is EC2?** Resizable virtual servers in AWS; an IaaS offering.

**Q2. What is an AMI?** Template containing OS and software used to launch instances.

**Q3. EBS vs S3 vs EFS?** EBS = block disk for one instance; S3 = object storage; EFS = shared file system.

**Q4. What is a VPC?** Logically isolated virtual network in AWS where you control IP ranges, subnets, routing.

**Q5. What makes a subnet public?** Route to an Internet Gateway in its route table.

**Q6. Security group vs NACL?** SG: instance level, stateful, allow-only. NACL: subnet level, stateless, allow+deny.

**Q7. ALB vs NLB?** ALB works at HTTP layer 7 with path/host routing; NLB at TCP layer 4 with very high performance.

**Q8. What is Auto Scaling?** Automatically adding/removing instances based on policies to match demand.

**Q9. Lambda vs EC2?** Lambda: serverless, event-driven, per-ms billing, time-limited; EC2: full servers, you manage OS.

**Q10. CloudWatch vs CloudTrail?** Monitoring metrics/logs vs auditing API activity.

**Q11. Why multi-AZ?** High availability against data center failure.

**Q12. What is a NAT Gateway?** Lets private instances reach the internet outbound without inbound exposure.

---

# 🧠 PART 2 MASTER SHEET

```
AWS families: Compute | Storage | Database | Network | Security | Integration | Monitor | Manage
EC2 = IaaS VM | Lambda = FaaS | Beanstalk = PaaS | ECS/EKS/Fargate = containers
EBS block | EFS file | S3 object (11 nines)
VPC → subnet (per AZ) → route table → IGW | NAT for private outbound
SG stateful allow-only (instance) | NACL stateless allow+deny (subnet)
RDS/Aurora SQL | DynamoDB NoSQL | ElastiCache cache | Redshift warehouse
ALB(L7) NLB(L4) + ASG = scalability
CloudWatch = metrics | CloudTrail = audit
Well-Architected: O S R P C S
```

---

## 🫡 PART 2 STATUS

You can now explain and draw a complete AWS architecture and you are ready for Practical 1 (EC2 + VPC + SG).

**STOP here.** Next up:

> **PART 3 — Azure Services + AWS vs Azure Comparison** ☁️
