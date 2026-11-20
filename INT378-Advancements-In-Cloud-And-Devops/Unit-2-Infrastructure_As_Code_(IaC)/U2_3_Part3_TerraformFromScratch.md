# 🟣 UNIT II — PART 3: TERRAFORM FROM ABSOLUTE SCRATCH

> **Covered in this document:** what Terraform is and who made it → how it works (core + providers + state + plan) → installing Terraform (Windows, macOS, Linux) and AWS credentials → HCL syntax (blocks, arguments, expressions, comments) → the main block types (terraform, provider, resource, variable, output, locals, data, module) → the 4-command workflow (init, plan, apply, destroy) plus fmt/validate/show/output → resource references and implicit dependencies → input variables (types, defaults, validation, sensitive, tfvars, precedence) → outputs → locals → data sources → built-in functions and expressions → a first complete project (S3 bucket) → state file introduction → project file layout → `.gitignore` → dry run of init/plan/apply → common errors → viva → master sheet.

---

Absolutely mate. 😤🔥 CloudFormation was the AWS-only tool. Now the **multi-cloud industry favourite: Terraform** (Practical 5).

---

# 1️⃣ WHAT IS TERRAFORM?

> **Terraform** is an open-source-style IaC tool by **HashiCorp** (now part of IBM) that lets you define cloud and on-prem resources in human-readable configuration files (**HCL: HashiCorp Configuration Language**) and manage their full lifecycle through a consistent workflow.

Key facts:

```
Declarative              → describe desired state
Cloud-agnostic           → one workflow for AWS, Azure, GCP, Kubernetes, GitHub, Cloudflare, Datadog ... via PROVIDERS
Written in Go            → single binary, easy to install
Language: HCL            → readable; also supports JSON syntax
Keeps STATE              → a file mapping your code to real resources
Plan before apply        → preview of changes
Large ecosystem          → Terraform Registry with thousands of providers and community modules
Licensing note           → since Aug 2023 under Business Source License (BSL 1.1);
                           the community fork is OpenTofu (MPL 2.0). Details in Part 6.
```

## 1.1 How Terraform works (architecture) ⭐⭐

```
┌───────────────┐    ┌─────────────────────────────────────────────┐
│ Your .tf files │──▶│ TERRAFORM CORE                               │
│ (desired state)│   │  1. reads config                             │
└───────────────┘    │  2. reads STATE (what it knows exists)        │
                     │  3. asks PROVIDER for real current status     │
┌───────────────┐    │  4. builds dependency graph                   │
│ terraform.tfstate│◀▶│  5. computes PLAN = desired − current        │
│ (state)        │   │  6. apply → calls providers in right order    │
└───────────────┘    └───────────────┬─────────────────────────────┘
                                      │ gRPC
                         ┌────────────▼────────────┐
                         │ PROVIDER (plugin)        │  e.g. hashicorp/aws
                         │ translates to API calls  │
                         └────────────┬────────────┘
                                      ▼
                          AWS / Azure / GCP / K8s API
```

| Component | Role |
|---|---|
| **Terraform Core** | The CLI engine: parsing, graph, plan, apply, state handling |
| **Providers** | Plugins that know how to talk to one platform's API (AWS provider, Azure provider...) |
| **Resources** | The infrastructure objects you declare |
| **State** | The record of what Terraform manages |
| **Registry** | Public library of providers and modules (registry.terraform.io) |
| **Backend** | Where state is stored and how operations run (local, S3, HCP Terraform...) |

---

# 2️⃣ INSTALLING TERRAFORM 💻

(Always confirm the latest steps at developer.hashicorp.com/terraform/install.)

## 2.1 Windows

```
Option A (winget):   winget install HashiCorp.Terraform
Option B (Chocolatey): choco install terraform
Option C (manual):  download ZIP from the official site → extract terraform.exe → add its folder to PATH
```

## 2.2 macOS

```bash
brew tap hashicorp/tap
brew install hashicorp/tap/terraform
```

## 2.3 Linux (Ubuntu/Debian, via HashiCorp apt repo)

```bash
sudo apt-get update && sudo apt-get install -y gnupg software-properties-common curl
curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt-get update && sudo apt-get install -y terraform
```

Amazon Linux/RHEL: use HashiCorp's yum/dnf repo instructions from the install page (or download the zip binary).

## 2.4 Verify

```bash
terraform version
terraform -help
terraform -install-autocomplete     # optional tab completion
```

## 2.5 Give Terraform AWS credentials (never hard-code in .tf files!)

Pick one:

```bash
# A) AWS CLI profile (recommended for lab)
aws configure                         # enter Access Key, Secret, region, json
aws sts get-caller-identity           # verify who you are

# B) Environment variables (temporary)
export AWS_ACCESS_KEY_ID="..."
export AWS_SECRET_ACCESS_KEY="..."
export AWS_DEFAULT_REGION="ap-south-1"

# C) On EC2/CI: use an IAM ROLE (no keys at all) ← best practice
# D) AWS SSO / IAM Identity Center:  aws sso login --profile myprofile
```

Create a dedicated **IAM user/role with limited permissions** for the lab (EC2, S3, VPC). Do **not** use the root account's keys.

---

# 3️⃣ HCL SYNTAX BASICS 📜

## 3.1 Building blocks

```hcl
# This is a comment      // also a comment      /* multi-line comment */

<BLOCK TYPE> "<LABEL 1>" "<LABEL 2>" {
  argument_name = expression
  nested_block {
    another_arg = "value"
  }
}
```

Example:

```hcl
resource "aws_instance" "web" {        # block type=resource, type label=aws_instance, name label=web
  ami           = "ami-xxxx"           # argument = value
  instance_type = "t3.micro"
  tags = {                             # map
    Name = "web-1"
  }
}
```

Full address of that resource inside Terraform: `aws_instance.web`.

## 3.2 Data types

| Type | Example |
|---|---|
| `string` | `"hello"` |
| `number` | `42`, `3.14` |
| `bool` | `true`, `false` |
| `list(type)` | `["a", "b", "c"]` |
| `set(type)` | unique unordered values |
| `map(type)` | `{ env = "dev", owner = "rahul" }` |
| `object({...})` | structured: `{ name = string, size = number }` |
| `tuple([...])` | fixed-length mixed list |
| `any` | any type |

## 3.2.1 Expressions

```hcl
"Hello ${var.name}"                 # string interpolation
var.count > 2 ? "big" : "small"     # conditional
[for s in var.names : upper(s)]     # for expression → list
{for k, v in var.tags : k => v}     # for expression → map
aws_instance.web.public_ip          # attribute reference
var.list[0]  /  var.map["key"]      # indexing
```

## 3.3 Block types you must know ⭐

| Block | Purpose |
|---|---|
| `terraform { }` | Settings: required Terraform version, required providers, backend |
| `provider "aws" { }` | Configure a provider (region, profile, default tags) |
| `resource "type" "name" { }` | Create/manage an infrastructure object |
| `data "type" "name" { }` | **Read** existing information (not create) |
| `variable "name" { }` | Input parameter |
| `output "name" { }` | Value to display/export |
| `locals { }` | Named intermediate values |
| `module "name" { }` | Call a reusable group of resources |
| `import { }` / `moved { }` / `removed { }` | Bring existing resource under management / refactor addresses / stop managing |

---

# 4️⃣ THE CORE WORKFLOW ⭐⭐⭐ (EVERY TERRAFORM EXAM QUESTION)

```
      write .tf files
            │
   terraform init      → download providers, set up backend, create .terraform/ and .terraform.lock.hcl
            │
   terraform fmt       → auto-format code (optional but good)
   terraform validate  → check syntax and internal consistency
            │
   terraform plan      → PREVIEW: what will be created / changed / destroyed
            │
   terraform apply     → execute the plan (asks "yes" to confirm)
            │
   (use / modify code / plan / apply again ...)
            │
   terraform destroy   → delete everything managed by this configuration
```

## 4.1 Command table

| Command | What it does |
|---|---|
| `terraform init` | Initialize working dir: downloads providers and modules, configures backend. **First command always** (and after adding providers/modules/backend) |
| `terraform fmt` | Rewrites files in canonical format |
| `terraform validate` | Verifies configuration is syntactically valid and internally consistent (no cloud calls) |
| `terraform plan` | Shows the execution plan; changes nothing |
| `terraform plan -out=tfplan` | Saves the plan so `apply` runs exactly that |
| `terraform apply` | Applies changes (prompts for confirmation) |
| `terraform apply tfplan` | Applies a saved plan |
| `terraform apply -auto-approve` | Skips the prompt (for automation; use carefully) |
| `terraform destroy` | Plans and executes deletion of all managed resources |
| `terraform show` | Human-readable view of state or a plan |
| `terraform output` | Prints output values |
| `terraform state list` | Lists resources in state |
| `terraform state show aws_instance.web` | Shows one resource's recorded attributes |
| `terraform refresh` | (older) update state from reality; now part of plan/apply (`-refresh-only`) |
| `terraform taint` / `-replace=ADDR` | Force recreation of a resource (`terraform apply -replace="aws_instance.web"`) |
| `terraform import` | Bring an existing resource into state |
| `terraform graph` | Output dependency graph (DOT format) |
| `terraform providers` / `terraform version` | Info |
| `terraform console` | Interactive expression evaluator |
| `terraform workspace ...` | Manage workspaces |
| `terraform test` | Run `.tftest.hcl` tests |

## 4.2 Plan symbols ⭐

```
+  create
-  destroy
~  update in-place
-/+ destroy and re-create (replacement)    ← read carefully!
<= read (data source)
```

Example plan output (shortened):

```
Terraform will perform the following actions:

  # aws_s3_bucket.demo will be created
  + resource "aws_s3_bucket" "demo" {
      + bucket = "tf-demo-bucket-a1b2c3"
      + id     = (known after apply)
      ...
    }

Plan: 1 to add, 0 to change, 0 to destroy.
```

`(known after apply)` means the value is decided by AWS during creation.

---

# 5️⃣ PROVIDERS AND THE `terraform` BLOCK

```hcl
terraform {
  required_version = ">= 1.5.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"      # any 6.x, not 7.0  (check the latest major on registry.terraform.io)
    }
    random = {
      source  = "hashicorp/random"
      version = "~> 3.6"
    }
  }
}

provider "aws" {
  region = var.region           # which region to deploy into
  # profile = "default"        # optional AWS CLI profile

  default_tags {                # applied to every taggable resource automatically
    tags = {
      Project   = "INT378-Lab"
      ManagedBy = "Terraform"
    }
  }
}
```

### Version constraints

| Constraint | Meaning |
|---|---|
| `= 6.1.0` | Exactly that |
| `>= 5.0` | At least |
| `~> 6.0` | `>= 6.0` and `< 7.0` (rightmost number may increase) |
| `~> 6.1.0` | `>= 6.1.0` and `< 6.2.0` |

### The lock file `.terraform.lock.hcl`

`init` records exact provider versions + checksums. **Commit it to Git** so teammates/CI use identical versions.

### Multiple regions: provider alias

```hcl
provider "aws" { region = "ap-south-1" }
provider "aws" { alias = "virginia", region = "us-east-1" }

resource "aws_s3_bucket" "replica" {
  provider = aws.virginia
  bucket   = "my-replica-bucket-123"
}
```

---

# 6️⃣ RESOURCES, REFERENCES AND DEPENDENCIES

```hcl
resource "aws_security_group" "web" {
  name        = "web-sg"
  description = "Allow HTTP"
  vpc_id      = data.aws_vpc.default.id

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_instance" "web" {
  ami                    = "ami-xxxx"
  instance_type          = "t3.micro"
  vpc_security_group_ids = [aws_security_group.web.id]   # ← reference → implicit dependency
}
```

Reference syntax: `<resource_type>.<name>.<attribute>`.
`aws_security_group.web.id` tells Terraform: create the SG **first**, then the instance.

| Dependency | How |
|---|---|
| **Implicit** | Any reference creates an ordering automatically ✅ |
| **Explicit** | `depends_on = [aws_internet_gateway.gw]` when no reference exists |

Resource meta-arguments (work on every resource): `count`, `for_each`, `depends_on`, `provider`, `lifecycle` (Part 4).

---

# 7️⃣ INPUT VARIABLES ⭐

```hcl
variable "region" {
  description = "AWS region"
  type        = string
  default     = "ap-south-1"
}

variable "instance_type" {
  type    = string
  default = "t3.micro"
  validation {
    condition     = contains(["t3.micro", "t3.small", "t2.micro"], var.instance_type)
    error_message = "instance_type must be t3.micro, t3.small or t2.micro."
  }
}

variable "allowed_ssh_cidr" {
  type        = string
  description = "CIDR allowed to SSH"
  # no default → Terraform will ASK for it
}

variable "db_password" {
  type      = string
  sensitive = true          # hidden in plan/apply output (still stored in state!)
}

variable "tags" {
  type    = map(string)
  default = { Environment = "dev" }
}
```

Use it: `var.instance_type`.

## 7.1 Ways to set a variable (precedence: later wins) ⭐

```
1. default in the variable block              (lowest)
2. Environment variable:  TF_VAR_instance_type=t3.small
3. terraform.tfvars  (auto-loaded)
4. terraform.tfvars.json
5. *.auto.tfvars / *.auto.tfvars.json   (auto-loaded alphabetically)
6. -var-file="prod.tfvars" and -var="name=value" on the command line   (highest, in the order given)
```

`terraform.tfvars`:

```hcl
region        = "ap-south-1"
instance_type = "t3.micro"
allowed_ssh_cidr = "203.0.113.25/32"
```

Environment-specific: `dev.tfvars`, `prod.tfvars` → `terraform apply -var-file=prod.tfvars`.

---

# 8️⃣ OUTPUTS AND LOCALS

## 8.1 Outputs (like CloudFormation Outputs)

```hcl
output "instance_public_ip" {
  description = "Public IP of the web server"
  value       = aws_instance.web.public_ip
}

output "website_url" {
  value = "http://${aws_instance.web.public_ip}"
}

output "db_password_hint" {
  value     = var.db_password
  sensitive = true
}
```

After apply: `terraform output`, `terraform output -raw website_url`, `terraform output -json`.

## 8.2 Locals (named expressions, avoid repetition)

```hcl
locals {
  name_prefix = "${var.project}-${var.environment}"
  common_tags = {
    Project     = var.project
    Environment = var.environment
  }
}

resource "aws_s3_bucket" "logs" {
  bucket = "${local.name_prefix}-logs-${random_id.suffix.hex}"
  tags   = local.common_tags
}
```

| | variable | local |
|---|---|---|
| Set from outside? | Yes | No (computed inside) |
| Purpose | Inputs | Internal reuse |

---

# 9️⃣ DATA SOURCES (READ, DON'T CREATE) ⭐

```hcl
# Default VPC and its subnets
data "aws_vpc" "default" {
  default = true
}

data "aws_subnets" "default" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.default.id]
  }
}

# Latest Amazon Linux 2023 AMI from the public SSM parameter
data "aws_ssm_parameter" "al2023" {
  name = "/aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64"
}

# Who am I
data "aws_caller_identity" "current" {}

data "aws_availability_zones" "available" {
  state = "available"
}
```

Use: `data.aws_vpc.default.id`, `data.aws_ssm_parameter.al2023.value`, `data.aws_caller_identity.current.account_id`.

| | `resource` | `data` |
|---|---|---|
| Creates? | Yes | No, only reads existing info |
| Managed in state | Yes | Cached read |
| Example | `aws_instance` | `aws_ami`, `aws_vpc`, `aws_caller_identity` |

Equivalent to CloudFormation's SSM parameter types and pseudo parameters.

---

# 🔟 USEFUL BUILT-IN FUNCTIONS

| Category | Functions |
|---|---|
| String | `upper`, `lower`, `format`, `join`, `split`, `replace`, `trimspace`, `substr` |
| Collection | `length`, `concat`, `contains`, `element`, `keys`, `values`, `merge`, `lookup`, `flatten`, `distinct`, `toset` |
| Numeric | `min`, `max`, `ceil`, `floor` |
| Filesystem | `file("path")`, `templatefile("path", {vars})`, `filebase64` |
| Encoding | `base64encode`, `jsonencode`, `yamlencode` |
| Network | `cidrsubnet("10.0.0.0/16", 8, 1)` → `10.0.1.0/24`, `cidrhost` |
| Type | `tostring`, `tonumber`, `tolist` |

Test them: `terraform console` → `> upper("hi")` → `"HI"`; `> cidrsubnet("10.0.0.0/16", 8, 2)` → `"10.0.2.0/24"`.

User data from a file:

```hcl
user_data = templatefile("${path.module}/userdata.sh.tftpl", { site_name = var.site_name })
```

---

# 1️⃣1️⃣ FIRST COMPLETE PROJECT: S3 BUCKET (SAFE, CHEAP, UNIQUE NAME) 🧪

Folder:

```
tf-s3-demo/
├── main.tf
├── variables.tf
├── outputs.tf
├── versions.tf
├── terraform.tfvars
└── .gitignore
```

`versions.tf`:

```hcl
terraform {
  required_version = ">= 1.5.0"
  required_providers {
    aws    = { source = "hashicorp/aws",    version = "~> 6.0" }
    random = { source = "hashicorp/random", version = "~> 3.6" }
  }
}
```

`variables.tf`:

```hcl
variable "region" {
  type    = string
  default = "ap-south-1"
}
variable "project" {
  type    = string
  default = "tf-demo"
}
```

`main.tf`:

```hcl
provider "aws" {
  region = var.region
  default_tags {
    tags = { Project = var.project, ManagedBy = "Terraform" }
  }
}

# random suffix so the globally-unique bucket name never clashes
resource "random_id" "suffix" {
  byte_length = 4
}

resource "aws_s3_bucket" "demo" {
  bucket = "${var.project}-${random_id.suffix.hex}"
}

resource "aws_s3_bucket_versioning" "demo" {
  bucket = aws_s3_bucket.demo.id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_public_access_block" "demo" {
  bucket                  = aws_s3_bucket.demo.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```

`outputs.tf`:

```hcl
output "bucket_name" {
  value = aws_s3_bucket.demo.bucket
}
output "bucket_arn" {
  value = aws_s3_bucket.demo.arn
}
```

`.gitignore` ⭐:

```
.terraform/
*.tfstate
*.tfstate.*
crash.log
*.tfvars          # if they contain secrets
.terraform.tfstate.lock.info
# DO commit: *.tf and .terraform.lock.hcl
```

Run:

```bash
terraform init
terraform fmt
terraform validate          # Success! The configuration is valid.
terraform plan              # Plan: 3 to add (random_id, bucket, versioning, access block → 4)
terraform apply             # type: yes
terraform output
aws s3 ls | grep tf-demo    # verify in AWS
terraform destroy           # type: yes → bucket removed
```

(Count: random_id + bucket + versioning + public access block = **4 resources**.)

---

# 1️⃣2️⃣ STATE: A FIRST LOOK ⭐⭐⭐

After `apply`, Terraform writes **`terraform.tfstate`** (JSON) next to your code (local backend).

## 12.1 What is state?

> **State** is Terraform's record of the real resources it manages: it maps resource addresses in your code (e.g., `aws_s3_bucket.demo`) to real-world IDs (e.g., bucket `tf-demo-a1b2c3d4`) plus all their attributes.

Why it exists:

```
1. MAPPING     → which real resource corresponds to which block of code
2. METADATA    → dependencies, resource order for destroy
3. PERFORMANCE → cache attributes so Terraform doesn't query everything every time
4. COLLABORATION (with remote state) → one shared truth for the team
```

## 12.2 Looking at state

```bash
terraform state list
# random_id.suffix
# aws_s3_bucket.demo
# aws_s3_bucket_versioning.demo
# aws_s3_bucket_public_access_block.demo

terraform state show aws_s3_bucket.demo
terraform show                      # whole state in readable form
```

## 12.3 Golden rules for state 🚨

```
❌ Never edit terraform.tfstate by hand
❌ Never commit it to Git (it can contain secrets in plain text: passwords, keys)
❌ Don't delete it: Terraform will "forget" resources and try to create duplicates
✔ Use REMOTE state (S3 + locking) for teams (Part 4)
✔ Enable versioning/encryption on the state bucket
✔ Use `terraform state` commands, `import`, `moved` blocks to change state safely
✔ Back up before risky operations
```

Local files you will see:

```
terraform.tfstate          current state
terraform.tfstate.backup   previous state (safety copy)
.terraform/                downloaded providers/modules
.terraform.lock.hcl        provider version lock (commit this)
```

---

# 1️⃣3️⃣ DRY RUN: WHAT HAPPENS ON EACH COMMAND 🏃

```
terraform init
  → reads required_providers, downloads hashicorp/aws and hashicorp/random into .terraform/
  → creates .terraform.lock.hcl with versions + checksums
  → "Terraform has been successfully initialized!"

terraform plan
  → loads config, loads state (empty on first run)
  → builds graph: random_id → aws_s3_bucket → (versioning, access block)
  → refreshes (queries AWS for existing matching resources: none)
  → diff = 4 resources to add
  → prints plan; changes NOTHING

terraform apply (yes)
  → creates random_id (no API call)
  → creates bucket (S3 API CreateBucket) → records ID in state
  → creates versioning + access block in parallel (they both depend only on bucket)
  → writes terraform.tfstate
  → "Apply complete! Resources: 4 added, 0 changed, 0 destroyed."  + outputs

terraform apply (again)
  → "No changes. Your infrastructure matches the configuration."      ← idempotency ✅

edit variables (e.g., project name) → plan shows -/+ replacement (bucket name changed → S3 can't rename)

terraform destroy
  → reverse order: versioning, access block → bucket → random_id → state emptied
```

---

# 1️⃣4️⃣ PROJECT LAYOUT BEST PRACTICE

```
project/
├── versions.tf        # terraform + required_providers (+ backend)
├── providers.tf       # provider configuration
├── variables.tf       # input variable declarations
├── locals.tf          # locals
├── main.tf            # resources (or split: network.tf, compute.tf, storage.tf)
├── data.tf            # data sources
├── outputs.tf         # outputs
├── terraform.tfvars   # values (not secrets)
└── modules/           # reusable child modules
```

Terraform loads **all `.tf` files** in the folder as one configuration; file names are only for humans.

---

# 1️⃣5️⃣ COMMON ERRORS TABLE 🛠️

| Error | Cause | Fix |
|---|---|---|
| `terraform: command not found` | Not installed / PATH | Install, restart terminal |
| `No valid credential sources found` / `NoCredentialProviders` | AWS creds missing | `aws configure`, env vars, or role |
| `Error: Missing required provider` / `Inconsistent dependency lock file` | Need init | `terraform init` (or `-upgrade`) |
| `Unsupported argument` / `Unsupported block type` | Typo or provider version change (e.g., AWS provider v4+ split S3 settings into separate resources) | Read docs for your provider version |
| `Error: Reference to undeclared resource/variable` | Wrong name | Check spelling and addresses |
| `BucketAlreadyExists` | Global name taken | Use `random_id` suffix |
| `Error acquiring the state lock` | Another run holds lock or crashed | Wait; `terraform force-unlock <ID>` only if sure |
| `UnauthorizedOperation` / `AccessDenied` | IAM permissions | Add needed permissions |
| `InvalidAMIID.NotFound` | AMI from another region | Use data source / SSM parameter |
| `Error: Cycle` | Circular dependency | Break reference loop |
| Plan wants to destroy/recreate unexpectedly | Changed an immutable argument (name, AMI, subnet) | Read plan carefully; use `lifecycle`/`moved` if needed |
| `terraform destroy` leaves S3 bucket | Not empty | `force_destroy = true` on bucket or empty first |

---

# 1️⃣6️⃣ VIVA QUESTIONS (PART 3) 🎯

**Q1. What is Terraform?** A declarative, cloud-agnostic IaC tool by HashiCorp that provisions resources via provider plugins using HCL.

**Q2. What is a provider?** Plugin that translates Terraform configuration into API calls for a platform (AWS, Azure, Kubernetes).

**Q3. Explain init, plan, apply, destroy.** init downloads providers/sets backend; plan previews changes; apply executes; destroy removes managed resources.

**Q4. What is the Terraform state?** A file mapping config to real resources and storing their attributes, enabling plan, dependencies and collaboration.

**Q5. Resource vs data source?** Resource creates/manages; data source reads existing information.

**Q6. Variable vs local vs output?** Inputs / internal named expressions / returned values.

**Q7. How does Terraform know the order of creation?** Builds a dependency graph from references; `depends_on` for explicit dependencies.

**Q8. What is `.terraform.lock.hcl`?** Lock file recording exact provider versions/checksums; commit it.

**Q9. Why shouldn't state be committed to Git?** It may contain sensitive data and causes conflicts; use remote state.

**Q10. What does `~> 6.0` mean?** Allow 6.x updates but not 7.0.

**Q11. Is Terraform idempotent?** Yes: re-applying without changes yields "No changes".

**Q12. What does `-/+` mean in a plan?** Resource will be destroyed and recreated (replacement).

**Q13. How do you pass secrets to Terraform safely?** Environment variables/secret managers/sensitive variables; remember they can still land in state, so protect state.

---

# 🧠 PART 3 MASTER SHEET

```
Terraform = declarative multi-cloud IaC (HashiCorp/IBM), HCL, providers, state, plan-before-apply
Architecture: Core + Providers (plugins) + State + Backend + Registry
Workflow: init → fmt → validate → plan → apply → (change) → destroy
Plan symbols: + create | - destroy | ~ update | -/+ replace | <= read
Blocks: terraform, provider, resource, data, variable, output, locals, module
Refs: type.name.attribute → implicit dependency ; depends_on = explicit
Variable precedence: default < env TF_VAR_ < terraform.tfvars < *.auto.tfvars < -var/-var-file
Data source = read ; resource = create ; sensitive hides output but state still stores it
State: terraform.tfstate (don't edit, don't commit) ; lock file .terraform.lock.hcl (commit)
Version constraints: = , >= , ~>
Creds: CLI profile / env vars / IAM role — never in .tf
```

---

## 🫡 PART 3 STATUS

You can install Terraform, write HCL, understand providers, variables, outputs, data sources and state, and run the full init → plan → apply → destroy loop.

**STOP here.** Next up:

> **PART 4 — Terraform Advanced: remote backends + locking, modules, count/for_each, lifecycle, provisioners, workspaces, import/moved, testing 🔥**
