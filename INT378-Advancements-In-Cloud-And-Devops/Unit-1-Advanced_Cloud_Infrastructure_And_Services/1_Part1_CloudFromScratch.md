# 🟢 UNIT I — PART 1: CLOUD COMPUTING FROM ABSOLUTE SCRATCH

> **Covered in this document:** Unit I roadmap → the problem cloud solves (own servers vs rented servers) → what cloud computing really is (NIST definition) → 5 essential characteristics → virtualization and hypervisors → service models (IaaS, PaaS, SaaS, FaaS) → deployment models (public, private, hybrid, community, multi-cloud) → regions, availability zones, edge locations → shared responsibility model → CapEx vs OpEx and pay-as-you-go → Thomas Erl style terminology (for your textbook) → what actually happens when you click "Launch instance" → viva questions → master facts.

---

Absolutely mate. 😤🔥 Unit I of **INT378: Advancements in Cloud and DevOps** starts now.

Same system we always use:

> Absolute scratch → intuition → concept → diagram → commands/code → dry run → practical thinking → exam/viva → STOP.

Your Unit I syllabus is:

> **Advanced Cloud Infrastructure and Services:** Overview of advanced cloud services provided by AWS and Azure, Resource provisioning and optimization techniques, Containerization concepts including Docker Hub, Docker Architecture and Life Cycle, Case studies on scalable cloud solutions from companies like Netflix, Amazon, and Uber.

Practicals attached to this unit:

```
Practical 1 → AWS free-tier, EC2, VPC, Security Groups, SSH, Nginx/Apache, deploy web app
Practical 4 → Docker image, container, ports, push to Docker Hub, pull on another system
```

---

# 🚀 UNIT I — ROADMAP

```
PART 1 — Cloud Computing from Absolute Scratch ⭐   (this file)
PART 2 — AWS Advanced Services 🔥
PART 3 — Azure Services + AWS vs Azure
PART 4 — Resource Provisioning and Optimization Techniques
PART 5 — Containerization + Docker Architecture + Life Cycle 🐳
PART 6 — Docker Hands-on: Dockerfile, Docker Hub, Volumes, Networks, Compose
PART 7 — Case Studies: Netflix, Amazon, Uber
PART 8 — Practical Lab Guide (EC2 + Docker, step by step)

🏆 GRAND CONSOLIDATION — full revision, tables, 30 viva Qs, traps, master sheet
```

---

# 1️⃣ THE PROBLEM — WHY DOES CLOUD EXIST?

## 1.1 Imagine you start a company

You build a shopping website. To run it you need:

```
Servers (computers that run 24x7)
Storage (disks)
Network (internet, routers, firewalls)
Power + cooling + backup power
Security guards, a room, technicians
Software licenses
```

### The old way (on-premises)

You **buy** all of it first.

```
Buy 10 servers  →  ₹50 lakh upfront
Wait weeks for delivery
Hire people to maintain them
```

### Problems with the old way

| Problem | Example |
|---|---|
| Huge upfront cost | Pay before you earn a single rupee |
| Guessing capacity | Buy for 1000 users, 100,000 arrive on sale day → site crashes |
| Wasted capacity | Buy for sale day, 364 other days servers sit idle |
| Slow | Weeks to add a server |
| Maintenance headache | Hardware fails, you fix it |
| Disaster risk | Fire/flood in your server room = data gone |

## 1.2 The cloud idea (electricity analogy ⚡)

Nobody builds their own power plant at home. You plug into the grid, use what you need, and **pay the bill for what you used**.

```
Old IT   =  own power plant
Cloud    =  electricity from the grid
```

Cloud = **renting computing power over the internet, paying only for use.**

Another analogy: **owning a car vs using Uber/Ola.** Owning = buy, insure, maintain, park. Cloud = book when needed, pay per ride.

---

# 2️⃣ WHAT IS CLOUD COMPUTING? (EXAM DEFINITION)

## 2.1 Simple definition

> Cloud computing is the on-demand delivery of IT resources (compute, storage, databases, networking, software) over the internet with pay-as-you-go pricing.

## 2.2 NIST definition (write this in exams ✍️)

> Cloud computing is a model for enabling ubiquitous, convenient, on-demand network access to a shared pool of configurable computing resources (networks, servers, storage, applications, services) that can be rapidly provisioned and released with minimal management effort or service provider interaction.

Keywords to remember: **on-demand, network access, shared pool, rapidly provisioned, minimal management effort.**

## 2.3 What is "the cloud" physically?

Not a magic cloud. It is **giant data centers** full of servers owned by companies like Amazon (AWS), Microsoft (Azure), Google (GCP).

```
You (laptop) ──internet──▶ Provider's data center
                              │
                         thousands of servers
                         virtualized and shared
                         among many customers
```

---

# 3️⃣ FIVE ESSENTIAL CHARACTERISTICS (NIST) ⭐

| # | Characteristic | Meaning | Example |
|---|---|---|---|
| 1 | **On-demand self-service** | Get resources yourself, no human needed | Launch a VM in 2 minutes from a console |
| 2 | **Broad network access** | Available over the network from any device | Access from phone, laptop |
| 3 | **Resource pooling** | Provider's resources shared across many customers (multi-tenancy) | Your VM and someone else's on the same physical server |
| 4 | **Rapid elasticity** | Scale out/in quickly, even automatically | Add 50 servers for sale day, remove after |
| 5 | **Measured service** | Usage is metered and billed | Billed per hour, per GB, per request |

💡 **Memory trick:** **O-B-R-R-M** → On-demand, Broad access, Resource pooling, Rapid elasticity, Measured service.

### Scalability vs Elasticity (classic viva question)

```
Scalability = ability to handle growth (can the system grow?)
Elasticity  = growing AND shrinking automatically with demand
```

---

# 4️⃣ VIRTUALIZATION — THE ENGINE BEHIND CLOUD 🔧

## 4.1 Why virtualization?

A physical server is powerful. Running one app on it wastes 90% of it.

**Virtualization** = splitting one physical machine into many **virtual machines (VMs)**, each acting like a separate computer.

```
WITHOUT virtualization          WITH virtualization
┌──────────────┐                ┌──────────────────────────┐
│  App         │                │ VM1  │ VM2  │ VM3  │ VM4 │
│  OS          │                │ App  │ App  │ App  │ App │
│  Hardware    │                │ OS   │ OS   │ OS   │ OS  │
└──────────────┘                ├──────┴──────┴──────┴─────┤
 1 app = 1 server                │        HYPERVISOR        │
                                 │        Hardware          │
                                 └──────────────────────────┘
```

## 4.2 Hypervisor

Software that creates and runs VMs, sharing CPU, RAM, disk among them.

| Type | Runs on | Examples | Used in |
|---|---|---|---|
| **Type 1 (bare-metal)** | Directly on hardware | VMware ESXi, Xen, KVM, Microsoft Hyper-V | Cloud data centers |
| **Type 2 (hosted)** | On top of an OS | VirtualBox, VMware Workstation | Laptops, learning |

AWS uses its **Nitro** system (KVM-based) for EC2; Azure uses **Hyper-V** based hypervisor.

## 4.3 VM = what you rent

When you "launch an EC2 instance," you are getting a **VM** carved out of a physical server. Containers (Part 5) are the lighter next step.

---

# 5️⃣ SERVICE MODELS — IaaS, PaaS, SaaS ⭐⭐

## 5.1 The pizza analogy 🍕

```
Make at home      → you do everything (on-premises)
Take and bake     → IaaS  (you get ingredients + oven, you cook)
Pizza delivered   → PaaS  (cooked for you, you add drinks/sides)
Dine at restaurant→ SaaS  (everything done, you just eat)
```

## 5.2 The stack view

```
                On-prem   IaaS     PaaS     SaaS
Application      YOU      YOU      YOU      provider
Data             YOU      YOU      YOU      provider
Runtime          YOU      YOU      provider provider
Middleware       YOU      YOU      provider provider
OS               YOU      YOU      provider provider
Virtualization   YOU      provider provider provider
Servers          YOU      provider provider provider
Storage          YOU      provider provider provider
Networking       YOU      provider provider provider
```

## 5.3 Models in detail

| Model | You get | You manage | Examples | Who uses it |
|---|---|---|---|---|
| **IaaS** Infrastructure as a Service | Virtual machines, storage, networks | OS, middleware, app, data | AWS EC2, Azure Virtual Machines, Google Compute Engine | Sysadmins, DevOps |
| **PaaS** Platform as a Service | Ready platform to run code | App + data only | AWS Elastic Beanstalk, Azure App Service, Heroku | Developers |
| **SaaS** Software as a Service | Finished software | Just use it | Gmail, Office 365, Salesforce, Zoom | End users |
| **FaaS** Function as a Service (serverless) | Run a function on events | Only code | AWS Lambda, Azure Functions | Event-driven apps |

⚠️ Common slip: **EC2 is IaaS, Elastic Beanstalk is PaaS, Lambda is FaaS/serverless.**

---

# 6️⃣ DEPLOYMENT MODELS ⭐

| Model | Meaning | Example | Pros | Cons |
|---|---|---|---|---|
| **Public cloud** | Owned by provider, shared by many customers | AWS, Azure, GCP | Cheap, no maintenance, scalable | Less control |
| **Private cloud** | Dedicated to one organization | Bank's own data center with OpenStack/VMware | Control, security, compliance | Costly, you maintain it |
| **Hybrid cloud** | Public + private connected | Sensitive data on-prem, web front on AWS | Flexibility | Complex integration |
| **Community cloud** | Shared by organizations with common concerns | Government/health consortium | Shared cost, shared compliance | Limited availability |
| **Multi-cloud** | Using 2+ public providers | Compute on AWS, analytics on GCP | Avoid vendor lock-in | Skill and tooling overhead |

💡 Hybrid = **public + private**. Multi-cloud = **multiple public**. Do not mix them.

---

# 7️⃣ GLOBAL INFRASTRUCTURE: REGION, AZ, EDGE 🌍

## 7.1 Concepts

```
REGION             = a geographic area (e.g., Mumbai, N. Virginia)
AVAILABILITY ZONE  = one or more isolated data centers inside a region
EDGE LOCATION      = small site near users used for caching (CDN)
```

```
Region: ap-south-1 (Mumbai)
 ├── AZ ap-south-1a  (data center group A)
 ├── AZ ap-south-1b  (data center group B)
 └── AZ ap-south-1c  (data center group C)
        ↑ separate power, cooling, network; low-latency links between them
```

## 7.2 Why it matters

| Goal | How |
|---|---|
| **High availability** | Run app in 2+ AZs; if one fails, other serves |
| **Low latency** | Choose region near your users |
| **Compliance / data residency** | Keep data in a specific country |
| **Disaster recovery** | Replicate to another region |

Azure equivalents: **Region, Availability Zone, Region pair**. (Part 3)

---

# 8️⃣ SHARED RESPONSIBILITY MODEL 🤝 (very important)

> **Provider secures the cloud itself; you secure what you put in the cloud.**

```
AWS / Azure responsibility ("security OF the cloud")
   Physical data centers, hardware, hypervisor, global network

YOUR responsibility ("security IN the cloud")
   Data, OS patches (on IaaS), firewall rules/Security Groups,
   IAM users and permissions, encryption choices, app code
```

Example: if you leave an S3 bucket public or keep SSH open to the whole world, **that is your fault, not AWS's.**

Responsibility shifts: IaaS → you manage more. SaaS → provider manages more.

---

# 9️⃣ CAPEX vs OPEX, PAY-AS-YOU-GO 💰

| | CapEx (old way) | OpEx (cloud) |
|---|---|---|
| Full form | Capital Expenditure | Operational Expenditure |
| Payment | Big upfront purchase | Pay as you consume |
| Risk | High (wrong guess = waste) | Low |
| Example | Buy 10 servers | Rent 10 VMs for 3 hours |

### Quick maths example

```
VM costs ₹5 per hour.
You run a batch job for 4 hours daily  → 4 × 5 = ₹20/day
Same job on a bought server (₹3 lakh) sits idle 20 hrs/day.
```

Pay-as-you-go + elasticity = **you stop paying the moment you stop using.**

### Cloud benefits (list for 5-mark answers)

```
Cost saving • Speed/agility • Global reach • Elasticity • Reliability
• Security tooling • No hardware maintenance • Innovation (AI, big data services)
```

### Cloud challenges

```
Vendor lock-in • Security/misconfiguration • Data privacy/compliance
• Downtime depends on provider/internet • Surprise bills if unmonitored
```

---

# 🔟 TERMINOLOGY FROM YOUR TEXTBOOK (Thomas Erl) 📖

Erl uses specific names. Knowing them helps in theory answers.

| Erl term | Plain meaning |
|---|---|
| **Cloud provider** | Company that owns/offers cloud (AWS) |
| **Cloud consumer** | Organization/person using cloud |
| **Cloud service owner** | Entity that owns a cloud service |
| **On-premise** | Hosted in your own premises (not cloud) |
| **Scaling** | Horizontal (out/in) vs vertical (up/down) |
| **Cloud service** | Any IT resource offered remotely |
| **IT resource** | Physical or virtual IT item (server, VM, software) |

Erl's **business drivers**: capacity planning, cost reduction, organizational agility.
Erl's **technology innovations**: clustering, grid computing, virtualization, capacity planning.
Erl's **risks**: security vulnerabilities, reduced operational governance, limited portability, multi-regional compliance issues.

---

# 1️⃣1️⃣ WHAT ACTUALLY HAPPENS WHEN YOU CLICK "LAUNCH INSTANCE"? (DRY RUN)

```
1. You choose: AMI (OS image), instance type (t2.micro), key pair, security group
2. Request goes to AWS API after authentication (IAM)
3. AWS control plane finds a physical host with free capacity in your AZ
4. Hypervisor (Nitro) carves out vCPU + RAM for your VM
5. Disk (EBS volume) is created from the AMI and attached
6. Virtual network interface gets a private IP (and public IP if enabled)
7. Instance boots, state: pending → running
8. Billing meter starts ⏱️
9. You SSH in using the key pair
```

Total time: roughly a minute. Old way: weeks. That is the cloud advantage in one dry run.

---

# 1️⃣2️⃣ VIVA QUESTIONS (PART 1) 🎯

**Q1. Define cloud computing.** On-demand delivery of shared computing resources over the internet with pay-per-use pricing, rapidly provisioned with minimal management effort.

**Q2. List the five essential characteristics.** On-demand self-service, broad network access, resource pooling, rapid elasticity, measured service.

**Q3. Difference between scalability and elasticity?** Scalability is capacity to grow; elasticity is automatic growing and shrinking as demand changes.

**Q4. IaaS vs PaaS vs SaaS?** IaaS gives infrastructure (EC2), PaaS gives a platform to run code (App Service), SaaS gives finished software (Gmail).

**Q5. What is a hypervisor? Types?** Software that creates VMs. Type 1 bare-metal (ESXi, KVM), Type 2 hosted (VirtualBox).

**Q6. Region vs Availability Zone?** Region = geographic area; AZ = isolated data center(s) inside a region.

**Q7. What is the shared responsibility model?** Provider secures the cloud infrastructure; customer secures their data, configuration, access.

**Q8. Hybrid vs multi-cloud?** Hybrid = public + private; multi-cloud = more than one public provider.

**Q9. CapEx vs OpEx?** Upfront purchase vs pay-as-you-consume.

**Q10. Why is virtualization important for cloud?** It allows many isolated VMs on one physical server, enabling pooling, multi-tenancy, fast provisioning.

---

# 🧠 PART 1 MASTER FACTS

```
Cloud = on-demand, pay-per-use computing over internet
5 traits: On-demand, Broad access, Pooling, Elasticity, Measured
Virtualization + hypervisor = foundation
IaaS (EC2) → PaaS (App Service) → SaaS (Gmail); FaaS (Lambda)
Public | Private | Hybrid | Community | Multi-cloud
Region → AZs → Edge locations
Shared responsibility: provider = OF the cloud, you = IN the cloud
CapEx → OpEx
```

---

## 🫡 PART 1 STATUS

You now know what cloud is, why it exists, how it is built (virtualization), how it is sold (IaaS/PaaS/SaaS), and where it lives (regions/AZs).

**STOP here.** Next up:

> **PART 2 — AWS Advanced Services 🔥** (compute, storage, networking/VPC, databases, serverless, containers, monitoring, Well-Architected Framework)
