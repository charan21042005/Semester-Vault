# 🟢 UNIT II — PART 1: INFRASTRUCTURE AS CODE (IaC) FROM ABSOLUTE SCRATCH

> **Covered in this document:** Unit II roadmap → how Unit I ended (manual console clicking) and why it breaks at scale → what Infrastructure as Code is → the 6 problems IaC solves (snowflake servers, config drift, slow/ unrepeatable builds, no audit trail, human error, no disaster recovery) → significance of IaC in DevOps (CI/CD, GitOps, DORA) → core principles (idempotency, declarative vs imperative, mutable vs immutable, push vs pull, agent vs agentless) → IaC vs configuration management vs provisioning → types and categories of IaC tools → the IaC lifecycle (write → validate → plan → review → apply → monitor → destroy) → state, drift, and desired-state reconciliation → IaC workflow with Git → IaC testing and security preview → benefits and challenges → a "same infrastructure, 3 ways" demo → viva → master sheet.

---

Absolutely mate. 😤🔥 Unit II starts now.

Same exact system that worked for Unit I:

> Absolute scratch → intuition → concept → diagram → code → dry run → practical thinking → exam/viva → STOP.

Your Unit II syllabus is:

> **Infrastructure as Code (IaC):** Introduction to Infrastructure as Code (IaC) and its significance in DevOps, Hands-on usage of Terraform and AWS CloudFormation, Comparison of Terraform and CloudFormation, future trends and emerging tools in IaC.

Practicals attached to this unit:

```
Practical 3 → Design and deploy AWS CloudFormation templates, update stacks, validate changes,
              rollback, compare IaC with CloudFormation and Terraform
Practical 5 → Install Terraform, write IaC for EC2 + S3, init/validate/apply/modify/destroy,
              manage Terraform state files
```

---

# 🚀 UNIT II — ROADMAP

```
PART 1 — IaC from Absolute Scratch ⭐                       (this file)
PART 2 — AWS CloudFormation (theory + templates) 🔥
PART 3 — Terraform from Scratch (HCL, providers, workflow, variables, outputs, state) 🔥
PART 4 — Terraform Advanced (backends, locking, modules, count/for_each, lifecycle, import, workspaces)
PART 5 — Terraform vs CloudFormation (full comparison) + Bicep/ARM/Ansible landscape
PART 6 — Future Trends and Emerging Tools in IaC 🔮 (OpenTofu, Pulumi, CDK, Crossplane, policy as code, AI, GitOps, platform engineering)
PART 7 — Practical Lab Guide (Practical 3 + Practical 5 end to end)

🏆 GRAND CONSOLIDATION — revision, comparison tables, 30 viva Qs, traps, master sheet
```

---

# 1️⃣ WHERE UNIT I LEFT YOU: THE PAIN OF CLICKING 🖱️

In Unit I, Practical 1 had you:

```
Console → create VPC → subnet → IGW → route table → security group → key pair → launch EC2 → install Nginx
```

That is maybe 40-60 clicks. Fine for one server. Now imagine reality:

```
Dev environment   → same 50 resources
Test environment  → same 50 resources
Staging           → same 50 resources
Production        → same 50 resources (but bigger) in 2 regions
Disaster recovery → same 50 resources in a 3rd region
```

That is **250+ resources**, created by humans clicking, from memory, at different times.

## 1.1 What goes wrong with manual (ClickOps) infrastructure

| Problem | What happens | Example |
|---|---|---|
| **Snowflake servers** | Every server is slightly different, nobody knows exactly why | "Prod has port 8080 open, test doesn't" |
| **Configuration drift** | Someone changes a setting by hand later; reality diverges from what you believe | A quick SSH fix on prod never documented |
| **Slow** | Hours/days to rebuild an environment | New project needs a week of setup |
| **Not repeatable** | Cannot reliably recreate the same thing | Disaster: "how did we build this?" |
| **Human error** | Wrong region, wrong CIDR, forgotten checkbox | Port 22 opened to the world by mistake |
| **No audit trail** | Who changed what, when, why? Unknown | Outage after unknown change |
| **No review** | No peer review before changes | One person alters production alone |
| **Disaster recovery pain** | Rebuilding after a failure is manual and error-prone | Region outage → days to recover |
| **Documentation rot** | Wiki says one thing, reality another | Outdated diagrams |

💡 **The core idea:** infrastructure should be treated with the same discipline as application code.

---

# 2️⃣ WHAT IS INFRASTRUCTURE AS CODE? ⭐

## 2.1 Simple definition

> **Infrastructure as Code (IaC)** is the practice of defining, creating and managing infrastructure (servers, networks, databases, storage, permissions) using **machine-readable code/configuration files**, instead of manual processes or interactive tools.

## 2.2 The recipe analogy 🍰

```
Manual infrastructure = a chef cooking from memory, a bit different every day
IaC                   = a written recipe; any chef, any kitchen, same cake every time
                        and you can hand the recipe to anyone, review it, version it, improve it
```

Another analogy: **architect's blueprint vs building a house by guessing.** The blueprint (code) is reviewed, versioned, reusable. Each building (environment) is created from it.

## 2.3 What the code looks like (tiny preview)

Instead of clicking, you write:

```hcl
# Terraform (HCL)
resource "aws_instance" "web" {
  ami           = "ami-0abcdef1234567890"
  instance_type = "t3.micro"
  tags = { Name = "web-server-1" }
}
```

```yaml
# CloudFormation (YAML)
Resources:
  WebServer:
    Type: AWS::EC2::Instance
    Properties:
      ImageId: ami-0abcdef1234567890
      InstanceType: t3.micro
      Tags:
        - Key: Name
          Value: web-server-1
```

Same result: one EC2 instance. But now it is a **text file**: store in Git, review, run again tomorrow, run in another region.

## 2.4 What can IaC manage?

```
Compute      → VMs, auto scaling groups, containers, Kubernetes clusters, Lambda
Network      → VPCs/VNets, subnets, route tables, gateways, load balancers, DNS
Storage      → S3 buckets, disks, file systems
Database     → RDS, DynamoDB, caches
Security     → IAM roles/policies, security groups, KMS keys, secrets
Monitoring   → alarms, dashboards, log groups
Platform/SaaS→ GitHub repos, DNS providers, Datadog monitors, Kubernetes manifests (via providers)
```

---

# 3️⃣ SIGNIFICANCE OF IaC IN DEVOPS ⭐⭐ (SYLLABUS POINT)

## 3.1 Where IaC fits in DevOps

DevOps = Development + Operations working together with automation and fast feedback.

```
         ┌──────── PLAN ──── CODE ──── BUILD ──── TEST ────┐
         │                                                   │
       MONITOR ◀── OPERATE ◀── DEPLOY ◀── RELEASE ◀─────────┘
                       ▲
                       │  IaC lives here: infrastructure is created and changed
                       │  by the same pipeline that ships the application
```

Without IaC, "Ops" remains a manual bottleneck inside an automated pipeline. IaC removes the bottleneck: **infrastructure becomes part of the pipeline.**

## 3.2 Key benefits (list these in exams)

| Benefit | Explanation |
|---|---|
| **Speed and agility** | Environments in minutes, not days |
| **Consistency** | Same code → same infrastructure every time; kills "works in dev, fails in prod" at infra level |
| **Repeatability / scalability** | Create 1 or 100 copies with the same code |
| **Version control** | Infra history in Git: who changed what, when, why; easy rollback |
| **Collaboration and review** | Pull requests and code review for infra changes |
| **Documentation** | The code *is* the up-to-date documentation |
| **Reduced human error** | Automation replaces manual steps |
| **Cost control** | Easily tear down dev/test environments after use; tags and policies in code |
| **Disaster recovery** | Rebuild entire environment from code in a new region |
| **Compliance and security** | Policies and guardrails checked automatically before deployment |
| **Automation with CI/CD** | Pipeline runs plan/apply on every merge |

## 3.3 IaC and the DORA metrics (nice modern touch)

DevOps performance is measured by:

```
Deployment frequency      → IaC helps ship more often
Lead time for changes     → IaC shortens environment setup
Change failure rate       → review + testing lowers failures
Time to restore service   → rebuild from code quickly
```

## 3.4 IaC and GitOps (preview)

> **GitOps:** Git is the single source of truth for desired infrastructure/application state; automated tools make reality match Git.

```
Developer opens Pull Request → review + automated plan → merge → pipeline applies → infra updated
```

---

# 4️⃣ CORE CONCEPTS OF IaC (THE VOCABULARY) 📖

## 4.1 Declarative vs Imperative ⭐⭐⭐

| | **Declarative** ("WHAT") | **Imperative** ("HOW") |
|---|---|---|
| You describe | The desired end state | The exact steps to execute |
| Tool figures out | The steps | Nothing: you wrote them |
| Example | "I want 3 servers" | "Create server 1, then create server 2, then create server 3" |
| Re-running | Safe: tool compares and only fixes differences | Can create duplicates unless you add checks |
| Tools | **Terraform, CloudFormation, Bicep, Kubernetes manifests** | Bash scripts, AWS CLI scripts, some Ansible playbooks (mixed), Pulumi (code but declarative engine) |

Taxi analogy 🚕:

```
Imperative: "Turn left, go 2 km, turn right, stop at the white building."
Declarative: "Take me to the airport."   ← you state the destination; driver (tool) chooses the route
```

Example:

```bash
# IMPERATIVE (script): runs again → creates ANOTHER instance
aws ec2 run-instances --image-id ami-xxx --count 1 --instance-type t3.micro

# DECLARATIVE (Terraform): "one aws_instance.web exists"
# Run 10 times → still exactly one instance (nothing to change after the first run)
```

## 4.2 Idempotency ⭐

> An operation is **idempotent** if running it once or many times gives the same final result.

```
Declarative IaC is naturally idempotent:
  apply #1 → creates resources
  apply #2 → "No changes. Infrastructure matches configuration."
```

Elevator button analogy: pressing "3rd floor" five times still takes you to the 3rd floor.

## 4.3 Desired state, current state, and drift

```
DESIRED STATE  = what your code says should exist
ACTUAL STATE   = what really exists in the cloud right now
DRIFT          = the difference between them (someone changed things outside IaC)
```

How IaC tools behave:

```
Read code → read current state → compute difference (the PLAN) → apply only the difference
```

Example: code says `instance_type = "t3.micro"`. Someone changes it to `t3.large` in the console → **drift**. Next plan: "change t3.large → t3.micro" (tool brings reality back to desired state).

## 4.4 Mutable vs Immutable infrastructure ⭐

| | **Mutable** | **Immutable** |
|---|---|---|
| Idea | Modify existing servers in place (patch, update) | Never modify; replace with new servers from a new image |
| Update flow | SSH in, apply patch | Build new image/instance, switch traffic, destroy old |
| Risk | Drift, snowflakes, failed partial updates | Very consistent; easy rollback (keep old image) |
| Example tools | Ansible/Chef on long-lived servers | Packer + Terraform; containers; AMIs; auto scaling replacement |
| Mindset | "Pets" 🐶 (named, cared for) | "Cattle" 🐄 (numbered, replaceable) |

Modern cloud practice leans **immutable**: replace, don't repair.

## 4.5 Push vs Pull model

| | Push | Pull |
|---|---|---|
| How | Central tool/engineer pushes config to servers | Agents on servers pull config from a central server periodically |
| Examples | Terraform, CloudFormation, Ansible (default) | Puppet, Chef, Argo CD/Flux (GitOps for Kubernetes) |
| Pros | Simple, immediate | Self-correcting drift automatically |
| Cons | Drift can persist until next push | Needs agent installation |

## 4.6 Agent-based vs Agentless

```
Agentless: nothing installed on target (Terraform calls cloud APIs; Ansible uses SSH)
Agent-based: a daemon runs on each server (Puppet agent, Chef client)
```

## 4.7 Provisioning vs Configuration Management ⭐

IaC covers two jobs. Don't mix them:

| | **Provisioning (infrastructure orchestration)** | **Configuration management** |
|---|---|---|
| Job | **Create** the infrastructure: VPC, VMs, DB, load balancers | **Configure** software inside servers: install Nginx, set files, users, patches |
| Level | Cloud resources | Operating system and applications |
| Tools | **Terraform, CloudFormation, ARM/Bicep, Pulumi** | **Ansible, Puppet, Chef, SaltStack** |
| Analogy | Building the house | Furnishing and decorating the rooms |

Typical combo: Terraform creates 5 servers → Ansible installs and configures software on them. With containers/user-data, much configuration moves into the image or bootstrap script.

Third helper: **Image building** (Packer) bakes AMIs/images (immutable approach).

---

# 5️⃣ THE IaC LIFECYCLE / WORKFLOW 🔄 ⭐

```
 ┌───────┐   ┌─────────┐   ┌──────┐   ┌────────┐   ┌───────┐   ┌─────────┐   ┌─────────┐
 │ WRITE │──▶│ VALIDATE│──▶│ PLAN │──▶│ REVIEW │──▶│ APPLY │──▶│ MONITOR │──▶│ DESTROY │
 └───────┘   └─────────┘   └──────┘   └────────┘   └───────┘   └─────────┘   │ (optional)
     ▲            syntax/        preview      human or       make real     detect drift   └─────────┘
     │            lint            changes     policy check   changes
     └───────────────────── change code and repeat ◀───────────────────────────────┘
```

| Step | Terraform | CloudFormation |
|---|---|---|
| Write | `.tf` files | YAML/JSON template |
| Validate | `terraform validate`, `terraform fmt` | `aws cloudformation validate-template`, `cfn-lint` |
| Plan/Preview | `terraform plan` | **Change set** |
| Apply | `terraform apply` | `create-stack` / `execute-change-set` / `update-stack` |
| Monitor drift | `terraform plan` (refresh) | **Drift detection** |
| Destroy | `terraform destroy` | `delete-stack` |

---

# 6️⃣ IaC IN THE GIT + CI/CD WORKFLOW 🧑‍💻

```
Developer
   │ 1. edit infra code on a feature branch
   ▼
Git repo ──2. Pull Request──▶ CI pipeline runs:
                               fmt / validate / lint / security scan / unit tests
                               terraform plan → posted as PR comment
   │ 3. teammate reviews code + plan
   ▼
Merge to main ──4.──▶ CD pipeline: terraform apply (with approval for prod)
                         │
                         ▼
                      Cloud infra updated ──▶ state saved ──▶ monitoring & drift checks
```

Rules of good IaC hygiene:

```
✔ One repo (or folder) per logical stack; environments via variables/workspaces/folders
✔ Never edit prod by hand ("no ClickOps"); emergency changes → back-port into code
✔ Remote state with locking, encrypted, access-controlled
✔ Secrets never in code (use secrets managers; Unit V)
✔ Pin tool and provider versions
✔ Small, frequent, reviewed changes
```

---

# 7️⃣ CATEGORIES OF IaC TOOLS 🧰 (MAP OF THE LANDSCAPE)

| Category | Tools | Notes |
|---|---|---|
| **Cloud-native, provider-specific** | AWS **CloudFormation**, Azure **ARM / Bicep**, Google Cloud Deployment Manager (being phased out; Infrastructure Manager is the Terraform-based successor) | Deep integration with one cloud |
| **Multi-cloud, declarative DSL** | **Terraform**, **OpenTofu** | HCL language; thousands of providers |
| **General-purpose language IaC** | **Pulumi**, **AWS CDK** (generates CloudFormation) | Write infra in Python/TypeScript/Go etc. |
| **Configuration management** | **Ansible**, Puppet, Chef, SaltStack | Configure software on servers |
| **Kubernetes-native** | Helm, Kustomize, **Crossplane**, Argo CD/Flux | Manage K8s apps and cloud resources via K8s APIs |
| **Image building** | **Packer** | Build AMIs/VM/container images |
| **Policy as code** | OPA/Rego, HashiCorp Sentinel, Checkov, Kyverno | Enforce rules on IaC |

Details of these appear in Parts 5 and 6.

---

# 8️⃣ DEMO: SAME INFRASTRUCTURE, THREE WAYS 🎭

Goal: "an S3 bucket with versioning".

## 8.1 Manual (console)
```
S3 → Create bucket → name → region → enable versioning → Create    (8 clicks, can't be reviewed or repeated exactly)
```

## 8.2 Imperative script (AWS CLI)
```bash
aws s3api create-bucket --bucket my-demo-bucket-123 --region ap-south-1 \
   --create-bucket-configuration LocationConstraint=ap-south-1
aws s3api put-bucket-versioning --bucket my-demo-bucket-123 \
   --versioning-configuration Status=Enabled
# Run again → ERROR (bucket exists). You must add if/else logic yourself.
```

## 8.3 Declarative IaC (Terraform)
```hcl
resource "aws_s3_bucket" "demo" {
  bucket = "my-demo-bucket-123"
}
resource "aws_s3_bucket_versioning" "demo" {
  bucket = aws_s3_bucket.demo.id
  versioning_configuration { status = "Enabled" }
}
# Run again → "No changes." ✅   Change a value → only the difference is applied.
```

## 8.4 Declarative IaC (CloudFormation)
```yaml
Resources:
  DemoBucket:
    Type: AWS::S3::Bucket
    Properties:
      VersioningConfiguration:
        Status: Enabled
```

Compare:

| | Console | CLI script | Terraform/CFN |
|---|---|---|---|
| Repeatable | ❌ | ⚠️ partly | ✅ |
| Idempotent | n/a | ❌ unless you code it | ✅ |
| Reviewable in Git | ❌ | ✅ (script) | ✅ |
| Shows a preview of changes | ❌ | ❌ | ✅ (plan / change set) |
| Tracks what it created | ❌ | ❌ | ✅ (state / stack) |
| Handles dependencies and ordering | you | you | ✅ automatic |

---

# 9️⃣ DEPENDENCY GRAPH (HOW TOOLS ORDER THINGS) 🕸️

Resources depend on each other: a subnet needs a VPC; an instance needs a subnet and a security group.

```
        VPC
       /   \
  Subnet    Security Group
       \   /
      EC2 Instance
```

IaC tools build this graph from references in your code (e.g., `aws_vpc.main.id` inside the subnet) and:

```
Create in dependency order (VPC → subnet/SG → instance)
Destroy in reverse order (instance → subnet/SG → VPC)
Create independent resources in parallel (faster)
```

You rarely write order manually. That is a big upgrade over scripts.

---

# 🔟 IaC TESTING AND SECURITY (PREVIEW) 🛡️

| Layer | What | Tools |
|---|---|---|
| Format/Syntax | Style and parse errors | `terraform fmt`, `terraform validate`, `cfn-lint` |
| Static analysis / security | Misconfigurations before deploy (open SSH, unencrypted S3) | **Checkov, tfsec/Trivy, cfn-nag, KICS** |
| Policy as code | Organization rules ("only allowed regions", "all resources must have tags") | OPA/Conftest, Sentinel, CloudFormation Guard, AWS CloudFormation Hooks |
| Plan review | Humans/automation inspect planned changes | `terraform plan`, change sets |
| Integration tests | Deploy to a test account and verify | Terratest, `terraform test`, kitchen-terraform |
| Drift detection | Compare reality with code | CloudFormation drift detection, scheduled `terraform plan` |
| Cost estimation | Predict monthly cost of a change | Infracost |

---

# 1️⃣1️⃣ BENEFITS vs CHALLENGES (BALANCED VIEW) ⚖️

| Benefits | Challenges / Risks |
|---|---|
| Speed, consistency, repeatability | Learning curve (new languages and concepts) |
| Version control and audit trail | State management complexity (state files, locking) |
| Easy disaster recovery | Drift if people still click in the console |
| Collaboration and review | Mistakes scale fast (a wrong `destroy` can delete a lot) |
| Compliance as code | Secrets can leak into code/state if careless |
| Cost control (ephemeral environments) | Tool/version/provider upgrades and breaking changes |
| Foundation for GitOps/platform engineering | Over-engineering small projects |

⚠️ **Blast radius:** automation is powerful both ways. Always read the plan before apply, protect critical resources, and keep backups and state versioning.

---

# 1️⃣2️⃣ DRY RUN: A FULL IaC CYCLE IN WORDS 🏃

```
Day 1: You write main code: 1 VPC + 1 EC2 (t3.micro). Run plan → "2 to add". Apply → resources created. State recorded.
Day 2: Boss wants t3.small. You edit ONE line. Plan → "1 to change". Review. Apply → instance modified.
Day 3: A teammate manually adds an open port in the console (drift). Scheduled plan flags: "1 to change (revert port)". 
       You decide: either update code to include it, or let the tool remove it.
Day 4: Region failure. You run the same code with region = another → whole environment rebuilt in minutes.
Day 5: Project ends. destroy → everything removed in reverse dependency order; billing stops.
```

---

# 1️⃣3️⃣ VIVA QUESTIONS (PART 1) 🎯

**Q1. What is Infrastructure as Code?** Managing and provisioning infrastructure through machine-readable definition files instead of manual processes.

**Q2. Why is IaC important in DevOps?** It automates environment creation inside the pipeline, brings version control, consistency, speed, auditability and enables CI/CD and GitOps for infrastructure.

**Q3. Declarative vs imperative?** Declarative states the desired end state and the tool decides steps (Terraform, CloudFormation); imperative lists exact steps (shell scripts).

**Q4. What is idempotency?** Re-running yields the same final state without duplicating or breaking things.

**Q5. What is configuration drift?** Divergence between the real infrastructure and the defined code, caused by manual or out-of-band changes.

**Q6. Mutable vs immutable infrastructure?** Mutable modifies servers in place; immutable replaces them with new ones built from a new image.

**Q7. Provisioning vs configuration management?** Provisioning creates infrastructure (Terraform); configuration management configures software inside servers (Ansible).

**Q8. Push vs pull?** Push sends configuration from a central point; pull has agents fetching configuration (Puppet, Argo CD).

**Q9. Name IaC tools.** Terraform, OpenTofu, CloudFormation, Bicep/ARM, Pulumi, AWS CDK, Ansible, Puppet, Chef, Crossplane.

**Q10. What is a dependency graph in IaC?** The order of resource creation/destruction derived from references between resources.

**Q11. What is GitOps?** Using Git as the single source of truth where automation reconciles actual infrastructure to the state in Git.

**Q12. Why shouldn't we click in the console in an IaC setup?** It creates drift and untracked changes that code does not reflect.

---

# 🧠 PART 1 MASTER SHEET

```
ClickOps problems: snowflakes, drift, slow, no audit, errors, no DR
IaC = infrastructure defined in code files (versioned, reviewed, repeatable)
DevOps value: speed, consistency, Git history, collaboration, CI/CD, DR, compliance
Declarative (WHAT: Terraform/CFN) vs Imperative (HOW: scripts)
Idempotent: run many times = same result
Desired state vs actual state → DRIFT → plan shows difference
Mutable (patch in place) vs Immutable (replace) ; pets vs cattle
Push (Terraform, Ansible) vs Pull (Puppet, Argo CD) ; agentless vs agent
Provisioning (Terraform/CFN) ≠ Config management (Ansible/Puppet)
Lifecycle: write → validate → plan → review → apply → monitor → destroy
Dependency graph = automatic ordering ; Test: fmt/validate/lint/security scan/plan/tests/drift
```

---

## 🫡 PART 1 STATUS

You now know why IaC exists, what it is, the vocabulary examiners love, and how it fits into DevOps pipelines.

**STOP here.** Next up:

> **PART 2 — AWS CloudFormation (template anatomy, intrinsic functions, stacks, change sets, drift, rollback, nested stacks) 🔥**
