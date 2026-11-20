# 🔵 UNIT I — PART 3: AZURE SERVICES + AWS vs AZURE

> **Covered in this document:** what Azure is and how it is organized (tenant → management group → subscription → resource group → resource) → regions, availability zones, region pairs → compute (Virtual Machines, VM Scale Sets, App Service, Azure Functions, Container Instances, AKS) → storage (Blob, Disk, Files, tiers) → networking (VNet, NSG, Load Balancer, Application Gateway, Front Door, DNS) → databases (Azure SQL, Cosmos DB, Azure Database for MySQL/PostgreSQL, Cache for Redis) → identity (Microsoft Entra ID, RBAC) → monitoring (Azure Monitor, Log Analytics) → ARM/Bicep → hybrid cloud (Azure Arc, Azure Stack) → AWS ↔ Azure mapping table → deep comparison → how to choose → Azure CLI examples → viva → master sheet.

---

Absolutely mate. 😤🔥 Part 2 = AWS. Part 3 = the main competitor: **Microsoft Azure**.

The good news: **if you understood AWS, you already understand 80% of Azure.** Same ideas, different names.

---

# 1️⃣ WHAT IS AZURE?

Microsoft's cloud platform. Strong in **enterprises** because of its integration with Windows Server, Active Directory, SQL Server, Office 365 and Microsoft developer tools.

Offers all three models: **IaaS** (Virtual Machines), **PaaS** (App Service), **SaaS** (Microsoft 365), plus **serverless** (Azure Functions).

---

# 2️⃣ HOW AZURE IS ORGANIZED (UNIQUE TO AZURE ⭐)

AWS: Account → Region → resources. Azure has a **hierarchy**:

```
Microsoft Entra tenant (your organization's identity directory)
 └── Management Groups  (group many subscriptions, apply policy)
      └── Subscription  (billing + access boundary)
           └── Resource Group  (logical folder for related resources)
                └── Resources (VM, disk, VNet, database ...)
```

| Level | Meaning |
|---|---|
| **Subscription** | Billing unit. Resources are paid under one |
| **Resource Group (RG)** | Container; every resource must be in exactly one RG. Delete the RG = delete everything inside (great for cleanup after labs) |
| **Resource** | The actual service instance |

💡 **Why RG matters:** manage lifecycle, apply tags, RBAC and cost tracking per project.

---

# 3️⃣ GLOBAL INFRASTRUCTURE

| Concept | Meaning |
|---|---|
| **Region** | Geographic area with data centers (e.g., Central India, South India, West Europe) |
| **Availability Zone** | Physically separate data centers within a region (most regions have 3) |
| **Region pair** | Each region paired with another in the same geography for disaster recovery and sequential updates |
| **Edge locations / PoPs** | CDN and Front Door delivery points |

---

# 4️⃣ COMPUTE SERVICES

| Azure service | What it is | AWS equivalent |
|---|---|---|
| **Virtual Machines** | IaaS VMs (Windows/Linux) | EC2 |
| **VM Scale Sets (VMSS)** | Auto-scaling group of identical VMs | Auto Scaling Group |
| **App Service** | PaaS to host web apps/APIs | Elastic Beanstalk |
| **Azure Functions** | Serverless functions | Lambda |
| **Azure Container Instances (ACI)** | Run a container quickly, no orchestration | Fargate (simple case) |
| **Azure Kubernetes Service (AKS)** | Managed Kubernetes | EKS |
| **Azure Container Apps** | Serverless containers/microservices | App Runner / Fargate |
| **Azure Batch** | Large parallel/HPC jobs | AWS Batch |

### VM naming (similar to EC2 idea)

```
Standard_B1s    → B = burstable (like T family)
Standard_D2s_v5 → D = general purpose
Standard_F...   → compute optimized
Standard_E...   → memory optimized
Standard_N...   → GPU
```

---

# 5️⃣ STORAGE SERVICES

| Azure | Type | AWS equivalent |
|---|---|---|
| **Blob Storage** | Object storage | S3 |
| **Managed Disks** | Block storage for VMs | EBS |
| **Azure Files** | Managed SMB/NFS file shares | EFS / FSx |
| **Queue Storage / Service Bus** | Messaging | SQS |
| **Table Storage** | NoSQL key-attribute | (DynamoDB-like) |

**Blob access tiers:** Hot → Cool → Cold → Archive (cheaper storage, costlier/slower access), same logic as S3 classes.
Storage account is the umbrella: one storage account holds Blob, Files, Queue, Table.

**Redundancy options:** LRS (3 copies in one data center), ZRS (across zones), GRS (copy to paired region), GZRS (zones + paired region).

---

# 6️⃣ NETWORKING

| Azure | Job | AWS equivalent |
|---|---|---|
| **Virtual Network (VNet)** | Private network | VPC |
| **Subnet** | Slice of VNet (not tied to one AZ, unlike AWS) | Subnet |
| **Network Security Group (NSG)** | Allow/deny rules at subnet or NIC | SG + NACL combined idea |
| **Load Balancer** | Layer 4 | NLB |
| **Application Gateway** | Layer 7, with optional WAF | ALB |
| **Azure Front Door** | Global entry, CDN + routing | CloudFront + Global Accelerator |
| **Azure DNS** | DNS hosting | Route 53 |
| **VPN Gateway / ExpressRoute** | Encrypted tunnel / private dedicated line | VPN / Direct Connect |
| **NAT Gateway** | Outbound internet for private subnets | NAT Gateway |
| **VNet Peering** | Connect VNets | VPC Peering |

### AWS vs Azure networking difference ⭐

```
AWS subnet  = lives in ONE availability zone
Azure subnet = spans the region (zones are chosen per VM)
AWS SG      = attached to instance, stateful allow-only
Azure NSG   = attached to subnet/NIC, has allow AND deny rules with priority numbers
```

---

# 7️⃣ DATABASE SERVICES

| Azure | Type | AWS equivalent |
|---|---|---|
| **Azure SQL Database / Managed Instance** | Managed SQL Server | RDS for SQL Server |
| **Azure Database for MySQL / PostgreSQL** | Managed open-source SQL | RDS MySQL/PostgreSQL |
| **Azure Cosmos DB** | Globally distributed multi-model NoSQL (SQL API, MongoDB API, Cassandra, Gremlin…) | DynamoDB + DocumentDB |
| **Azure Cache for Redis** | In-memory cache | ElastiCache |
| **Azure Synapse Analytics** | Data warehouse/analytics | Redshift |

Cosmos DB highlight: single-digit-ms latency, multi-region writes, five consistency levels (Strong, Bounded Staleness, Session, Consistent Prefix, Eventual). Also has a **MongoDB-compatible API**, a link back to Unit V. 🔗

---

# 8️⃣ IDENTITY AND SECURITY

| Azure | Meaning | AWS equivalent |
|---|---|---|
| **Microsoft Entra ID** (formerly Azure Active Directory) | Cloud identity directory for users, groups, apps, SSO, MFA | IAM + IAM Identity Center |
| **Azure RBAC** | Roles assigned at a scope (management group / subscription / RG / resource) | IAM policies/roles |
| **Managed Identity** | Identity for a resource so no passwords in code | IAM role for EC2 |
| **Key Vault** | Secrets, keys, certificates | Secrets Manager + KMS |
| **Microsoft Defender for Cloud** | Security posture and threat protection | Security Hub / GuardDuty |
| **Azure Policy** | Enforce rules (e.g., only allowed regions) | AWS Config / SCP |

Built-in roles to know: **Owner** (full + grant access), **Contributor** (full, can't grant access), **Reader** (view only).

---

# 9️⃣ MONITORING AND MANAGEMENT

| Azure | Job | AWS equivalent |
|---|---|---|
| **Azure Monitor** | Metrics, alerts, autoscale triggers | CloudWatch |
| **Log Analytics** | Query logs with KQL | CloudWatch Logs Insights |
| **Application Insights** | App performance monitoring | X-Ray + CloudWatch |
| **Activity Log** | Who did what on resources | CloudTrail |
| **ARM templates / Bicep** | Infrastructure as Code | CloudFormation |
| **Azure Cost Management** | Spend analysis, budgets | Cost Explorer + Budgets |
| **Azure Advisor** | Recommendations (cost, security, performance) | Trusted Advisor |

**ARM** = Azure Resource Manager, the single management layer. Portal, CLI, PowerShell, Terraform, all go through ARM. **Bicep** = simpler language that compiles to ARM JSON.

---

# 🔟 HYBRID CLOUD — AZURE'S STRONG POINT

| Tool | Purpose |
|---|---|
| **Azure Arc** | Manage on-prem/other-cloud servers and Kubernetes from the Azure portal |
| **Azure Stack (HCI/Hub)** | Run Azure services in your own data center |
| **Hybrid Benefit** | Reuse existing Windows/SQL licenses to cut cost |

---

# 1️⃣1️⃣ FULL MAPPING TABLE (REVISION GOLD) ⭐⭐⭐

| Category | AWS | Azure |
|---|---|---|
| Virtual machines | EC2 | Virtual Machines |
| Auto scaling | Auto Scaling Group | VM Scale Sets |
| PaaS web hosting | Elastic Beanstalk | App Service |
| Serverless | Lambda | Functions |
| Containers (managed K8s) | EKS | AKS |
| Container registry | ECR | Azure Container Registry (ACR) |
| Object storage | S3 | Blob Storage |
| Block storage | EBS | Managed Disks |
| File storage | EFS | Azure Files |
| Virtual network | VPC | VNet |
| Firewall rules | Security Group / NACL | NSG |
| Load balancing L7 | ALB | Application Gateway |
| Load balancing L4 | NLB | Load Balancer |
| DNS | Route 53 | Azure DNS |
| CDN | CloudFront | Front Door / Azure CDN |
| Relational DB | RDS / Aurora | Azure SQL / Database for MySQL, PostgreSQL |
| NoSQL | DynamoDB | Cosmos DB |
| Cache | ElastiCache | Cache for Redis |
| Data warehouse | Redshift | Synapse |
| Queue | SQS | Queue Storage / Service Bus |
| Pub/Sub | SNS | Event Grid / Service Bus Topics |
| Identity | IAM | Entra ID + RBAC |
| Secrets | Secrets Manager / KMS | Key Vault |
| Monitoring | CloudWatch | Azure Monitor |
| Audit | CloudTrail | Activity Log |
| IaC | CloudFormation | ARM / Bicep |
| Cost tools | Cost Explorer, Budgets | Cost Management |

---

# 1️⃣2️⃣ AWS vs AZURE — DEEP COMPARISON

| Aspect | AWS | Azure |
|---|---|---|
| Launched | 2006 | 2010 |
| Origin story | Amazon's own e-commerce infra turned into a service | Microsoft enterprise software moving to cloud |
| Market position | Largest market share, widest service catalog | Second largest, fast growing |
| Strength | Maturity, breadth, startups, global reach | Enterprise integration, hybrid, Microsoft stack |
| Best for | Cloud-native apps, startups, huge scale | Windows/.NET/SQL Server shops, Microsoft 365 customers |
| Hierarchy | Account / Organization | Tenant / Subscription / Resource Group |
| Identity | IAM (separate from corporate directory) | Entra ID (ties with corporate Active Directory) |
| Hybrid | Outposts, Direct Connect | Arc, Azure Stack, Hybrid Benefit |
| Pricing | Per second for many services (EC2 Linux), many purchase options | Per minute/hour, Hybrid Benefit, Reservations |
| Console feel | Many separate services, region-bound | Unified portal, resource-group centric |
| IaC native | CloudFormation | ARM/Bicep |
| Terraform support | Both are first-class | Both are first-class |

⚠️ Don't memorize exact market-share numbers; say "AWS leads, Azure is second, GCP third" and mention figures change yearly.

### How to choose (viva-style answer)

```
Existing Microsoft ecosystem (AD, .NET, SQL Server, M365)   → Azure
Cloud-native startup, widest service choice                 → AWS
Strong hybrid requirement                                   → Azure (Arc/Stack)
Best practice for big firms                                 → often multi-cloud, avoiding lock-in
```

---

# 1️⃣3️⃣ AZURE CLI QUICK TASTE 💻

```bash
az login                                         # sign in via browser
az account show                                  # current subscription

# resource group (folder)
az group create --name rg-demo --location centralindia

# create a Linux VM
az vm create --resource-group rg-demo --name vm-web1 \
   --image Ubuntu2204 --size Standard_B1s \
   --admin-username azureuser --generate-ssh-keys

# open port 80 (adds an NSG rule)
az vm open-port --resource-group rg-demo --name vm-web1 --port 80

# list VMs with IPs
az vm list -d -o table

# CLEANUP: delete everything in one shot
az group delete --name rg-demo --yes --no-wait
```

Mapping idea: `az group delete` = best cleanup trick, equivalent to terminating all resources of a project at once.

---

# 1️⃣4️⃣ AZURE ARCHITECTURE (MIRROR OF THE AWS 3-TIER)

```
Users → Azure Front Door (CDN + WAF)
           │
   Application Gateway (L7)
      ┌────┴─────┐
   VM Scale Set (zones 1,2,3)  or  App Service
      └────┬─────┘
   Azure Cache for Redis
           │
   Azure SQL Database (zone-redundant)  +  Blob Storage
 Azure Monitor → alerts + autoscale rules
```

---

# 1️⃣5️⃣ VIVA QUESTIONS (PART 3) 🎯

**Q1. Hierarchy of Azure?** Tenant → management groups → subscriptions → resource groups → resources.

**Q2. What is a resource group?** Logical container for related resources that share a lifecycle, permissions and tags.

**Q3. AWS EC2 equivalent in Azure?** Azure Virtual Machines.

**Q4. S3 equivalent?** Blob Storage.

**Q5. What is Microsoft Entra ID?** Cloud identity and access service (formerly Azure AD) for users, SSO, MFA.

**Q6. What is Cosmos DB?** Globally distributed multi-model NoSQL database with multiple APIs and tunable consistency.

**Q7. NSG vs AWS Security Group?** NSG applies at subnet/NIC with allow and deny and priority; SG is instance-level, allow-only, stateful.

**Q8. What is ARM?** Azure Resource Manager, the deployment/management layer; templates (ARM/Bicep) are Azure's IaC.

**Q9. What is Azure Arc?** Extends Azure management to on-prem and other clouds.

**Q10. When pick Azure over AWS?** Microsoft-centric enterprises, hybrid needs, license reuse benefit.

**Q11. What is a region pair?** Two regions in the same geography linked for disaster recovery.

**Q12. Azure Functions equivalent?** AWS Lambda.

---

# 🧠 PART 3 MASTER SHEET

```
Azure hierarchy: Tenant > Mgmt Group > Subscription > Resource Group > Resource
VM = EC2 | VMSS = ASG | App Service = Beanstalk | Functions = Lambda | AKS = EKS
Blob = S3 | Managed Disk = EBS | Files = EFS | VNet = VPC | NSG = SG/NACL
App Gateway = ALB | Load Balancer = NLB | Front Door = CDN
Azure SQL = RDS | Cosmos DB = DynamoDB(+DocumentDB) | Entra ID = IAM(+directory)
Key Vault = Secrets Manager+KMS | Monitor = CloudWatch | Activity Log = CloudTrail
ARM/Bicep = CloudFormation | Arc/Stack = hybrid
AWS = breadth/maturity | Azure = enterprise/hybrid
```

---

## 🫡 PART 3 STATUS

You can now translate any AWS service into its Azure twin, and justify choosing one over the other.

**STOP here.** Next up:

> **PART 4 — Resource Provisioning and Optimization Techniques** ⚙️ (types of provisioning, auto-scaling, load balancing, rightsizing, pricing models, Erl's mechanisms)
