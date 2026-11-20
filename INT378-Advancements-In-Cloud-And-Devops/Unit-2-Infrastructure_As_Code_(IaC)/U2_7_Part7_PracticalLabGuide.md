# 🧪 UNIT II — PART 7: PRACTICAL LAB GUIDE (CLOUDFORMATION + TERRAFORM)

> **Covered in this document:** lab rules and prerequisites → collecting your VPC/subnet/key/IP values → **Practical 3** (AWS CloudFormation): validate, create stack, verify, update stack (parameter change via change set, add a resource), validate changes, **force a failure and observe rollback**, drift detection, delete stack, console route as well as CLI route → **comparing CloudFormation and Terraform implementations** (hands-on method + report table) → **Practical 5** (Terraform): install, credentials, project files for EC2 + security group + S3, `init`, `fmt`, `validate`, `plan`, `apply`, verify website, inspect state, modify infrastructure, `destroy`, managing state files (list/show/backup/remote S3 backend/lock) → troubleshooting flowcharts → cleanup checklist → lab report templates → practical viva → quick checklists.

---

Absolutely mate. 😤🔥 This is the lab manual for **Practical 3 and Practical 5**. Follow in order, capture screenshots as you go, and **clean up at the end**.

Your syllabus lines:

> **P3:** *Design and deploy AWS CloudFormation templates to provision cloud resources. Update stack configurations after modifying templates, validate stack changes, perform rollback if required, and compare Infrastructure as Code implementation using AWS CloudFormation and Terraform.*
>
> **P5:** *Install Terraform and develop Infrastructure as Code (IaC) configuration files to provision cloud resources such as EC2 instances and S3 storage. Initialize, validate, apply, modify, and destroy the infrastructure using Terraform commands while managing Terraform state files.*

---

# 0️⃣ LAB RULES AND PREREQUISITES 🚨

```
1. AWS account with a budget alert (see Unit I Part 8). Use a non-root IAM user/role with EC2, S3, CloudFormation permissions.
2. AWS CLI v2 installed and configured:  aws configure   →   aws sts get-caller-identity
3. A region (examples use ap-south-1). Use the SAME region everywhere.
4. A key pair (only needed if you want SSH). Optional for the web-page check.
5. Default VPC present (new accounts have one). If you deleted it, create a VPC first (Unit II Part 2 vpc.yaml).
6. Always run cleanup (delete-stack / terraform destroy) after each practical.
7. Free Tier terms changed for new AWS accounts in 2025: check your Billing console for current credits/limits.
```

## 0.1 Collect your values once (CLI)

```bash
export AWS_REGION=ap-south-1
export AWS_DEFAULT_REGION=$AWS_REGION

# Default VPC
VPC_ID=$(aws ec2 describe-vpcs --filters Name=isDefault,Values=true --query "Vpcs[0].VpcId" --output text)

# A subnet in that VPC (default subnets auto-assign public IPs)
SUBNET_ID=$(aws ec2 describe-subnets --filters Name=vpc-id,Values=$VPC_ID Name=default-for-az,Values=true \
            --query "Subnets[0].SubnetId" --output text)

# Your public IP in CIDR form
MY_IP="$(curl -s https://checkip.amazonaws.com)/32"

# Key pair (create if you don't have one)
aws ec2 create-key-pair --key-name lab-key --query KeyMaterial --output text > lab-key.pem && chmod 400 lab-key.pem

echo "VPC=$VPC_ID  SUBNET=$SUBNET_ID  MYIP=$MY_IP"
```

(On Windows PowerShell use `$VPC_ID = aws ec2 describe-vpcs ...` and `(Invoke-RestMethod https://checkip.amazonaws.com).Trim()`; or copy the IDs from the console.)

---

# 🅰️ PRACTICAL 3 — AWS CLOUDFORMATION

## A.1 Objective and mental model

```
template.yaml ──▶ [CloudFormation service] ──▶ STACK ──▶ SG + EC2 (+ S3)
      │                    ▲
      │ edit               │ change set (preview) → execute → UPDATE_COMPLETE
      └─ update ───────────┘                       └─ on failure → UPDATE_ROLLBACK_COMPLETE
```

## A.2 Step 1: Create the project folder and template

```bash
mkdir ~/cfn-lab && cd ~/cfn-lab
```

Create `webserver.yaml` (same as Unit II Part 2 §16, repeated here so the lab is self-contained):

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: Practical 3 - EC2 web server with security group (Nginx), deployed by CloudFormation

Parameters:
  VpcId:
    Type: AWS::EC2::VPC::Id
  SubnetId:
    Type: AWS::EC2::Subnet::Id
  KeyName:
    Type: AWS::EC2::KeyPair::KeyName
  MyIpCidr:
    Type: String
    AllowedPattern: '^(\d{1,3}\.){3}\d{1,3}/\d{1,2}$'
    ConstraintDescription: must look like x.x.x.x/32
  InstanceType:
    Type: String
    Default: t3.micro
    AllowedValues: [t3.micro, t3.small, t2.micro]
  LatestAmiId:
    Type: AWS::SSM::Parameter::Value<AWS::EC2::Image::Id>
    Default: /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64

Resources:
  WebSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: HTTP from anywhere, SSH from my IP
      VpcId: !Ref VpcId
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          CidrIp: 0.0.0.0/0
        - IpProtocol: tcp
          FromPort: 22
          ToPort: 22
          CidrIp: !Ref MyIpCidr
      Tags:
        - Key: Name
          Value: !Sub "${AWS::StackName}-web-sg"

  WebServer:
    Type: AWS::EC2::Instance
    Properties:
      ImageId: !Ref LatestAmiId
      InstanceType: !Ref InstanceType
      KeyName: !Ref KeyName
      SubnetId: !Ref SubnetId
      SecurityGroupIds:
        - !Ref WebSecurityGroup
      UserData:
        Fn::Base64: !Sub |
          #!/bin/bash
          dnf install -y nginx
          systemctl enable --now nginx
          echo "<h1>Deployed by CloudFormation stack ${AWS::StackName} (${AWS::Region})</h1>" > /usr/share/nginx/html/index.html
      Tags:
        - Key: Name
          Value: !Sub "${AWS::StackName}-web"

Outputs:
  InstanceId:
    Value: !Ref WebServer
  PublicIP:
    Value: !GetAtt WebServer.PublicIp
  WebsiteURL:
    Value: !Sub "http://${WebServer.PublicIp}"
```

## A.3 Step 2: Validate the template ✅

```bash
aws cloudformation validate-template --template-body file://webserver.yaml
# Returns the parameter list/description if syntax is OK; error text if not

pip install cfn-lint                # optional stronger linter
cfn-lint webserver.yaml             # no output = no issues found
```

Remember: `validate-template` checks **syntax/structure**, not whether values are valid for AWS (e.g., wrong property values are discovered at deploy time).

## A.4 Step 3: Create (deploy) the stack

### CLI route

```bash
aws cloudformation create-stack --stack-name lab-web \
  --template-body file://webserver.yaml \
  --parameters ParameterKey=VpcId,ParameterValue=$VPC_ID \
               ParameterKey=SubnetId,ParameterValue=$SUBNET_ID \
               ParameterKey=KeyName,ParameterValue=lab-key \
               ParameterKey=MyIpCidr,ParameterValue=$MY_IP \
               ParameterKey=InstanceType,ParameterValue=t3.micro

aws cloudformation wait stack-create-complete --stack-name lab-web
aws cloudformation describe-stacks --stack-name lab-web --query "Stacks[0].[StackStatus,Outputs]" --output table
```

### Console route (for screenshots)

```
CloudFormation → Create stack → With new resources → Upload a template file → webserver.yaml
→ Stack name: lab-web → select VPC, subnet, key pair; MyIpCidr; keep t3.micro → Next → Next → Submit
→ Events tab: watch CREATE_IN_PROGRESS → CREATE_COMPLETE → Outputs tab: copy WebsiteURL
```

### Watch the events (shows dependency order)

```bash
aws cloudformation describe-stack-events --stack-name lab-web \
  --query "StackEvents[].[Timestamp,LogicalResourceId,ResourceStatus,ResourceStatusReason]" --output table
```

Expected order: `WebSecurityGroup` created first, then `WebServer`, then the stack.

## A.5 Step 4: Verify

```bash
URL=$(aws cloudformation describe-stacks --stack-name lab-web \
      --query "Stacks[0].Outputs[?OutputKey=='WebsiteURL'].OutputValue" --output text)
echo $URL
curl $URL                # <h1>Deployed by CloudFormation stack lab-web (ap-south-1)</h1>
```

(Wait ~1-2 minutes after `CREATE_COMPLETE` if the page isn't ready: UserData is still running. Use `http://`, not `https://`.)

Optional SSH check: `ssh -i lab-key.pem ec2-user@<PublicIP>`.

📸 Screenshots: stack events, Outputs tab, browser page, resources tab.

## A.6 Step 5: UPDATE #1, change a parameter using a **change set** (validate the change before applying) ⭐

Goal: change instance type `t3.micro → t3.small`.

```bash
aws cloudformation create-change-set --stack-name lab-web --change-set-name resize \
  --use-previous-template \
  --parameters ParameterKey=VpcId,UsePreviousValue=true \
               ParameterKey=SubnetId,UsePreviousValue=true \
               ParameterKey=KeyName,UsePreviousValue=true \
               ParameterKey=MyIpCidr,UsePreviousValue=true \
               ParameterKey=InstanceType,ParameterValue=t3.small \
               ParameterKey=LatestAmiId,UsePreviousValue=true
```

(`--use-previous-template` reuses the stored template, so only the parameter changes. It cannot be combined with `--template-body`.)

```bash
aws cloudformation wait change-set-create-complete --stack-name lab-web --change-set-name resize
aws cloudformation describe-change-set --stack-name lab-web --change-set-name resize \
  --query "Changes[].ResourceChange.[Action,LogicalResourceId,Replacement,Details[0].Target.Attribute]" --output table
```

Expected: `Modify | WebServer | False/Conditional | InstanceType`. **Replacement = False** means in-place update with **some interruption** (instance is stopped and started). Public IP may change if no Elastic IP.

Execute it:

```bash
aws cloudformation execute-change-set --stack-name lab-web --change-set-name resize
aws cloudformation wait stack-update-complete --stack-name lab-web
```

Verify: Console → EC2 → instance type now t3.small; stack status `UPDATE_COMPLETE`; **re-read the new PublicIP from Outputs** (it may differ).

Console route: Stack → **Update** → Replace/Use existing template → change `InstanceType` → Next ... → on the Review page you can see **Change set preview** → Submit.

## A.7 Step 6: UPDATE #2, add a new resource (S3 bucket) by modifying the template

Edit `webserver.yaml`: add under `Resources:`

```yaml
  AppBucket:
    Type: AWS::S3::Bucket
    Properties:
      VersioningConfiguration:
        Status: Enabled
      PublicAccessBlockConfiguration:
        BlockPublicAcls: true
        BlockPublicPolicy: true
        IgnorePublicAcls: true
        RestrictPublicBuckets: true
```

and under `Outputs:`

```yaml
  BucketName:
    Value: !Ref AppBucket
```

Preview and apply:

```bash
aws cloudformation validate-template --template-body file://webserver.yaml
aws cloudformation create-change-set --stack-name lab-web --change-set-name add-bucket \
  --template-body file://webserver.yaml \
  --parameters ParameterKey=VpcId,UsePreviousValue=true ParameterKey=SubnetId,UsePreviousValue=true \
               ParameterKey=KeyName,UsePreviousValue=true ParameterKey=MyIpCidr,UsePreviousValue=true \
               ParameterKey=InstanceType,UsePreviousValue=true ParameterKey=LatestAmiId,UsePreviousValue=true
aws cloudformation wait change-set-create-complete --stack-name lab-web --change-set-name add-bucket
aws cloudformation describe-change-set --stack-name lab-web --change-set-name add-bucket \
   --query "Changes[].ResourceChange.[Action,LogicalResourceId,ResourceType,Replacement]" --output table
aws cloudformation execute-change-set --stack-name lab-web --change-set-name add-bucket
aws cloudformation wait stack-update-complete --stack-name lab-web
```

Expected change set: `Add | AppBucket | AWS::S3::Bucket`. After update the `BucketName` output appears. Verify: `aws s3 ls`.

## A.8 Step 7: Show **replacement** (optional, instructive) 🔄

Change a property marked "Replacement": e.g., add `BucketName` to the bucket (a new name) in the template. The change set will show `Replacement: True` for `AppBucket`. Explain in your report: CloudFormation creates a new bucket then deletes the old one, so data would not carry over. (Do not apply in real production without a plan.)

## A.9 Step 8: ROLLBACK demonstration ⭐⭐ (syllabus: "perform rollback if required")

We deliberately cause a **runtime failure**, so CloudFormation reverts automatically.

**Failure design:** add a second bucket with a name that already exists globally (S3 names are global):

```yaml
  BadBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: test        # almost certainly taken by someone else → AlreadyExists error
```

Run an update with this template:

```bash
aws cloudformation update-stack --stack-name lab-web --template-body file://webserver.yaml \
  --parameters ParameterKey=VpcId,UsePreviousValue=true ParameterKey=SubnetId,UsePreviousValue=true \
               ParameterKey=KeyName,UsePreviousValue=true ParameterKey=MyIpCidr,UsePreviousValue=true \
               ParameterKey=InstanceType,UsePreviousValue=true ParameterKey=LatestAmiId,UsePreviousValue=true
aws cloudformation describe-stack-events --stack-name lab-web \
  --query "StackEvents[0:8].[LogicalResourceId,ResourceStatus,ResourceStatusReason]" --output table
```

Observe the event sequence:

```
UPDATE_IN_PROGRESS            lab-web
CREATE_FAILED                 BadBucket   "test already exists"  (reason text)
UPDATE_ROLLBACK_IN_PROGRESS   lab-web
UPDATE_ROLLBACK_COMPLETE      lab-web   ← stack is back to the previous working state (EC2 + SG + AppBucket)
```

Verify: the website still works; `BadBucket` does not exist; stack status `UPDATE_ROLLBACK_COMPLETE`.

Then **fix and move forward**: remove `BadBucket` from the template (or give it a unique name) and update again → `UPDATE_COMPLETE`.

📸 Screenshots: events showing CREATE_FAILED and UPDATE_ROLLBACK_COMPLETE.

### Other safe ways to trigger/see rollback

| Method | What you see |
|---|---|
| Invalid property value accepted by syntax but rejected by service (e.g., unsupported `InstanceType` in your region/AZ for a parameter without AllowedValues) | Runtime failure → rollback |
| Wrong AMI/subnet combination | `CREATE_FAILED` for instance → rollback |
| **Manual rollback**: during a long update, console → **Stack actions → Cancel update stack** | `UPDATE_ROLLBACK_IN_PROGRESS` |
| Rollback triggers with CloudWatch alarm | Alarm fires during monitoring window → auto rollback |

⚠️ A parameter value outside `AllowedValues` (e.g. `t3.superlarge`) is rejected **before** the update starts (validation error), so that is *not* a rollback demo; it shows parameter validation instead. Mention both in the report.

## A.10 Step 9: Drift detection 🔍

1. Manually change something in the console: EC2 → Security Groups → `lab-web-...` → add inbound rule: custom TCP 8080 from anywhere.
2. Detect drift:

```bash
DRIFT_ID=$(aws cloudformation detect-stack-drift --stack-name lab-web --query StackDriftDetectionId --output text)
aws cloudformation describe-stack-drift-detection-status --stack-drift-detection-id $DRIFT_ID
aws cloudformation describe-stack-resource-drifts --stack-name lab-web \
   --stack-resource-drift-status-filters MODIFIED DELETED --output json
```

Expected: `WebSecurityGroup` → `MODIFIED` with a property difference for `SecurityGroupIngress` (expected vs actual).

3. Fix: remove the manual rule (or update the template to include it), then re-run drift detection → `IN_SYNC`.

Console: Stack → **Stack actions → Detect drift** → **View drift results**.

## A.11 Step 10: Protection features (quick demo)

```bash
aws cloudformation update-termination-protection --enable-termination-protection --stack-name lab-web
aws cloudformation delete-stack --stack-name lab-web        # ← fails/blocked: termination protection
aws cloudformation update-termination-protection --no-enable-termination-protection --stack-name lab-web
```

## A.12 Step 11: Delete the stack (cleanup) 🧹

```bash
aws s3 rm s3://$(aws cloudformation describe-stacks --stack-name lab-web \
   --query "Stacks[0].Outputs[?OutputKey=='BucketName'].OutputValue" --output text) --recursive   # if objects exist
aws cloudformation delete-stack --stack-name lab-web
aws cloudformation wait stack-delete-complete --stack-name lab-web
aws ec2 delete-key-pair --key-name lab-key          # only if you won't use it for Practical 5
```

Observe delete order: `WebServer` first, then `WebSecurityGroup` (reverse dependency). A non-empty bucket gives `DELETE_FAILED` unless emptied.

## A.13 Troubleshooting (Practical 3)

| Symptom | Cause | Fix |
|---|---|---|
| `ValidationError: Template format error` | YAML indentation/tabs | Use spaces; `cfn-lint` |
| `Parameters: [KeyName] must have values` | Missing parameter | Provide all |
| `ROLLBACK_COMPLETE` on create; reason: `Unsupported ...` | Instance type not in AZ/region | Choose another type/subnet |
| Page not opening | SG rule/IP/UserData delay/https | Wait 2 min; check SG, public IP, use http |
| `Stack is in ROLLBACK_COMPLETE and cannot be updated` | Create failed earlier | Delete stack and recreate |
| `No updates are to be performed` | Template/params identical | Make an actual change |
| `AccessDenied` | IAM user lacks permissions | Add CloudFormation/EC2/S3 permissions |
| `InsufficientCapabilitiesException` | Template creates IAM resources | Add `--capabilities CAPABILITY_IAM` |

---

# 🆚 COMPARING CLOUDFORMATION AND TERRAFORM (PRACTICAL 3, LAST PART)

Do **Practical 5 below** (same infrastructure with Terraform), then fill this table with your real observations:

| Item | CloudFormation (measured) | Terraform (measured) |
|---|---|---|
| Lines of code (`wc -l`) | ___ | ___ |
| Tools to install | AWS CLI | Terraform CLI + provider download (`init`) |
| Create command | `create-stack`/`deploy` | `terraform apply` |
| Time to create | ___ s | ___ s |
| Preview before change | Change set output | `terraform plan` output |
| Update experience | change set → execute | edit → plan → apply |
| Failure behavior | `UPDATE_ROLLBACK_COMPLETE` (automatic) | Partial apply: fix and re-apply, or git revert |
| Drift handling | `detect-stack-drift` report | `plan` shows drift; `apply` reverts it |
| State location | AWS service (no file) | `terraform.tfstate` (local or S3) |
| Delete command | `delete-stack` | `terraform destroy` |
| Readability (your opinion) | ___ | ___ |

Sample conclusion (adapt with your numbers):

```
Both tools produced the same EC2 + security group + S3 infrastructure declaratively and idempotently.
CloudFormation required no extra tooling and automatically rolled back the failed update, while Terraform
gave a more compact HCL syntax, a mandatory plan preview and portable multi-cloud workflow but required
state-file management and manual recovery after failures. For an AWS-only team that wants managed state and
native rollback, CloudFormation is convenient; for multi-cloud or highly modular environments Terraform is preferable.
```

Count lines: `wc -l webserver.yaml` and `cat *.tf | wc -l`.

---

# 🅱️ PRACTICAL 5 — TERRAFORM (EC2 + S3, FULL LIFECYCLE + STATE)

## B.1 Objective and flow

```
write .tf → init → fmt → validate → plan → apply → verify → modify → plan/apply → state mgmt → destroy
```

## B.2 Step 1: Install and verify Terraform

Windows: `winget install HashiCorp.Terraform` · macOS: `brew tap hashicorp/tap && brew install hashicorp/tap/terraform` · Ubuntu: HashiCorp apt repo (Part 3 §2.3).

```bash
terraform version
aws sts get-caller-identity        # confirms credentials Terraform will use
```

## B.3 Step 2: Create the project

```bash
mkdir ~/tf-lab && cd ~/tf-lab
```

### `versions.tf`

```hcl
terraform {
  required_version = ">= 1.5.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"
    }
    random = {
      source  = "hashicorp/random"
      version = "~> 3.6"
    }
  }
}
```

(If `init` reports no matching version, check registry.terraform.io for the current AWS provider major version and adjust.)

### `variables.tf`

```hcl
variable "region" {
  description = "AWS region"
  type        = string
  default     = "ap-south-1"
}

variable "project" {
  description = "Name prefix for resources"
  type        = string
  default     = "tf-lab"
}

variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "t3.micro"

  validation {
    condition     = contains(["t3.micro", "t3.small", "t2.micro"], var.instance_type)
    error_message = "Use t3.micro, t3.small or t2.micro."
  }
}

variable "my_ip_cidr" {
  description = "Your public IP in CIDR form for SSH, e.g. 203.0.113.25/32"
  type        = string
}
```

### `terraform.tfvars`

```hcl
my_ip_cidr = "203.0.113.25/32"     # replace with: curl https://checkip.amazonaws.com  (+ /32)
```

### `main.tf`

```hcl
provider "aws" {
  region = var.region

  default_tags {
    tags = {
      Project   = var.project
      ManagedBy = "Terraform"
    }
  }
}

# ---------- lookups (data sources) ----------
data "aws_vpc" "default" {
  default = true
}

data "aws_subnets" "default" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.default.id]
  }
}

data "aws_ssm_parameter" "al2023" {
  name = "/aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64"
}

# ---------- security group ----------
resource "aws_security_group" "web" {
  name        = "${var.project}-web-sg"
  description = "HTTP from anywhere, SSH from my IP"
  vpc_id      = data.aws_vpc.default.id

  ingress {
    description = "HTTP"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    description = "SSH from my IP"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = [var.my_ip_cidr]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "${var.project}-web-sg"
  }
}

# ---------- EC2 instance ----------
resource "aws_instance" "web" {
  ami                         = data.aws_ssm_parameter.al2023.value
  instance_type               = var.instance_type
  subnet_id                   = data.aws_subnets.default.ids[0]
  vpc_security_group_ids      = [aws_security_group.web.id]
  associate_public_ip_address = true

  user_data = <<-EOF
    #!/bin/bash
    dnf install -y nginx
    systemctl enable --now nginx
    echo "<h1>Deployed by Terraform: ${var.project}</h1>" > /usr/share/nginx/html/index.html
  EOF

  tags = {
    Name = "${var.project}-web"
  }
}

# ---------- S3 bucket ----------
resource "random_id" "suffix" {
  byte_length = 4
}

resource "aws_s3_bucket" "app" {
  bucket        = "${var.project}-${random_id.suffix.hex}"
  force_destroy = true          # lab convenience: lets destroy delete a non-empty bucket
}

resource "aws_s3_bucket_versioning" "app" {
  bucket = aws_s3_bucket.app.id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_public_access_block" "app" {
  bucket                  = aws_s3_bucket.app.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```

### `outputs.tf`

```hcl
output "instance_id" {
  value = aws_instance.web.id
}

output "public_ip" {
  value = aws_instance.web.public_ip
}

output "website_url" {
  value = "http://${aws_instance.web.public_ip}"
}

output "bucket_name" {
  value = aws_s3_bucket.app.bucket
}
```

### `.gitignore`

```
.terraform/
*.tfstate
*.tfstate.*
tfplan
crash.log
```

## B.4 Step 3: `terraform init` ⭐

```bash
terraform init
```

Expected highlights:

```
Initializing the backend...
Initializing provider plugins...
- Installing hashicorp/aws v6.x.x...
- Installing hashicorp/random v3.x.x...
Terraform has created a lock file .terraform.lock.hcl ...
Terraform has been successfully initialized!
```

Check what appeared: `ls -a` → `.terraform/`, `.terraform.lock.hcl`.

## B.5 Step 4: `fmt` and `validate`

```bash
terraform fmt
terraform validate        # Success! The configuration is valid.
```

Break something on purpose to see an error (e.g., misspell `aws_security_group.web.id` as `aws_security_group.webb.id`) → `validate` reports "Reference to undeclared resource". Fix it again.

## B.6 Step 5: `terraform plan` ⭐

```bash
terraform plan -out=tfplan
```

Read it: `+` for each resource, `Plan: 6 to add, 0 to change, 0 to destroy.` (security group, instance, random_id, bucket, versioning, public access block). Note `(known after apply)` values.

📸 Screenshot the plan summary.

## B.7 Step 6: `terraform apply`

```bash
terraform apply tfplan            # saved plan → applies without extra prompt
# or:  terraform apply   → type yes
```

Expected end:

```
Apply complete! Resources: 6 added, 0 changed, 0 destroyed.

Outputs:
bucket_name = "tf-lab-a1b2c3d4"
instance_id = "i-0abc..."
public_ip   = "13.x.x.x"
website_url = "http://13.x.x.x"
```

## B.8 Step 7: Verify in AWS and in the browser ✅

```bash
terraform output
curl $(terraform output -raw website_url)            # <h1>Deployed by Terraform: tf-lab</h1>  (wait 1-2 min if not ready)
aws s3 ls | grep tf-lab
aws ec2 describe-instances --instance-ids $(terraform output -raw instance_id) \
    --query "Reservations[0].Instances[0].[State.Name,InstanceType,PublicIpAddress]" --output text
```

Re-run `terraform apply` immediately → **"No changes. Your infrastructure matches the configuration."** (idempotency proof 📸).

## B.9 Step 8: Inspect the state ⭐ (syllabus: "managing Terraform state files")

```bash
ls -l terraform.tfstate*                 # terraform.tfstate (and .backup after later changes)
terraform state list
# aws_instance.web
# aws_s3_bucket.app
# aws_s3_bucket_public_access_block.app
# aws_s3_bucket_versioning.app
# aws_security_group.web
# random_id.suffix
# data.aws_ssm_parameter.al2023  (data sources also listed)

terraform state show aws_instance.web    # all recorded attributes
terraform show | head -50
head -30 terraform.tfstate               # raw JSON: version, serial, lineage, resources
```

📸 Screenshot `state list` and `state show`. Explain in your report: state maps `aws_instance.web` → `i-0abc...`.

## B.10 Step 9: Modify the infrastructure (and watch the plan) ⭐

### Modification A: change instance type (in-place update, some interruption)

```bash
terraform apply -var="instance_type=t3.small"
```

Plan shows `~ update in-place` for `aws_instance.web` (`instance_type: "t3.micro" -> "t3.small"`). Type `yes`.

(Persist the change by editing `terraform.tfvars`: `instance_type = "t3.small"`; otherwise the next plain apply wants to revert to the default `t3.micro`. Good teaching moment!)

### Modification B: change tags (no interruption)

Add to `aws_instance.web.tags`: `Owner = "your-name"`. `terraform plan` → `~ update in-place` (tags only). Apply.

### Modification C: force a replacement (see `-/+`)

```bash
terraform plan -replace="aws_instance.web"       # shows -/+ destroy and then create replacement
terraform apply -replace="aws_instance.web"      # public IP will change
```

### Modification D: add a second bucket with `for_each` (scale the code)

```hcl
resource "aws_s3_bucket" "extra" {
  for_each = toset(["logs", "backups"])
  bucket   = "${var.project}-${each.key}-${random_id.suffix.hex}"
  force_destroy = true
}
```

`terraform plan` → `Plan: 2 to add`. Apply, then `terraform state list` shows `aws_s3_bucket.extra["logs"]`, `["backups"]`. (Remove later or leave for destroy to clean.)

## B.11 Step 10: Drift demo (optional but impressive)

1. Console: EC2 → Security groups → `tf-lab-web-sg` → add inbound 8080 from anywhere.
2. `terraform plan` → shows `~ update in-place` removing the unmanaged rule (Terraform wants to restore desired state).
3. `terraform apply` → drift corrected. (Or run `terraform plan -refresh-only` to only view drift.)

## B.12 Step 11: State management exercises ⭐

### (a) State backup behavior
After each apply that changes state, `terraform.tfstate.backup` holds the previous state. Show both files with timestamps.

### (b) Move from local to **remote S3 state with locking** (optional advanced, high marks)

```bash
BUCKET=tf-state-$(aws sts get-caller-identity --query Account --output text)-lab
aws s3api create-bucket --bucket $BUCKET --region ap-south-1 --create-bucket-configuration LocationConstraint=ap-south-1
aws s3api put-bucket-versioning --bucket $BUCKET --versioning-configuration Status=Enabled
aws s3api put-public-access-block --bucket $BUCKET --public-access-block-configuration BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true
```

Add `backend.tf` (use YOUR bucket name; backend blocks can't use variables):

```hcl
terraform {
  backend "s3" {
    bucket       = "tf-state-123456789012-lab"
    key          = "lab/terraform.tfstate"
    region       = "ap-south-1"
    encrypt      = true
    use_lockfile = true         # Terraform 1.10+; older versions: use dynamodb_table
  }
}
```

```bash
terraform init -migrate-state        # answer yes → copies local state to S3
aws s3 ls s3://$BUCKET/lab/          # terraform.tfstate now in S3
terraform state list                 # still works (now reading remote)
rm -f terraform.tfstate terraform.tfstate.backup   # local copies no longer authoritative (after verifying remote!)
```

Lock demo: start `terraform apply` in one terminal and wait at the "yes" prompt; in a second terminal run `terraform plan` → **"Error acquiring the state lock"** (with who/when). Answer `no` in the first terminal to release the lock.

### (c) Other state commands to demonstrate

```bash
terraform state list
terraform state mv aws_instance.web aws_instance.frontend      # then rename the resource in code to match, or use a moved block
terraform state rm 'aws_s3_bucket.extra["logs"]'                # stop managing (real bucket remains!)
terraform apply -refresh-only                                   # sync state with reality
```

## B.13 Step 12: `terraform destroy` 🧹

If you migrated to remote state, destroy first, then clean the state bucket last.

```bash
terraform plan -destroy              # preview deletion
terraform destroy                    # type yes
# Destroy complete! Resources: 6 destroyed.
terraform state list                 # empty
```

Verify: EC2 console (instance terminated), `aws s3 ls | grep tf-lab` (none), SG gone.

If you created the remote-state bucket for the lab, remove it manually after destroy:

```bash
aws s3 rm s3://$BUCKET --recursive
# versioned bucket: delete all versions/markers too (console "Empty" is easiest), then
aws s3api delete-bucket --bucket $BUCKET
```

## B.14 Troubleshooting (Practical 5)

| Symptom | Cause | Fix |
|---|---|---|
| `terraform: command not found` | PATH | Reopen terminal / reinstall |
| `No valid credential sources found` | AWS creds missing | `aws configure` / env vars |
| `Error: Invalid provider version` / `no available releases match` | Version constraint | Check registry, adjust `~>` |
| `var.my_ip_cidr: required` prompt | No value set | Add to `terraform.tfvars` |
| `InvalidParameterValue ... AMI` | Wrong AMI path | Use the SSM data source shown |
| Website not opening | Instance still booting; SG/IP; https | Wait; `terraform output`; use http |
| `UnauthorizedOperation` | IAM permissions | Add EC2/S3/VPC permissions |
| `BucketAlreadyExists` | Name taken | `random_id` suffix (already used) |
| `Unsupported instance type in AZ` | t3 not in that default subnet AZ | Pick another subnet index or instance type |
| `Error acquiring the state lock` | Another run/stale lock | Wait; `terraform force-unlock <ID>` if sure |
| Destroy fails on bucket not empty | Objects present | `force_destroy = true` + apply, then destroy |
| Plan shows unexpected `-/+` | Changed immutable attribute | Read plan; revert or accept replacement |

## B.15 Evidence checklist for the report 📸

```
☐ terraform version output                    ☐ init success + lock file
☐ validate success                             ☐ plan summary (6 to add)
☐ apply complete + outputs                     ☐ browser page with public IP
☐ idempotency: "No changes"                    ☐ state list + state show
☐ modification plan (~ update in-place)        ☐ -replace plan (-/+) (optional)
☐ drift fixed by apply (optional)              ☐ remote state in S3 + lock error (optional)
☐ destroy complete                             ☐ AWS console empty after destroy
```

---

# 🧹 FINAL CLEANUP CHECKLIST (BOTH PRACTICALS)

```
☐ CloudFormation stack lab-web deleted (DELETE_COMPLETE)
☐ terraform destroy completed; state empty
☐ No running EC2 instances; no extra EBS volumes/Elastic IPs
☐ S3 buckets removed (app, extra, state bucket if created)
☐ Key pair deleted (console + local .pem secured/deleted)
☐ Budgets/alerts still configured; Billing dashboard shows no surprise charges
```

---

# 📝 LAB REPORT TEMPLATES

```
PRACTICAL 3 REPORT
Aim            : (copy syllabus line)
Requirements   : AWS account, AWS CLI, editor, key pair
Theory         : IaC, CloudFormation (template, stack, change set, rollback, drift), declarative idea
Procedure      : validate → create stack → verify → update via change set → add resource → rollback demo → drift → delete
Output         : screenshots (events, outputs, change set, rollback, drift)
Comparison     : table CloudFormation vs Terraform with measured values + conclusion
Result/Precautions/Viva

PRACTICAL 5 REPORT
Aim            : (copy syllabus line)
Requirements   : Terraform CLI, AWS CLI, credentials, editor
Theory         : Terraform architecture, providers, state, workflow
Procedure      : install → write files → init → fmt/validate → plan → apply → verify → modify → state mgmt → destroy
Output         : screenshots from checklist B.15
Result/Precautions/Viva
```

---

# 🎯 PRACTICAL VIVA QUESTIONS

**Q1. What does `validate-template` check?** Template syntax and structure, not whether AWS will accept all property values.

**Q2. What is a change set and why use it?** A preview of stack changes (add/modify/remove, replacement) to avoid surprises before updating.

**Q3. What is `Replacement: True`?** The resource will be recreated (new resource, old deleted), so data may be lost.

**Q4. How did you demonstrate rollback?** Updated the stack with a resource that fails (duplicate S3 name); CloudFormation reverted to `UPDATE_ROLLBACK_COMPLETE`.

**Q5. How do you detect drift in CloudFormation?** `detect-stack-drift` and view resource drifts.

**Q6. Why can't you update a stack in `ROLLBACK_COMPLETE`?** Creation failed; the stack must be deleted and recreated.

**Q7. What does `terraform init` do?** Downloads providers/modules, configures the backend, creates the lock file.

**Q8. What does `terraform plan` show and does it change anything?** The diff between code and real state (+ ~ - -/+); it changes nothing.

**Q9. Why did the second `apply` say "No changes"?** Declarative idempotency: real infrastructure matches configuration.

**Q10. What is in `terraform.tfstate`?** Mapping of resources to real IDs and all their attributes (so protect it).

**Q11. How did you modify infrastructure in Terraform?** Changed variable/code, ran plan, saw `~ update in-place`, applied.

**Q12. What happens if apply fails halfway in Terraform?** Created resources remain in state; fix the error and re-apply or revert code.

**Q13. How do you share state with a team?** Remote backend (S3 with versioning, encryption, `use_lockfile`).

**Q14. What does `terraform destroy` do and in what order?** Deletes managed resources in reverse dependency order.

**Q15. Compare the two tools based on your lab.** Use your measured table: lines of code, preview, rollback, state, tooling.

---

# ✅ QUICK CHECKLISTS

**Practical 3:** collect VPC/subnet/key/IP → validate → create stack → verify URL → change set (t3.micro→t3.small) → add S3 bucket → forced failure → `UPDATE_ROLLBACK_COMPLETE` → drift detection → delete stack → fill comparison table.

**Practical 5:** install → files (versions, variables, main, outputs, tfvars) → `init` → `fmt`/`validate` → `plan -out` → `apply` → verify (curl + S3) → `No changes` → state list/show → modify → (remote state + lock) → `destroy` → verify empty.

---

## 🫡 PART 7 STATUS

Both Unit II practicals are now fully scripted with expected outputs, rollback and drift demos, state management, comparison table and cleanup.

**STOP here.** Last file:

> **🏆 GRAND CONSOLIDATION: Unit II full revision, comparison tables, 30 viva Qs, traps, what to write in exams, master sheet**
