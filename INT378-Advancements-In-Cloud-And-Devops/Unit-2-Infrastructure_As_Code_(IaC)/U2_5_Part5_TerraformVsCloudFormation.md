# ⚖️ UNIT II — PART 5: TERRAFORM vs CLOUDFORMATION (FULL COMPARISON) + THE WIDER IaC LANDSCAPE

> **Covered in this document:** why comparison questions are asked → same infrastructure written in both tools side by side → concept-by-concept mapping table (template/stack ↔ config/state, change set ↔ plan, drift detection ↔ refresh, nested stacks ↔ modules...) → the big comparison table (language, scope, state, rollback, speed of new features, modularity, ecosystem, cost, learning curve, licensing) → workflow comparison (create/update/delete/rollback/drift) → rollback philosophy difference (CloudFormation auto-rollback vs Terraform partial apply) → state management difference → multi-cloud and ecosystem → when to choose which (decision table with scenarios) → using both together → comparison with Azure ARM/Bicep, Ansible, Pulumi, AWS CDK → exam-ready 10-mark answer → practical comparison checklist (for Practical 3) → viva → master sheet.

---

Absolutely mate. 😤🔥 Your syllabus line: **"Comparison of Terraform and CloudFormation."** Examiners love this one. This file gives you a tight, structured, evidence-style answer plus the real hands-on comparison needed in Practical 3.

---

# 1️⃣ ONE-LINE SUMMARY

```
CloudFormation = AWS's own, AWS-only, managed-state, automatic-rollback IaC service (YAML/JSON)
Terraform      = HashiCorp's multi-cloud IaC tool (HCL), you manage the state, rich ecosystem, plan-first workflow
```

---

# 2️⃣ SAME INFRASTRUCTURE, TWO TOOLS (SIDE BY SIDE) 👀

Goal: **a security group allowing HTTP + one EC2 instance using it.**

## 2.1 CloudFormation (YAML)

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Parameters:
  VpcId:    { Type: AWS::EC2::VPC::Id }
  SubnetId: { Type: AWS::EC2::Subnet::Id }
  InstanceType:
    Type: String
    Default: t3.micro
  LatestAmiId:
    Type: AWS::SSM::Parameter::Value<AWS::EC2::Image::Id>
    Default: /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64
Resources:
  WebSG:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Allow HTTP
      VpcId: !Ref VpcId
      SecurityGroupIngress:
        - { IpProtocol: tcp, FromPort: 80, ToPort: 80, CidrIp: 0.0.0.0/0 }
  Web:
    Type: AWS::EC2::Instance
    Properties:
      ImageId: !Ref LatestAmiId
      InstanceType: !Ref InstanceType
      SubnetId: !Ref SubnetId
      SecurityGroupIds: [!Ref WebSG]
Outputs:
  PublicIP: { Value: !GetAtt Web.PublicIp }
```

## 2.2 Terraform (HCL)

```hcl
variable "instance_type" {
  type    = string
  default = "t3.micro"
}

data "aws_vpc" "default" {
  default = true
}
data "aws_subnets" "default" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.default.id]
  }
}
data "aws_ssm_parameter" "ami" {
  name = "/aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64"
}

resource "aws_security_group" "web" {
  name   = "web-sg"
  vpc_id = data.aws_vpc.default.id
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_instance" "web" {
  ami                         = data.aws_ssm_parameter.ami.value
  instance_type               = var.instance_type
  subnet_id                   = data.aws_subnets.default.ids[0]
  vpc_security_group_ids      = [aws_security_group.web.id]
  associate_public_ip_address = true
}

output "public_ip" {
  value = aws_instance.web.public_ip
}
```

## 2.3 What to notice

```
• Both are DECLARATIVE and build a dependency graph from references (!Ref vs aws_security_group.web.id)
• CloudFormation: parameters by type (VPC::Id dropdown) ; Terraform: data sources look up the default VPC
• CloudFormation has no state file for you to manage; Terraform has terraform.tfstate
• Terraform needs 'init' (downloads provider) ; CloudFormation needs nothing installed (just AWS CLI/console)
• Terraform HCL is slightly more concise and supports loops/expressions natively
```

---

# 3️⃣ CONCEPT MAPPING TABLE ⭐⭐⭐ (REVISION GOLD)

| Idea | CloudFormation | Terraform |
|---|---|---|
| Definition file | **Template** (YAML/JSON) | **Configuration** (`.tf`, HCL) |
| Deployed unit | **Stack** | **Working directory + state** (or workspace) |
| Inputs | `Parameters` | `variable` |
| Outputs | `Outputs` (+ `Export`) | `output` |
| Internal named values | `Mappings`, `Fn::Sub` | `locals` |
| Conditions | `Conditions` + `Fn::If` | `count`/ternary `? :` |
| Loops | `Fn::ForEach` (LanguageExtensions) / limited | `count`, `for_each`, `for` expressions, `dynamic` |
| Read existing info | SSM parameter types, `Fn::ImportValue`, pseudo params | `data` sources, `terraform_remote_state` |
| Reference | `!Ref`, `!GetAtt` | `type.name.attribute` |
| Preview changes | **Change set** | **`terraform plan`** |
| Apply | `create-stack` / `deploy` / `execute-change-set` | `terraform apply` |
| Delete | `delete-stack` | `terraform destroy` |
| State | Managed by AWS service (no file for you) | **State file** you must store (local/S3/HCP) |
| Drift detection | `detect-stack-drift` (report only) | `terraform plan` / `plan -refresh-only` |
| Reuse | Nested stacks, StackSets, modules (registry) | **Modules** (+ Registry) |
| Multi-account/region rollout | **StackSets** | Provider aliases, workspaces, pipelines, HCP Terraform |
| Import existing resources | Resource import / IaC generator | `import` block, `terraform import`, `-generate-config-out` |
| Protect from deletion | `DeletionPolicy`, termination protection, stack policy | `lifecycle { prevent_destroy }` |
| Dependencies | Implicit + `DependsOn` | Implicit + `depends_on` |
| Bootstrap on EC2 | UserData, cfn-init, cfn-signal | `user_data`, provisioners (discouraged) |
| Lint/Policy | cfn-lint, cfn-guard, Hooks | tflint, Checkov/Trivy, Sentinel, OPA |
| Testing | TaskCat, cfn-lint, manual | `terraform test`, Terratest |

---

# 4️⃣ THE BIG COMPARISON TABLE ⭐⭐⭐ (WRITE THIS IN EXAMS)

| Aspect | **AWS CloudFormation** | **Terraform** |
|---|---|---|
| **Vendor / origin** | AWS (launched 2011) | HashiCorp, now IBM (first release 2014) |
| **Language** | JSON / YAML (+ intrinsic functions) | HCL (also JSON syntax) |
| **Paradigm** | Declarative | Declarative |
| **Cloud support** | AWS-centric (third-party types via registry) | **Multi-cloud**: AWS, Azure, GCP, Kubernetes, GitHub, Datadog... via providers |
| **State management** | **Managed by AWS** automatically (stack) | **You manage** state file; remote backend + locking recommended |
| **Preview** | Change sets (optional step) | `plan` is a core, always-used step |
| **Rollback on failure** | **Automatic rollback** to last good state | **No automatic rollback**: partial apply may leave some resources created; fix and re-apply or destroy |
| **Drift detection** | Built-in detection (reports) | Detected during `plan`/refresh (and fixed by apply) |
| **New AWS feature support** | Often day-one/very fast for AWS services (not always) | Slight lag (provider updates), usually fast; Cloud Control-based `awscc` provider also exists |
| **Modularity / reuse** | Nested stacks, modules, StackSets | Modules + large public registry |
| **Programming constructs** | Limited (conditions, mappings, some functions; macros for more) | Richer (loops, conditionals, expressions, functions) |
| **Ecosystem / community** | AWS docs, samples, SAM, CDK | Very large community, registry modules, many third-party tools |
| **Multi-account/region** | StackSets (native) | Provider aliases/pipelines/HCP |
| **Cost of the tool** | Free (pay for resources) | CLI free to use (BSL license); paid HCP Terraform/Enterprise optional |
| **Installation** | None (AWS service) | Install CLI + providers |
| **Learning curve** | Easy start for AWS users; verbose YAML | Readable HCL; extra concepts (state, backends, providers) |
| **Execution location** | Runs inside AWS (service-side) | Runs on your machine/CI, calling APIs |
| **Permissions model** | CloudFormation can assume a service role | The caller's credentials/role |
| **Locking/concurrency** | Stack-level operations serialized by service | State locking via backend |
| **Import of existing resources** | Resource import, IaC generator | `import` block / CLI |
| **Vendor lock-in** | Tied to AWS | Portable skills across clouds (resources themselves still cloud-specific) |
| **Licensing** | Proprietary service | BSL 1.1 since 2023 (fork: OpenTofu, MPL) |
| **Best fit** | AWS-only, deep AWS integration, managed state, compliance via AWS tooling | Multi-cloud/hybrid, complex logic, rich module ecosystem |

⚠️ Note for accuracy: statements like "Terraform lags new AWS features" are **tendencies, not guarantees**. Say "often/typically".

---

# 5️⃣ WORKFLOW COMPARISON (LIFECYCLE) 🔄

```
CLOUDFORMATION                                   TERRAFORM
────────────────────────────────                 ──────────────────────────────
write template.yaml                              write *.tf
aws cloudformation validate-template             terraform init  (providers)
(aws cloudformation deploy / create-stack)       terraform validate
  └─ optional: create-change-set → review        terraform plan   (always)
  └─ execute-change-set                          terraform apply
stack events → CREATE_COMPLETE                   state updated → "Apply complete!"
update template → change set → execute           edit .tf → plan → apply
detect-stack-drift                               terraform plan (shows drift)
delete-stack                                     terraform destroy
```

---

# 6️⃣ ROLLBACK PHILOSOPHY ⭐⭐ (A FAVOURITE EXAM DIFFERENCE)

## CloudFormation: transactional-style

```
Create/update fails at resource 5 of 8
   → CloudFormation automatically reverts:
       create failure → deletes what it built → ROLLBACK_COMPLETE
       update failure → restores previous config → UPDATE_ROLLBACK_COMPLETE
You end up at a known-good state without doing anything.
```

## Terraform: no automatic rollback

```
apply fails at resource 5 of 8
   → resources 1-4 STAY created and are recorded in state; 5 failed; 6-8 not attempted
You then: fix the cause and run apply again (it continues), OR change code back and apply (a "rollback" by applying the previous version from Git), OR destroy.
```

Terraform "rollback" in practice = **revert the commit in Git and apply** (works because IaC + Git history).

| | CloudFormation | Terraform |
|---|---|---|
| Failure behavior | Auto-revert | Partial infrastructure remains |
| Operator effort | Low | Medium (re-apply or revert) |
| Visibility | Stack events | Error output + plan |
| Risk | Rollback can itself fail (`UPDATE_ROLLBACK_FAILED`) | Half-applied states need attention |

---

# 7️⃣ STATE DIFFERENCE ⭐

| | CloudFormation | Terraform |
|---|---|---|
| Who stores state | AWS (hidden inside the service) | You: local file or remote backend |
| Risk | Almost none | Lost/corrupt/leaked state |
| Flexibility | Low (can't inspect/modify easily) | High (`state` commands, `mv`, `rm`, `import`) |
| Collaboration | Built-in | Needs remote backend + lock |
| Secrets exposure | Parameters can be `NoEcho`; use dynamic references | Secrets can be in state; protect state |

```
CloudFormation: "Trust AWS to remember."
Terraform:      "You are responsible for remembering, but you get full control."
```

---

# 8️⃣ MULTI-CLOUD AND ECOSYSTEM

```
Terraform example: one repo manages
   AWS (VPC, EC2, S3) + Azure (resource group, AKS) + Cloudflare (DNS) + GitHub (repos) + Datadog (monitors)
   with the same HCL and same init/plan/apply workflow.

CloudFormation: AWS resources (plus registry extensions); other clouds need other tools.
```

Terraform's Registry has thousands of providers and many popular modules (e.g., `terraform-aws-modules/*`). CloudFormation has AWS-published samples, SAM, and the CDK ecosystem.

---

# 9️⃣ DECISION TABLE: WHICH ONE FOR WHICH SITUATION? 🎯

| Scenario | Better choice | Why |
|---|---|---|
| Company is **100% AWS**, wants minimal tooling and AWS-managed state | **CloudFormation** | No state to host, native rollback, StackSets, deep AWS integration |
| Need to manage **AWS + Azure + GCP** or SaaS (GitHub/Datadog) | **Terraform** | One tool, many providers |
| Heavy use of **AWS Service Catalog, Control Tower, Organizations StackSets** | CloudFormation | Native integrations |
| Complex logic (many loops/conditions, big module library) | Terraform | Richer language and modules |
| Team wants to use **TypeScript/Python** for infra on AWS | **AWS CDK** (compiles to CloudFormation) or Pulumi | Real programming languages |
| Strict "AWS-supported only" policy | CloudFormation | First-party support |
| Avoid vendor lock-in for **skills** | Terraform/OpenTofu | Portable workflow |
| Small team, wants safety net on failed deploys | CloudFormation | Auto rollback |
| Open-source licensing concerns | **OpenTofu** | MPL 2.0, Linux Foundation |
| Brand-new AWS service needed immediately | Check both; CloudFormation often first, but verify docs/provider | Coverage varies |

## 9.1 Using both together (very common)

```
CloudFormation for the AWS "landing zone"/account baseline (Control Tower, StackSets)
Terraform for application and multi-cloud infrastructure
Or: Terraform's aws_cloudformation_stack resource to deploy a CFN template from Terraform
```

---

# 🔟 THE WIDER LANDSCAPE: WHERE ELSE DOES IaC LIVE? 🌐

## 10.1 Azure ARM templates and Bicep

```bicep
// Bicep: Azure's friendlier language that compiles to ARM JSON
param location string = resourceGroup().location
param storageName string = 'st${uniqueString(resourceGroup().id)}'

resource sa 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  name: storageName
  location: location
  sku: { name: 'Standard_LRS' }
  kind: 'StorageV2'
}

output storageId string = sa.id
```

```bash
az deployment group create --resource-group rg-demo --template-file main.bicep
az deployment group what-if  --resource-group rg-demo --template-file main.bicep    # = change set / plan
```

| | CloudFormation | ARM/Bicep |
|---|---|---|
| Cloud | AWS | Azure |
| State | Managed by service | Managed by Azure Resource Manager (no state file) |
| Preview | Change set | `what-if` |
| Rollback | Automatic | Rollback-on-error option available |
| Scope | Stack | Resource group / subscription / management group deployment |

## 10.2 Ansible (configuration management and also provisioning)

```yaml
- hosts: webservers
  become: true
  tasks:
    - name: install nginx
      ansible.builtin.package: { name: nginx, state: present }
    - name: start nginx
      ansible.builtin.service: { name: nginx, state: started, enabled: true }
```

Agentless (SSH), YAML playbooks, idempotent modules. Strength: configuring servers/apps. Terraform provisions → Ansible configures.

## 10.3 Pulumi

Define infra in **Python/TypeScript/Go/C#/Java**:

```python
import pulumi, pulumi_aws as aws
bucket = aws.s3.Bucket("demo", versioning={"enabled": True})
pulumi.export("bucket", bucket.id)
```

Multi-cloud; state in Pulumi Cloud or self-managed backend; real loops, functions, unit tests.

## 10.4 AWS CDK (Cloud Development Kit)

Write in TypeScript/Python/Java/C#/Go; `cdk synth` generates CloudFormation; `cdk deploy` creates a stack. You get CloudFormation's rollback and state with a real programming language.

```python
from aws_cdk import Stack, aws_s3 as s3
class DemoStack(Stack):
    def __init__(self, scope, id, **kw):
        super().__init__(scope, id, **kw)
        s3.Bucket(self, "Demo", versioned=True)
```

## 10.5 Quick landscape table

| Tool | Language | Scope | State | Style |
|---|---|---|---|---|
| CloudFormation | YAML/JSON | AWS | AWS-managed | Declarative |
| Terraform / OpenTofu | HCL | Multi-cloud | You manage | Declarative |
| Bicep/ARM | Bicep/JSON | Azure | Azure-managed | Declarative |
| Pulumi | Python/TS/Go/C#/Java | Multi-cloud | Pulumi backend | Code (declarative engine) |
| AWS CDK | TS/Python/Java/C#/Go | AWS | Via CloudFormation | Code → CFN |
| Ansible | YAML | Servers/apps (+cloud modules) | Stateless | Procedural-ish/idempotent |
| Crossplane | K8s YAML (CRDs) | Multi-cloud via Kubernetes | In K8s (etcd) | Declarative, control plane |
| Packer | HCL/JSON | Image building | — | Build |

---

# 1️⃣1️⃣ MODEL 10-MARK ANSWER ✍️

**Q. Compare Terraform and AWS CloudFormation.**

```
INTRODUCTION
Both are declarative Infrastructure as Code tools that provision infrastructure from text definitions.
CloudFormation is AWS's native service; Terraform is HashiCorp's multi-cloud tool.

TABLE (write 8-10 rows from Section 4):
Language | Cloud support | State | Preview | Rollback | Drift | Modularity | Ecosystem | Cost/licensing | Learning curve

KEY DIFFERENCES (2-3 lines each)
1. State: CFN managed by AWS; Terraform needs remote state with locking.
2. Rollback: CFN automatic; Terraform requires re-apply/revert in Git.
3. Scope: CFN AWS only; Terraform multi-cloud through providers.
4. Language: YAML/JSON with intrinsic functions vs HCL with loops, expressions, modules.

DIAGRAM (small): code → [CFN service | Terraform core + provider] → AWS APIs

WHEN TO USE: AWS-only/managed-state → CFN; multi-cloud/complex logic → Terraform.
CONCLUSION: Choose by cloud strategy, team skills, and governance; they can coexist.
```

---

# 1️⃣2️⃣ PRACTICAL COMPARISON CHECKLIST (FOR PRACTICAL 3 REPORT) 📋

When you deploy the **same** EC2 + SG + S3 in both tools, record:

| Metric | CloudFormation | Terraform |
|---|---|---|
| Lines of code | ____ | ____ |
| Tools to install | AWS CLI only | Terraform CLI + provider download |
| Command used to create | `deploy` / `create-stack` | `init` + `apply` |
| Time to create (seconds) | ____ | ____ |
| Preview feature used | Change set (screenshot) | `plan` output (screenshot) |
| Where state/record is visible | Console: stack events, resources | `terraform.tfstate`, `state list` |
| How update was done | update-stack / change set | edit + `apply` |
| How failure/rollback behaved | `UPDATE_ROLLBACK_COMPLETE` | partial apply message / revert |
| Drift check | `detect-stack-drift` | `plan` after manual change |
| Cleanup | `delete-stack` | `destroy` |
| Readability opinion | ____ | ____ |

Then write 5-6 lines of conclusion using this table (Part 7 lab explains how to produce each row).

---

# 1️⃣3️⃣ COMMON MISCONCEPTIONS 🚨

```
✘ "Terraform is imperative"         → No, declarative (HCL describes desired state)
✘ "CloudFormation has no preview"   → It has change sets
✘ "Terraform has automatic rollback"→ No
✘ "Terraform works only on AWS"     → Multi-cloud
✘ "CloudFormation can manage Azure" → Not natively (AWS-centric)
✘ "Terraform state is optional"     → Core to how it works
✘ "Terraform is free for everything"→ CLI use is free, but license is BSL; commercial HCP features are paid; OpenTofu is the MPL alternative
✘ "CDK is a separate engine"        → CDK synthesizes CloudFormation templates
```

---

# 1️⃣4️⃣ VIVA QUESTIONS (PART 5) 🎯

**Q1. Main difference between Terraform and CloudFormation?** Terraform is multi-cloud with user-managed state; CloudFormation is AWS-only with AWS-managed state and automatic rollback.

**Q2. Equivalent of `terraform plan` in CloudFormation?** Change sets.

**Q3. Equivalent of Terraform modules?** Nested stacks (and CFN modules/registry); StackSets for multi-account.

**Q4. Which supports automatic rollback?** CloudFormation.

**Q5. Where does each store state?** CFN in the AWS service; Terraform in a state file (local or remote).

**Q6. Which language does each use?** YAML/JSON vs HCL.

**Q7. How to detect drift in each?** CFN `detect-stack-drift`; Terraform `plan`/`-refresh-only`.

**Q8. What is AWS CDK?** Framework for defining AWS infrastructure in programming languages that synthesizes CloudFormation.

**Q9. What is OpenTofu?** Linux Foundation, MPL-licensed fork of Terraform created after the 2023 license change.

**Q10. What is Bicep?** Azure's concise language for ARM deployments.

**Q11. Can both be used together?** Yes; e.g., CFN for account baselines and Terraform for apps, or Terraform deploying a CFN stack.

**Q12. When choose CloudFormation?** AWS-only shops needing managed state, native rollback and StackSets.

**Q13. Why is Terraform popular?** Multi-cloud providers, readable HCL, plan-first workflow, big module ecosystem.

---

# 🧠 PART 5 MASTER SHEET

```
CFN: AWS-only | YAML/JSON | state managed by AWS | change sets | AUTO ROLLBACK | StackSets | free service
TF : multi-cloud | HCL | state you manage (remote+lock) | plan always | NO auto rollback | modules+registry | BSL license (OpenTofu = MPL fork)
Mapping: template↔config, stack↔state+dir, parameters↔variables, outputs↔outputs, change set↔plan,
         drift detect↔plan/refresh, nested stacks↔modules, !Ref/!GetAtt↔resource.attr, DeletionPolicy↔prevent_destroy
Choose: AWS-only+safety → CFN ; multi-cloud+complex logic → Terraform ; code languages → CDK/Pulumi ; Azure → Bicep ; configure servers → Ansible
Rollback: CFN reverts itself ; Terraform = re-apply or git revert
```

---

## 🫡 PART 5 STATUS

You can now give a structured, table-backed comparison of Terraform and CloudFormation, and place them in the wider IaC landscape.

**STOP here.** Next up:

> **PART 6 — Future Trends and Emerging Tools in IaC 🔮** (OpenTofu, Pulumi, CDK, Crossplane, policy as code, GitOps, platform engineering, AI-assisted IaC, drift automation, FinOps)
