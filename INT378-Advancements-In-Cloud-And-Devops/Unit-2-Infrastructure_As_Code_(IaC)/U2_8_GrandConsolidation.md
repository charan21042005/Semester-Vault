# 🏆 UNIT II — GRAND CONSOLIDATION

## Infrastructure as Code (IaC) — From Scratch → End-to-End

> **Covered in this document:** the whole unit in one sentence → the complete story → part-by-part revision (Parts 1-7) → master comparison tables (IaC concepts, CloudFormation vs Terraform, Terraform vs OpenTofu vs Pulumi vs CDK, declarative vs imperative, count vs for_each) → command cheat sheets (CloudFormation and Terraform) → decision tables → common errors table → IaC security checklist → lab ideas → 3 real-world scenarios → 30 best viva questions with answers → common traps → what to write (and not write) in an exam → master sheet → memory maps → how Unit II connects to Units I and III-VI.

---

# 🚀 THE ENTIRE UNIT II IN ONE SENTENCE

> **Infrastructure as Code replaces manual console clicking with versioned, reviewable, repeatable declarative code (AWS CloudFormation for AWS-native stacks with managed state and automatic rollback; Terraform for multi-cloud with providers, plan-first workflow and a state file you protect), compared by scope, state, rollback and ecosystem, while the field moves toward open forks, GitOps, policy as code, platform engineering and AI-assisted but human-governed automation.**

---

# 1️⃣ THE COMPLETE UNIT II STORY

```
PROBLEM    → Unit I's manual EC2/VPC clicking: slow, error-prone, drifting, undocumented       [Part 1]
IDEA       → Describe infrastructure in code: declarative, idempotent, version-controlled       [Part 1]
DEVOPS     → IaC puts infrastructure into the CI/CD pipeline (speed, consistency, audit, DR)    [Part 1]
CFN        → AWS-native: template → stack, change sets, rollback, drift, nested stacks         [Part 2]
TERRAFORM  → HCL + providers + state; init → plan → apply → destroy; variables/outputs/data    [Part 3]
TF ADVANCED→ remote S3 state + locking, modules, count/for_each, lifecycle, import, workspaces [Part 4]
COMPARE    → Same infra, two tools: state, rollback, scope, language, ecosystem                [Part 5]
FUTURE     → OpenTofu, Pulumi/CDK, Crossplane, GitOps, policy as code, platform eng., AI       [Part 6]
PRACTICE   → P3: CloudFormation lifecycle + rollback + drift ; P5: Terraform EC2+S3 + state    [Part 7]
```

---

# 2️⃣ PART-BY-PART REVISION

## Part 1: IaC from scratch
```
ClickOps problems: snowflakes, drift, slow, no audit, human error, no DR
IaC = infra defined in machine-readable files; code is the documentation
DevOps value: speed, consistency, Git history, review, CI/CD, DR, compliance, DORA, GitOps
Declarative (WHAT) vs imperative (HOW) ; idempotency ; desired vs actual state → DRIFT
Mutable vs immutable (pets vs cattle) ; push vs pull ; agent vs agentless
Provisioning (Terraform/CFN) vs configuration management (Ansible/Puppet/Chef) vs images (Packer)
Lifecycle: write → validate → plan → review → apply → monitor → destroy
Dependency graph = automatic ordering ; testing: fmt, validate, lint, scan, plan, tests, drift
```

## Part 2: CloudFormation
```
Template (YAML/JSON) → Stack → Resources ; AWS-managed state ; regional ; free service
Sections: Version, Description, Metadata, Parameters, Rules, Mappings, Conditions, Transform, Resources (REQUIRED), Outputs
Functions: Ref, GetAtt, Sub, Join, Select, Split, FindInMap, If, ImportValue, Base64, GetAZs
Pseudo params: AWS::Region, AccountId, StackName ; SSM parameter type for latest AMI
Statuses: CREATE_COMPLETE | ROLLBACK_COMPLETE | UPDATE_COMPLETE | UPDATE_ROLLBACK_COMPLETE | DELETE_FAILED
Update behavior: no interruption | some interruption | REPLACEMENT
Change set = preview ; drift detection = report ; automatic rollback ; DeletionPolicy/UpdateReplacePolicy
Reuse: nested stacks, Export/ImportValue, StackSets ; helper: UserData, cfn-init, cfn-signal ; CDK/SAM
```

## Part 3: Terraform basics
```
HashiCorp (IBM) | HCL | declarative | multi-cloud via providers | state | plan-first
Architecture: Core + Providers + State + Backend + Registry
Workflow: init → fmt → validate → plan → apply → destroy ; plan symbols + ~ - -/+ <=
Blocks: terraform, provider, resource, data, variable, output, locals, module
References type.name.attr → implicit dependency ; depends_on explicit
Variables precedence: default < TF_VAR_ < terraform.tfvars < *.auto.tfvars < -var / -var-file
State = mapping code ↔ real resources ; never edit/commit ; lock file .terraform.lock.hcl (commit)
Version constraints: = , >= , ~>
```

## Part 4: Terraform advanced
```
Remote state: S3 backend + encrypt + versioning + use_lockfile (TF ≥1.10; DynamoDB locking deprecated)
Backend can't use variables ; init -migrate-state ; force-unlock only if stuck
count (index, shift risk, 0/1 toggle) vs for_each (keys, stable) ; dynamic blocks
lifecycle: create_before_destroy, prevent_destroy, ignore_changes, replace_triggered_by, pre/postconditions
Modules: root/child, inputs/outputs, local/registry/git sources, version pinning
Environments: workspaces vs separate state ; provisioners = last resort
import / moved / removed blocks ; plan -refresh-only ; apply -replace
Quality: fmt, validate, tflint, checkov/trivy, terraform test, Terratest, Infracost, OPA/Sentinel
```

## Part 5: Comparison
```
CFN: AWS-only, managed state, change sets, AUTO ROLLBACK, StackSets, YAML/JSON
TF : multi-cloud, user-managed state, plan always, NO auto rollback, modules/registry, HCL, BSL license
Mapping: template↔config, stack↔state+dir, parameters↔variables, change set↔plan, nested stack↔module
Landscape: Bicep/ARM (Azure), Ansible (config), Pulumi, AWS CDK (→CFN), Crossplane, Packer
Choose: AWS-only+safety → CFN ; multi-cloud/complex logic → Terraform ; languages → CDK/Pulumi
```

## Part 6: Future trends
```
OpenTofu (MPL, Linux Foundation) vs Terraform (BSL, HashiCorp/IBM) • IaC in languages (Pulumi, CDK; CDKTF sunset Dec 2025)
• Kubernetes-native (Crossplane, ACK) • GitOps (Argo CD, Flux, Atlantis) • Policy as code (OPA, Sentinel, Checkov, Guard)
• Drift/continuous reconciliation • Platform engineering (Backstage, golden modules) • AI-assisted IaC (+ guard rails)
• FinOps (Infracost) • Immutable/ephemeral environments • Supply-chain security • State/secrets upgrades (S3 native lock, OIDC)
```

## Part 7: Practicals
```
P3: collect IDs → validate → create stack → verify → change set (t3.micro→t3.small) → add S3 → forced failure → UPDATE_ROLLBACK_COMPLETE
    → drift detect → delete stack → compare with Terraform (table)
P5: install → versions/variables/main/outputs → init → fmt/validate → plan → apply → verify → state list/show → modify → remote S3 state/lock → destroy
```

---

# 3️⃣ MASTER COMPARISON TABLES

## 3.1 Declarative vs Imperative

| | Declarative | Imperative |
|---|---|---|
| Describes | Desired end state | Step-by-step commands |
| Re-run safety | Idempotent | May duplicate/fail |
| Examples | Terraform, CloudFormation, Bicep, K8s YAML | Bash/AWS CLI scripts |
| Tool decides order? | Yes (dependency graph) | No, you do |

## 3.2 CloudFormation vs Terraform (top 12 rows)

| Aspect | CloudFormation | Terraform |
|---|---|---|
| Vendor | AWS | HashiCorp (IBM) |
| Language | YAML/JSON | HCL |
| Cloud scope | AWS-centric | Multi-cloud |
| State | AWS-managed | Your state file (remote + lock) |
| Preview | Change set | plan |
| Failure handling | Automatic rollback | No auto rollback |
| Drift | detect-stack-drift (report) | plan/refresh (and apply fixes) |
| Reuse | Nested stacks, StackSets | Modules, registry |
| Logic/loops | Limited (conditions, ForEach ext.) | Rich (count, for_each, for, dynamic) |
| Install | None | CLI + providers |
| Cost of tool | Free service | CLI free (BSL); HCP paid options |
| Best for | AWS-only, managed, compliance | Multi-cloud, modular, complex |

## 3.3 IaC tool families

| Family | Tools | Notes |
|---|---|---|
| Cloud-native DSL | CloudFormation, ARM/Bicep | One cloud, service-managed |
| Multi-cloud DSL | Terraform, OpenTofu | HCL + providers |
| Language-based | Pulumi, AWS CDK | Real languages |
| Config management | Ansible, Puppet, Chef, Salt | Configure servers |
| K8s-native | Crossplane, ACK, Helm/Kustomize, Argo CD/Flux | Control-plane/GitOps |
| Image build | Packer, EC2 Image Builder | Immutable images |
| Policy/security | OPA, Sentinel, Checkov, Guard, Hooks | Gate changes |

## 3.4 count vs for_each

| | count | for_each |
|---|---|---|
| Input | number | map/set(string) |
| ID | index | key |
| Remove middle item | shifts later items ⚠️ | stable ✅ |
| Use | N identical / on-off toggle | distinct named items |

## 3.5 Mutable vs immutable; push vs pull

| | Mutable | Immutable |
|---|---|---|
| Change method | In-place patch | Replace with new image |
| Drift risk | High | Low |
| Rollback | Hard | Switch back to old image |

| | Push | Pull |
|---|---|---|
| Model | Central tool applies | Agent fetches/reconciles |
| Examples | Terraform, Ansible | Puppet, Argo CD |

---

# 4️⃣ COMMAND CHEAT SHEETS ⌨️

## 4.1 CloudFormation

```bash
aws cloudformation validate-template --template-body file://t.yaml
aws cloudformation create-stack --stack-name S --template-body file://t.yaml --parameters ParameterKey=K,ParameterValue=V [--capabilities CAPABILITY_IAM]
aws cloudformation deploy --stack-name S --template-file t.yaml --parameter-overrides K=V
aws cloudformation create-change-set --stack-name S --change-set-name CS --template-body file://t.yaml
aws cloudformation describe-change-set --stack-name S --change-set-name CS
aws cloudformation execute-change-set --stack-name S --change-set-name CS
aws cloudformation update-stack --stack-name S --template-body file://t.yaml
aws cloudformation describe-stacks | describe-stack-events | list-stack-resources --stack-name S
aws cloudformation detect-stack-drift --stack-name S
aws cloudformation describe-stack-resource-drifts --stack-name S
aws cloudformation update-termination-protection --enable-termination-protection --stack-name S
aws cloudformation delete-stack --stack-name S
```

## 4.2 Terraform

```bash
terraform init [-upgrade | -migrate-state | -reconfigure]
terraform fmt [-check] ; terraform validate
terraform plan [-out=tfplan] [-var k=v] [-var-file=f] [-refresh-only] [-destroy]
terraform apply [tfplan] [-auto-approve] [-replace=ADDR] [-refresh-only]
terraform destroy
terraform output [-raw name | -json] ; terraform show
terraform state list | show ADDR | mv A B | rm ADDR | pull
terraform import ADDR ID   (or import block) ; terraform force-unlock ID
terraform workspace list|new|select|show
terraform console ; terraform graph ; terraform test ; terraform providers
```

---

# 5️⃣ DECISION TABLES 🎯

## 5.1 Which IaC tool?

| Situation | Choose |
|---|---|
| AWS-only, wants managed state + auto rollback | CloudFormation |
| Multi-cloud or SaaS providers | Terraform/OpenTofu |
| Prefer Python/TypeScript for AWS | AWS CDK (or Pulumi) |
| Azure-only | Bicep/ARM |
| Configure software inside servers | Ansible |
| Kubernetes platform with self-service infra | Crossplane + GitOps |
| Need open-source license guarantee | OpenTofu |
| Build golden images | Packer / EC2 Image Builder |

## 5.2 Which environment strategy (Terraform)?

| Need | Choose |
|---|---|
| Quick temporary copy of same stack | Workspace |
| Strong prod/non-prod isolation | Separate state/folders + shared modules |
| Many accounts/regions at scale | Pipeline/orchestrator (HCP Terraform, Spacelift, Atlantis) or StackSets for CFN |

## 5.3 How to handle drift?

| Drift is… | Action |
|---|---|
| Unwanted manual change | Re-apply (revert) |
| Desired change | Update code, re-plan to zero diff |
| Unknown cause | Investigate (CloudTrail), then decide |

---

# 6️⃣ COMMON ERRORS TABLE 🛠️

| Symptom | Tool | Cause | Fix |
|---|---|---|---|
| `ROLLBACK_COMPLETE` | CFN | Create failed | Read first `CREATE_FAILED`; delete stack; fix; retry |
| `UPDATE_ROLLBACK_COMPLETE` | CFN | Update failed | Fix change; update again |
| `InsufficientCapabilities` | CFN | IAM resources | `--capabilities CAPABILITY_IAM` |
| `DELETE_FAILED` (S3) | CFN | Bucket not empty | Empty then delete |
| `Template format error` | CFN | YAML/indent | cfn-lint, spaces |
| `No valid credential sources` | TF | No AWS creds | aws configure / role |
| `Inconsistent dependency lock file` | TF | Needs init | `terraform init` |
| `Unsupported argument` | TF | Typo/provider version | Check docs for your provider version |
| `Error acquiring the state lock` | TF | Concurrent/stale lock | Wait; `force-unlock` carefully |
| Unexpected `-/+` | TF | Changed immutable attribute | Review; revert or accept |
| Instances deleted after list edit | TF | `count` shift | Use `for_each` |
| `BucketAlreadyExists` | Both | Global name taken | Unique suffix / auto-name |
| Website not opening | Both | SG/public IP/boot delay/https | Check SG 80, IP, wait, http |

---

# 7️⃣ IaC SECURITY CHECKLIST 🔐 (BRIDGE TO UNIT V)

```
☐ No secrets in code, tfvars in Git, templates, or logs (use Secrets Manager/SSM, dynamic references)
☐ State bucket private, encrypted, versioned, restricted; treat state as sensitive
☐ Pipeline uses short-lived OIDC/role credentials with least privilege (plan role ≠ apply role)
☐ Scan IaC (Checkov/Trivy/cfn-nag) and enforce policy (OPA/Sentinel/Guard/Hooks) before apply
☐ Pin provider/module versions; commit lock files; vet third-party modules
☐ prevent_destroy / DeletionPolicy / termination protection for critical resources
☐ Mandatory tags + cost estimate in PRs; review every plan/change set
☐ Never "ClickOps" in prod; detect and fix drift
☐ If AI generates code: lint, scan, review; give agents least-privilege/sandbox access
```

---

# 8️⃣ LAB IDEAS BEYOND THE SYLLABUS

| Lab | Idea |
|---|---|
| A | Terraform module `web_server` called for dev/test; compare plans |
| B | Rebuild Unit I's VPC + EC2 + SG fully as code (CloudFormation `vpc.yaml` + Terraform) |
| C | S3 remote state + `use_lockfile`; try a lock conflict with two terminals |
| D | GitHub Actions pipeline: fmt, validate, checkov, plan on PR; apply on merge |
| E | `cfn-lint` + `cfn-guard` rule: deny SG with 0.0.0.0/0 on 22 |
| F | Import a manually created S3 bucket into Terraform (import block) |
| G | Same app with OpenTofu (`tofu init/plan/apply`) and compare |
| H | AWS CDK Python stack that synthesizes CloudFormation (`cdk synth` and read output) |
| I | Infracost on a PR: add instances and see monthly cost change |
| J | Terraform + Docker: EC2 user_data runs your Unit I Docker image |

---

# 9️⃣ THREE REAL-WORLD SCENARIOS

## Scenario A: "Prod and test environments don't match"
```
Problem: hand-built environments drift; bugs appear only in prod
Fix: define VPC/ALB/ASG/DB in modules; deploy identical stacks via per-environment variables; remote state; PR review; drift checks
Concepts: IaC consistency, modules, state, drift (Parts 1, 4)
```

## Scenario B: "AWS-only startup wants safe deployments with minimal tooling"
```
Problem: small team, no time to run state backends
Fix: CloudFormation (or CDK) with change sets in CI, termination protection, DeletionPolicy on data stores, automatic rollback, StackSets for multi-account baseline
Concepts: CloudFormation strengths (Part 2, 5)
```

## Scenario C: "Multi-cloud company, compliance team demands guardrails"
```
Problem: AWS + Azure + SaaS tools; manual reviews slow
Fix: Terraform/OpenTofu with shared modules, remote locked state, PR plans, policy as code (OPA/Sentinel/Checkov), Infracost, OIDC roles, GitOps-style pipeline, platform team golden modules
Concepts: Terraform ecosystem, policy as code, platform engineering (Parts 4, 6)
```

---

# 🔟 30 BEST UNIT II VIVA QUESTIONS 🔥 (WITH ANSWERS)

**IaC fundamentals**

1. **What is IaC?** Managing infrastructure through machine-readable definition files instead of manual processes.
2. **Why is IaC important in DevOps?** It automates environment creation within CI/CD and provides consistency, speed, version control, auditability, DR and collaboration.
3. **Declarative vs imperative?** Declarative describes the end state (Terraform/CFN); imperative lists steps (scripts).
4. **What is idempotency?** Re-running gives the same result without duplicates; a second apply shows no changes.
5. **What is configuration drift?** Reality diverging from code due to manual or external changes.
6. **Mutable vs immutable infrastructure?** In-place modification vs replacement from new images; immutable reduces drift.
7. **Provisioning vs configuration management?** Creating infrastructure (Terraform) vs configuring software inside servers (Ansible).
8. **What is GitOps?** Git as source of truth with automated reconciliation.

**CloudFormation**

9. **Template vs stack?** Blueprint file vs deployed resource group.
10. **Which section is mandatory?** `Resources`.
11. **Ref vs GetAtt?** Default identifier/parameter value vs specific attribute.
12. **What is a change set?** Preview of stack changes including replacement.
13. **What happens on failed creation/update?** Automatic rollback (`ROLLBACK_COMPLETE` / `UPDATE_ROLLBACK_COMPLETE`).
14. **Update behaviors?** No interruption, some interruption, replacement.
15. **How do you detect drift?** `detect-stack-drift` and view resource drifts.
16. **Nested stacks vs StackSets?** Reusable child stacks inside a parent vs one template across accounts/regions.
17. **DeletionPolicy values?** Delete, Retain, Snapshot (RetainExceptOnCreate).

**Terraform**

18. **Terraform workflow?** init → validate → plan → apply → destroy.
19. **What is a provider?** Plugin translating configuration into platform API calls.
20. **What is state and why needed?** Mapping of code to real resources with attributes; enables plans, dependencies, collaboration.
21. **Resource vs data source?** Create/manage vs read existing info.
22. **Why remote state with locking?** Shared truth, concurrency safety, backups, security; S3 `use_lockfile` provides native locking.
23. **count vs for_each?** Index-based (can shift) vs key-based (stable).
24. **What is a module?** Reusable folder of Terraform config called via `module` block.
25. **What does `prevent_destroy` do?** Blocks plans that would destroy the resource.
26. **How to import an existing resource?** `import` block or `terraform import`, then align code.

**Comparison and trends**

27. **Terraform vs CloudFormation in one line?** Terraform: multi-cloud, user-managed state, no auto rollback; CloudFormation: AWS-only, managed state, automatic rollback.
28. **What is OpenTofu?** MPL-licensed Linux Foundation fork of Terraform created after the 2023 BSL change.
29. **What is AWS CDK / Pulumi?** Infrastructure in programming languages; CDK synthesizes CloudFormation, Pulumi is multi-cloud.
30. **Name three IaC trends.** Policy as code/shift-left security, GitOps and platform engineering, AI-assisted IaC with guard rails (also OpenTofu/licensing, Crossplane, FinOps).

---

# 1️⃣1️⃣ COMMON UNIT II TRAPS 🚨

```
1. Calling Terraform "imperative" (it is declarative)
2. Saying CloudFormation has no preview (change sets exist)
3. Saying Terraform auto-rolls back (it does not)
4. Forgetting Resources is the ONLY mandatory CFN section
5. Confusing Ref and GetAtt
6. Thinking validate-template or terraform validate proves it will deploy (they check syntax/consistency only)
7. Committing terraform.tfstate or secrets tfvars to Git
8. Editing state by hand instead of using state commands / moved / import
9. Using count for named resources (index shift) instead of for_each
10. Using variables inside a backend block
11. Hard-coding AMI IDs (region-specific, go stale) instead of SSM parameter/data source
12. Forgetting that S3 bucket names are global and non-empty buckets block deletion
13. Not distinguishing provisioning (Terraform) from configuration management (Ansible)
14. Saying CloudFormation is multi-cloud (AWS-centric)
15. Stating licensing/pricing/version facts as permanent (they change; say "as of" or "currently")
16. Treating drift detection in CFN as auto-fix (it only reports)
17. Skipping cleanup → surprise bills
```

---

# 1️⃣2️⃣ WHAT TO WRITE IN AN EXAM ✍️

| Question type | Structure |
|---|---|
| **Define IaC / significance (3-5 marks)** | Definition + problem solved (drift, errors) + 4-5 benefits + DevOps link |
| **Declarative vs imperative / mutable vs immutable** | Table 5 rows + one example each |
| **CloudFormation architecture/anatomy (5-8 marks)** | Template → stack → resources diagram + template sections list + 1 small YAML + change set/rollback note |
| **Terraform workflow (5-8 marks)** | Diagram (write → init → plan → apply → destroy) + command table + state explanation + small HCL example |
| **Comparison (8-10 marks)** | Intro + 10-row table + key differences (state, rollback, scope, language) + when to use + conclusion |
| **Future trends (5-10 marks)** | 8-10 trends: trend → reason → tool → impact; table of tools; challenges; conclusion |
| **Practical/procedure** | Aim, requirements, numbered steps with commands, expected output, cleanup, result |

Writing habits: underline keywords (state, plan, change set, rollback, drift, provider), include one tiny code snippet, one diagram, and an "AWS-only vs multi-cloud" statement.

---

# 1️⃣3️⃣ WHAT NOT TO WRITE ❌

```
✘ "Terraform is better than CloudFormation" without conditions (say when/why)
✘ Exact prices, version numbers, or "latest" claims presented as permanent
✘ YAML with tabs or HCL with wrong block syntax (single-line blocks hold one argument only)
✘ Hard-coded secrets or real keys in sample code
✘ Mixing tool terms (calling a CFN stack "state file", calling plan a "change set" in Terraform answers)
✘ Skipping the diagram in workflow/architecture answers
✘ Claiming AI-generated IaC needs no review
```

---

# 1️⃣4️⃣ UNIT II MASTER SHEET 🧠🔥

```
IaC       code-defined infra: versioned, reviewable, repeatable, idempotent, declarative
BENEFITS  speed, consistency, audit, collaboration, DR, compliance, CI/CD/GitOps, cost control
CONCEPTS  desired vs actual → drift ; mutable/immutable ; push/pull ; provisioning vs config mgmt
CFN       template→stack ; Resources required ; Ref/GetAtt/Sub ; change set ; auto rollback ; drift report ; nested/StackSets
TERRAFORM HCL ; providers ; init→plan→apply→destroy ; variables/outputs/data/locals ; state
STATE     TF: file you protect (S3+encrypt+version+use_lockfile) ; CFN: AWS-managed
LOOPS     count (index) vs for_each (key) ; dynamic ; lifecycle ; modules ; import/moved
COMPARE   scope (AWS vs multi-cloud) • state • rollback • preview • language • ecosystem • license
TOOLS     OpenTofu, Pulumi, CDK, Bicep, Ansible, Crossplane, Argo CD/Flux, Checkov, OPA/Sentinel, Infracost, Packer
TRENDS    forks/licensing • languages • K8s-native • GitOps • policy as code • drift • platform eng. • AI • FinOps • immutable • supply chain • secrets/state
LABS      P3 CFN: validate→create→change set→update→failure→rollback→drift→delete
          P5 TF : init→validate→plan→apply→verify→modify→state→destroy
RULE      Never ClickOps prod • protect state • review every plan • scan + policy • always clean up
```

---

# 1️⃣5️⃣ MEMORY MAPS 🗺️

## IaC map
```
IaC ─┬─ Why (ClickOps pain: drift, errors, slow)
     ├─ What (declarative, idempotent, versioned)
     ├─ DevOps (CI/CD, GitOps, DR, audit)
     ├─ Concepts (state, drift, mutable/immutable, push/pull)
     └─ Tools (CFN, Terraform, Pulumi, CDK, Bicep, Ansible)
```

## Terraform map
```
.tf files ─init▶ providers ─plan▶ diff(code, state, reality) ─apply▶ cloud APIs ─▶ STATE
                                                   ▲                            │
                                      drift ───────┴──────── remote backend + lock
```

## CloudFormation map
```
Template ─▶ Stack ─▶ Change set (preview) ─▶ Execute ─▶ UPDATE_COMPLETE
                         │ failure                      │ manual change
                         ▼                               ▼
                 UPDATE_ROLLBACK_COMPLETE          Drift detection (MODIFIED)
```

## Choice map
```
AWS only?  ─yes▶ CloudFormation / CDK
   │no
   └▶ Multi-cloud/SaaS? ─▶ Terraform / OpenTofu (or Pulumi)
Configure inside servers? ─▶ Ansible
Kubernetes-centric platform? ─▶ Crossplane + GitOps
```

---

# 1️⃣6️⃣ HOW UNIT II CONNECTS TO THE REST OF INT378 🔗

| Unit | Connection |
|---|---|
| **Unit I** Cloud infrastructure and Docker | Everything you clicked (EC2, VPC, SG, S3) and the Docker image you ran can now be defined as code (CFN/Terraform, `user_data` running a Docker image) |
| **Unit III** Agile, CI/CD, Jenkins | The Jenkins/GitHub pipeline will run `fmt/validate/plan/apply` or `deploy` on every merge; IaC enables CD of infrastructure |
| **Unit IV** Monitoring and observability | Provision Prometheus/Grafana servers and CloudWatch alarms/dashboards as code; drift detection and alerts |
| **Unit V** DevSecOps and cloud security | IaC scanning, policy as code, secrets management, least-privilege IAM defined in code, state protection |
| **Unit VI** Cloud-native and cost | Serverless and Lambda deployed via IaC (SAM/Terraform), tagging, budgets, Infracost, governance and compliance as code |

```
Unit I (what runs) → Unit II (define it as code) → Unit III (deliver it automatically)
   → Unit IV (watch it) → Unit V (secure it) → Unit VI (optimize and govern it)
```

---

# 🏆 UNIT II — FINAL TAKEAWAY

```
1. Manual infrastructure doesn't scale; IaC makes it code: declarative, versioned, repeatable, reviewable.
2. CloudFormation = AWS-native, AWS-managed state, change sets, automatic rollback, drift reports.
3. Terraform = multi-cloud, providers + HCL, plan-first, state you must protect (remote + locked + encrypted).
4. Compare by scope, state, rollback, language, ecosystem and licensing; choose by context, and they can coexist.
5. The future: open forks (OpenTofu), code-based IaC, GitOps, policy as code, platform engineering,
   FinOps in pipelines, and AI assistance under strict human governance.
6. You can do it yourself: CloudFormation lifecycle with rollback and drift; Terraform EC2 + S3 with state management and destroy.
```

---

## 🫡 UNIT II STATUS

✅ Part 1 IaC from scratch
✅ Part 2 AWS CloudFormation
✅ Part 3 Terraform from scratch
✅ Part 4 Terraform advanced (state, modules, loops, lifecycle)
✅ Part 5 Terraform vs CloudFormation + landscape
✅ Part 6 Future trends and emerging tools
✅ Part 7 Practical lab guide (P3 + P5)
✅ Grand Consolidation

## Next up in your syllabus: UNIT III, Agile and CI/CD Practices for Scalable Product Development (SAFe, LeSS, TDD, BDD, CI/CD, Jenkins, A/B testing, technical debt) 🚀

Send the go-ahead and I'll build Unit III the same way.
