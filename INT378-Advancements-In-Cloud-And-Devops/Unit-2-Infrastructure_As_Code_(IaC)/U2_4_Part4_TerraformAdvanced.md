# 🟣 UNIT II — PART 4: TERRAFORM ADVANCED (STATE, BACKENDS, MODULES, LOOPS, LIFECYCLE)

> **Covered in this document:** state deep dive (what's inside, risks, commands) → remote state with the S3 backend, encryption, versioning, native locking with `use_lockfile` (and the older DynamoDB locking) → bootstrapping the state bucket → migrating state → `terraform_remote_state` → `count` vs `for_each` (with examples and the index-shift trap) → conditional resources → `dynamic` blocks → `lifecycle` (`create_before_destroy`, `prevent_destroy`, `ignore_changes`, `replace_triggered_by`, pre/postconditions) → modules (structure, calling, inputs/outputs, registry, versioning) with a full VPC/EC2 module example → workspaces vs folders for environments → provisioners (and why they're a last resort) → import blocks, `moved` blocks, `removed` blocks → drift handling (`plan -refresh-only`, `-replace`) → `terraform test` → security (secrets in state, scanning) → CI/CD workflow → HCP Terraform awareness → common errors → viva → master sheet.

---

Absolutely mate. 😤🔥 Part 3 got you running Terraform alone. Part 4 makes it **team-ready and production-style**.

---

# 1️⃣ STATE DEEP DIVE ⭐⭐⭐

## 1.1 What is inside `terraform.tfstate`?

JSON with: Terraform version, a `serial` (increments each change), a `lineage` (unique ID), and for each resource: type, name, provider, instances with **all attributes** (IDs, ARNs, IPs, and even sensitive values like generated passwords).

```json
{
  "version": 4,
  "terraform_version": "1.x.x",
  "serial": 7,
  "lineage": "…uuid…",
  "resources": [
    { "mode": "managed", "type": "aws_s3_bucket", "name": "demo",
      "instances": [ { "attributes": { "id": "tf-demo-a1b2c3d4", "arn": "...", ... } } ] }
  ]
}
```

## 1.2 Problems with LOCAL state in a team

| Problem | Effect |
|---|---|
| File lives on one laptop | Others can't see/modify infra safely |
| No locking | Two people apply at once → corrupted/conflicting state |
| Not backed up | Laptop dies → lost mapping → duplicates / orphaned resources |
| Contains secrets | Committing to Git leaks them |
| No history | Can't roll back state |

**Solution: remote backend with locking.**

## 1.3 State commands (safe surgery) 🛠️

| Command | Use |
|---|---|
| `terraform state list` | Resources in state |
| `terraform state show ADDR` | Details of one |
| `terraform state mv OLD NEW` | Rename/move address without destroying (refactor) |
| `terraform state rm ADDR` | Stop managing a resource (does NOT delete the real thing) |
| `terraform state pull` / `push` | Download/upload raw state (careful) |
| `terraform force-unlock LOCK_ID` | Break a stuck lock (only if you're sure nobody is running) |
| `terraform apply -refresh-only` | Update state to match reality (record drift) without changing infra |

---

# 2️⃣ REMOTE BACKEND: S3 + LOCKING ⭐⭐⭐ (PRACTICAL 5 "MANAGING STATE FILES")

## 2.1 What is a backend?

> A **backend** defines where Terraform stores state and (for some backends) where operations run.

| Backend | Notes |
|---|---|
| `local` (default) | File on disk |
| **`s3`** | State in S3 (+ locking); very common on AWS |
| `azurerm` | Azure Blob Storage |
| `gcs` | Google Cloud Storage |
| `remote` / `cloud` | **HCP Terraform** (HashiCorp's managed service: state, runs, policy, SSO) |
| `http`, `pg`, `kubernetes`, `consul` | Others |

## 2.2 S3 backend configuration

```hcl
terraform {
  backend "s3" {
    bucket       = "myorg-terraform-state-123456789012"
    key          = "dev/network/terraform.tfstate"      # path inside the bucket (one key per stack/environment)
    region       = "ap-south-1"
    encrypt      = true                                 # server-side encryption of the state object
    use_lockfile = true                                 # S3 native locking (Terraform 1.10+, GA in 1.11)
  }
}
```

**Locking history (nice exam point):**

```
Old way: S3 for state + a DynamoDB table (dynamodb_table = "tf-locks") for the lock
New way: use_lockfile = true → Terraform writes a <key>.tflock object in the same bucket;
         DynamoDB-based locking is now deprecated and expected to be removed in a future version.
```

Requirements for native locking: Terraform ≥ 1.10, S3 versioning recommended, IAM permissions for `s3:GetObject`, `s3:PutObject`, `s3:DeleteObject` (the lock file too) and `s3:ListBucket`. If you're on an older Terraform, use the DynamoDB table (`LockID` partition key, string).

⚠️ Backend blocks **cannot use variables** (they're evaluated before variables). Use `-backend-config` flags or a partial config file for dynamic values.

## 2.3 Bootstrapping the state bucket (chicken-and-egg)

The state bucket must exist **before** you point a backend at it. Create it once:

```bash
BUCKET=myorg-terraform-state-$(aws sts get-caller-identity --query Account --output text)
aws s3api create-bucket --bucket $BUCKET --region ap-south-1 \
    --create-bucket-configuration LocationConstraint=ap-south-1
aws s3api put-bucket-versioning --bucket $BUCKET --versioning-configuration Status=Enabled
aws s3api put-bucket-encryption --bucket $BUCKET --server-side-encryption-configuration \
   '{"Rules":[{"ApplyServerSideEncryptionByDefault":{"SSEAlgorithm":"AES256"}}]}'
aws s3api put-public-access-block --bucket $BUCKET --public-access-block-configuration \
   BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true
```

(Or create the bucket with a tiny separate Terraform config using local state, or with CloudFormation. 😉)

## 2.4 Migrating local state to the remote backend

```bash
# 1. add the backend block to versions.tf / backend.tf
# 2. re-initialize and answer "yes" to copy existing state
terraform init -migrate-state
# 3. verify
terraform state list
aws s3 ls s3://$BUCKET/dev/network/
```

## 2.5 What locking does (DRY RUN)

```
Alice: terraform apply  → creates lock (dev/network/terraform.tfstate.tflock) → running...
Bob:   terraform apply  → "Error acquiring the state lock ... Lock Info: ID, Who: alice@..., Created: ..."
Alice finishes → lock released → Bob can now run
```

If Alice's run crashed and left the lock: `terraform force-unlock <LOCK_ID>`.

## 2.6 Reading another stack's outputs

```hcl
data "terraform_remote_state" "network" {
  backend = "s3"
  config = {
    bucket = "myorg-terraform-state-123456789012"
    key    = "dev/network/terraform.tfstate"
    region = "ap-south-1"
  }
}

# use: data.terraform_remote_state.network.outputs.vpc_id
```

(Alternative with less coupling: publish values to SSM Parameter Store and read with a `data` source.)

## 2.7 State security checklist

```
☐ S3 bucket: private, versioning ON, encryption ON, access logging/CloudTrail
☐ Restrict IAM access to the state bucket to the pipeline/role and few admins
☐ Never commit state; never paste state in chats
☐ Separate state per environment/stack (small blast radius)
☐ Sensitive values: they still appear in state → protect state like a secret
```

---

# 3️⃣ `count` AND `for_each` ⭐⭐ (CREATE MANY RESOURCES)

## 3.1 `count` (by number)

```hcl
resource "aws_instance" "web" {
  count         = 3
  ami           = data.aws_ssm_parameter.al2023.value
  instance_type = "t3.micro"
  tags = { Name = "web-${count.index}" }       # web-0, web-1, web-2
}
# addresses: aws_instance.web[0], [1], [2]
# reference all: aws_instance.web[*].id
```

## 3.2 Conditional creation with `count`

```hcl
resource "aws_cloudwatch_metric_alarm" "cpu" {
  count = var.environment == "prod" ? 1 : 0      # only in prod
  ...
}
```

## 3.3 `for_each` (by map or set of strings) ✅ preferred for named things

```hcl
variable "buckets" {
  type    = set(string)
  default = ["logs", "backups", "assets"]
}

resource "aws_s3_bucket" "this" {
  for_each = var.buckets
  bucket   = "${var.project}-${each.key}-${random_id.suffix.hex}"
}
# addresses: aws_s3_bucket.this["logs"], ["backups"], ["assets"]
```

With a map of objects:

```hcl
variable "servers" {
  type = map(object({ instance_type = string, az_index = number }))
  default = {
    web = { instance_type = "t3.micro", az_index = 0 }
    api = { instance_type = "t3.small", az_index = 1 }
  }
}

resource "aws_instance" "srv" {
  for_each      = var.servers
  ami           = data.aws_ssm_parameter.al2023.value
  instance_type = each.value.instance_type
  subnet_id     = data.aws_subnets.default.ids[each.value.az_index]
  tags          = { Name = each.key }
}
```

## 3.4 `count` vs `for_each` ⭐ (classic viva)

| | `count` | `for_each` |
|---|---|---|
| Input | Whole number | Map or set of strings |
| Identity | By **index** (0,1,2) | By **key** ("logs") |
| Remove an item from the middle | Indexes shift → Terraform may **destroy/recreate** later items ⚠️ | Only that key is removed ✅ |
| Best for | Identical copies, on/off switch (`0/1`) | Distinct named resources |
| Reference | `aws_instance.web[0]` | `aws_s3_bucket.this["logs"]` |

**Index-shift trap example:** list `["a","b","c"]` with `count`; remove `"a"` → `b` becomes index 0 and `c` index 1 → Terraform thinks resource 0 changed from a to b, and destroys index 2. With `for_each` keys stay stable.

## 3.5 `dynamic` blocks (generate repeated nested blocks)

```hcl
variable "ingress_ports" { default = [80, 443, 8080] }

resource "aws_security_group" "web" {
  name   = "web-sg"
  vpc_id = data.aws_vpc.default.id

  dynamic "ingress" {
    for_each = var.ingress_ports
    content {
      from_port   = ingress.value
      to_port     = ingress.value
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    }
  }
}
```

---

# 4️⃣ `lifecycle` META-ARGUMENT ⭐

```hcl
resource "aws_instance" "web" {
  # ...
  lifecycle {
    create_before_destroy = true       # build the replacement first, then delete old (zero downtime pattern)
    prevent_destroy       = true       # error on any plan that would destroy this (protect DBs/buckets)
    ignore_changes        = [tags, ami]# don't react to changes made outside Terraform to these attributes
    replace_triggered_by  = [aws_launch_template.lt.latest_version]   # force replace when another resource changes

    precondition {                     # custom check BEFORE creating
      condition     = contains(["t3.micro","t3.small"], var.instance_type)
      error_message = "Unsupported instance type."
    }
    postcondition {                    # check AFTER creating
      condition     = self.public_ip != ""
      error_message = "Instance must have a public IP."
    }
  }
}
```

| Setting | When to use |
|---|---|
| `create_before_destroy` | Replacements that must not cause downtime |
| `prevent_destroy` | Critical stateful resources; the guard rail against `terraform destroy` accidents |
| `ignore_changes` | Attributes managed by something else (autoscaling desired capacity, external tag tools) |
| `replace_triggered_by` | Cascading replacement |
| pre/postconditions | Validate assumptions |

Other meta-arguments: `provider`, `depends_on`, `count`, `for_each`.

---

# 5️⃣ MODULES ⭐⭐⭐ (REUSABLE BUILDING BLOCKS)

> A **module** is a folder of Terraform files used as a unit. The folder you run Terraform in is the **root module**; modules it calls are **child modules**. Modules are Terraform's version of functions/packages: define once, reuse many times.

CloudFormation parallel: nested stacks / modules.

## 5.1 Module structure

```
project/
├── main.tf                 # root: calls modules
├── variables.tf
├── outputs.tf
└── modules/
    └── web_server/
        ├── main.tf         # resources
        ├── variables.tf    # module inputs
        └── outputs.tf      # module outputs
```

## 5.2 Child module: `modules/web_server`

`variables.tf`:

```hcl
variable "name"          { type = string }
variable "instance_type" {
  type    = string
  default = "t3.micro"
}
variable "vpc_id"        { type = string }
variable "subnet_id"     { type = string }
variable "ami_id"        { type = string }
variable "ssh_cidr"      { type = string }
variable "key_name" {
  type    = string
  default = null
}
```

`main.tf`:

```hcl
resource "aws_security_group" "this" {
  name        = "${var.name}-sg"
  description = "HTTP from anywhere, SSH from allowed CIDR"
  vpc_id      = var.vpc_id

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = [var.ssh_cidr]
  }
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
  tags = { Name = "${var.name}-sg" }
}

resource "aws_instance" "this" {
  ami                         = var.ami_id
  instance_type               = var.instance_type
  subnet_id                   = var.subnet_id
  vpc_security_group_ids      = [aws_security_group.this.id]
  key_name                    = var.key_name
  associate_public_ip_address = true

  user_data = <<-EOF
    #!/bin/bash
    dnf install -y nginx
    systemctl enable --now nginx
    echo "<h1>${var.name} deployed by Terraform module</h1>" > /usr/share/nginx/html/index.html
  EOF

  tags = { Name = var.name }
}
```

(HCL note: a block written on one line may hold only ONE argument, so multi-argument blocks are written one argument per line. Commas on one line are only valid for object/map **values**.)

`outputs.tf`:

```hcl
output "instance_id" {
  value = aws_instance.this.id
}
output "public_ip" {
  value = aws_instance.this.public_ip
}
output "sg_id" {
  value = aws_security_group.this.id
}
```

## 5.3 Root module calling it twice

```hcl
module "web_dev" {
  source        = "./modules/web_server"
  name          = "web-dev"
  instance_type = "t3.micro"
  vpc_id        = data.aws_vpc.default.id
  subnet_id     = data.aws_subnets.default.ids[0]
  ami_id        = data.aws_ssm_parameter.al2023.value
  ssh_cidr      = var.my_ip_cidr
}

module "web_test" {
  source        = "./modules/web_server"
  name          = "web-test"
  instance_type = "t3.small"
  vpc_id        = data.aws_vpc.default.id
  subnet_id     = data.aws_subnets.default.ids[0]
  ami_id        = data.aws_ssm_parameter.al2023.value
  ssh_cidr      = var.my_ip_cidr
}

output "dev_ip" {
  value = module.web_dev.public_ip
}
output "test_ip" {
  value = module.web_test.public_ip
}
```

After adding a module (or changing its source): run `terraform init` again.

## 5.4 Module sources

| Source | Example |
|---|---|
| Local path | `./modules/web_server` |
| **Terraform Registry** | `source = "terraform-aws-modules/vpc/aws"` with `version = "~> 5.0"` (check current version) |
| Git | `git::https://github.com/org/repo.git//modules/web?ref=v1.2.0` |
| S3 / HTTP | archive URLs |

Registry example (community VPC module):

```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"

  name = "lab-vpc"
  cidr = "10.0.0.0/16"
  azs             = ["ap-south-1a", "ap-south-1b"]
  public_subnets  = ["10.0.1.0/24", "10.0.2.0/24"]
  private_subnets = ["10.0.11.0/24", "10.0.12.0/24"]
  enable_nat_gateway = false      # NAT costs money; keep off in labs
}
```

## 5.5 Module best practices

```
Small, focused, single-purpose modules      Version-pin (tags/version constraints)
Expose only necessary inputs and outputs    Document variables (description, type)
Don't hard-code provider config inside modules (pass providers from root)
Test modules (terraform test / Terratest)   Avoid deep nesting
```

---

# 6️⃣ ENVIRONMENTS: WORKSPACES vs FOLDERS vs VARFILES

| Approach | How | Pros | Cons |
|---|---|---|---|
| **Workspaces** | `terraform workspace new dev` — same code, separate state per workspace (`terraform.workspace` available in code) | Quick, little duplication | Same backend/credentials; easy to mix up; not strong isolation for prod |
| **Separate folders/stacks per environment** | `envs/dev`, `envs/prod` each with own backend key and tfvars, calling shared modules | Strong isolation, per-env settings | Some duplication |
| **tfvars files** | `terraform apply -var-file=prod.tfvars` (+ different backend config) | Simple | Must pair with separate state |

```bash
terraform workspace list            # * default
terraform workspace new dev
terraform workspace select prod
terraform workspace show
```

Use in code:

```hcl
locals { instance_type = terraform.workspace == "prod" ? "t3.large" : "t3.micro" }
```

Recommendation: workspaces for lightweight, temporary copies; **separate state per environment with shared modules** for real prod/non-prod separation.

---

# 7️⃣ PROVISIONERS (LAST RESORT) ⚠️

Provisioners run scripts on a resource after creation: `local-exec` (on your machine) or `remote-exec` (on the target via SSH), plus `file`.

```hcl
resource "aws_instance" "web" {
  # ...
  provisioner "local-exec" {
    command = "echo ${self.public_ip} >> inventory.txt"
  }
}
```

HashiCorp advises: **avoid provisioners when possible.** Reasons: not declarative, not tracked in plan, failures leave tainted resources, require network access/SSH keys.
Prefer: `user_data`/cloud-init, pre-baked images (Packer), Ansible after Terraform, SSM, or containers. Use `terraform_data` or `null_resource` only as a workaround.

---

# 8️⃣ IMPORT, MOVED, REMOVED 🔁

## 8.1 Import: bring existing (ClickOps) resources under Terraform

Modern way: `import` block (Terraform 1.5+):

```hcl
import {
  to = aws_s3_bucket.legacy
  id = "my-existing-bucket-name"
}

resource "aws_s3_bucket" "legacy" {
  bucket = "my-existing-bucket-name"
}
```

```bash
terraform plan -generate-config-out=generated.tf     # Terraform can DRAFT the resource code for you
terraform plan
terraform apply
```

Old CLI way: `terraform import aws_s3_bucket.legacy my-existing-bucket-name` (then write matching code).

## 8.2 `moved` block (rename/refactor without destroying)

```hcl
moved {
  from = aws_instance.web
  to   = aws_instance.frontend
}
```

Also used when moving a resource into a module.

## 8.3 `removed` block (stop managing, keep the real resource)

```hcl
removed {
  from = aws_s3_bucket.legacy
  lifecycle { destroy = false }
}
```

---

# 9️⃣ DRIFT HANDLING AND REPLACEMENT

```bash
terraform plan                        # compares code ↔ state ↔ reality; shows drift as changes
terraform plan -refresh-only          # show only drift (what changed outside Terraform)
terraform apply -refresh-only         # accept reality into state (no infra change)
terraform apply -replace="aws_instance.web"   # force re-create one resource (modern replacement for `taint`)
```

Decision on drift:

```
Drift is unwanted → just apply (Terraform reverts it)
Drift is desired  → update the code to match, then plan shows no changes
```

---

# 🔟 TESTING AND QUALITY ✅

| Tool/Command | Purpose |
|---|---|
| `terraform fmt -check` | Style check in CI |
| `terraform validate` | Syntax/consistency |
| **TFLint** | Provider-aware linting (invalid instance types, etc.) |
| **Checkov / Trivy (tfsec) / KICS** | Security & compliance scanning of `.tf` |
| **`terraform test`** (1.6+) | Native tests in `*.tftest.hcl` with `run` blocks and `assert` conditions (can use `command = plan` for cheap tests) |
| **Terratest** | Go-based integration tests that deploy real infra |
| **Infracost** | Cost estimate in pull requests |
| **OPA / Conftest / Sentinel** | Policy as code on plan JSON (`terraform show -json tfplan`) |

Example `terraform test` file:

```hcl
# tests/bucket.tftest.hcl
run "bucket_name_has_prefix" {
  command = plan
  assert {
    condition     = startswith(aws_s3_bucket.demo.bucket, "tf-demo-")
    error_message = "Bucket must start with tf-demo-"
  }
}
```

---

# 1️⃣1️⃣ CI/CD WORKFLOW FOR TERRAFORM 🔄

```
Pull Request:
  terraform fmt -check → validate → tflint → checkov → terraform plan -out=tfplan → comment plan on PR
Merge to main:
  terraform apply tfplan   (prod requires manual approval)
```

GitHub Actions skeleton:

```yaml
name: terraform
on: { pull_request: {}, push: { branches: [main] } }
permissions: { id-token: write, contents: read }     # OIDC → no long-lived AWS keys
jobs:
  tf:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/terraform-ci
          aws-region: ap-south-1
      - run: terraform init
      - run: terraform fmt -check
      - run: terraform validate
      - run: terraform plan -out=tfplan
      - if: github.ref == 'refs/heads/main'
        run: terraform apply -auto-approve tfplan
```

(Jenkins version appears in Unit III. Platforms like Atlantis, Spacelift, env0, Terraform Cloud/HCP Terraform automate this too.)

---

# 1️⃣2️⃣ SECRETS AND SECURITY 🔐

```
✔ Credentials via IAM roles/OIDC, not keys in code
✔ Mark variables sensitive = true (hides from CLI output)
✔ But STATE still contains values → encrypted, private, access-controlled state
✔ Don't output secrets; fetch at runtime from AWS Secrets Manager / SSM Parameter Store
✔ Ephemeral values / write-only arguments exist in newer Terraform versions to keep secrets out of state (check current docs)
✔ Least-privilege IAM for the pipeline role
✔ Scan IaC for open security groups, public buckets, unencrypted volumes
```

---

# 1️⃣3️⃣ HCP TERRAFORM (TERRAFORM CLOUD) AWARENESS

Managed platform by HashiCorp/IBM: remote state + locking, remote runs, VCS integration, policy (Sentinel), private registry, SSO, run history. Plan/pricing details have changed over time (including free-tier changes), so check the current pricing page rather than memorizing it. Alternatives: Spacelift, env0, Scalr, Atlantis (open-source), plain CI + S3 backend.

---

# 1️⃣4️⃣ COMMON ERRORS TABLE 🛠️

| Error | Cause | Fix |
|---|---|---|
| `Error: Backend initialization required` | Backend changed | `terraform init -reconfigure` or `-migrate-state` |
| `Error acquiring the state lock` | Concurrent run / stale lock | Wait; `force-unlock` if sure |
| `Variables not allowed` in backend | Used `var.` in backend block | Hard-code or `-backend-config` |
| `The "count" value depends on resource attributes that cannot be determined until apply` | count based on unknown value | Use known values/`for_each` with static keys; apply in stages |
| `Invalid for_each argument ... set includes values derived from resource attributes` | Keys unknown at plan | Use static keys |
| Instances destroyed after list edit | `count` index shift | Switch to `for_each` |
| `Module not installed` | Need init | `terraform init` |
| `prevent_destroy` blocking | Guard rail triggered | Remove only when truly intended |
| `Error: Provider produced inconsistent result` | Provider bug/version issue | Upgrade provider, retry |
| Import mismatch causes planned changes | Code differs from real settings | Adjust code until plan is clean |
| Cycle error in modules | Output/Input circular reference | Restructure dependencies |

---

# 1️⃣5️⃣ VIVA QUESTIONS (PART 4) 🎯

**Q1. Why use remote state?** Shared source of truth, locking, backups/versioning, security, team collaboration.

**Q2. How is state locking done with S3?** With `use_lockfile = true` (Terraform 1.10+), a lock object is written next to the state; older setups used a DynamoDB table.

**Q3. Why can't backend blocks use variables?** The backend is initialized before variables are evaluated.

**Q4. `count` vs `for_each`?** count uses numeric indexes and can shift on removal; for_each uses stable keys.

**Q5. What is a module?** A reusable folder of Terraform configuration called with a `module` block; root vs child modules.

**Q6. What does `prevent_destroy` do?** Makes Terraform error on any plan that would destroy that resource.

**Q7. What does `create_before_destroy` do?** Creates the replacement before deleting the old resource to avoid downtime.

**Q8. `terraform import` use?** Bring existing resources under Terraform management.

**Q9. How do you rename a resource without recreating it?** Use a `moved` block or `terraform state mv`.

**Q10. What are workspaces?** Multiple state instances of the same configuration, switched with `terraform workspace`.

**Q11. Why avoid provisioners?** They are imperative, hidden from plans, brittle; use user data, images or config management.

**Q12. How do you handle drift?** `plan`/`-refresh-only` to detect; either apply to revert or update code to accept.

**Q13. How to keep secrets out of Git and logs?** Env vars/roles/secret managers, sensitive variables, `.gitignore` state and tfvars with secrets, protect state.

**Q14. What does `terraform apply -replace` do?** Forces re-creation of a specific resource.

---

# 🧠 PART 4 MASTER SHEET

```
STATE: mapping code↔real resources ; remote + locked + encrypted + versioned for teams ; never edit/commit
S3 backend: bucket, key, region, encrypt, use_lockfile=true (TF ≥1.10) ; DynamoDB locking deprecated
Backend can't use variables ; init -migrate-state to move local→remote
state cmds: list, show, mv, rm, pull, push ; force-unlock only if stuck
count (index, can shift, 0/1 toggle) vs for_each (keys, stable) ; dynamic blocks for repeated nested blocks
lifecycle: create_before_destroy, prevent_destroy, ignore_changes, replace_triggered_by, pre/postcondition
Modules: root vs child ; inputs(variable) / outputs ; sources: local, registry, git ; pin versions
Environments: workspaces (light) vs separate state/folders (strong)
Provisioners = last resort ; prefer user_data/Packer/Ansible
import block, moved block, removed block ; plan -refresh-only ; apply -replace
Quality: fmt, validate, tflint, checkov/trivy, terraform test, Terratest, Infracost, OPA/Sentinel
CI/CD: PR plan → review → merge apply (OIDC roles, approvals)
```

---

## 🫡 PART 4 STATUS

You can manage Terraform like a team: remote locked state, modules, loops, lifecycle guards, imports, and CI workflows.

**STOP here.** Next up:

> **PART 5 — Terraform vs CloudFormation (full comparison) + Bicep/ARM, Ansible, Pulumi/CDK landscape ⚖️**
