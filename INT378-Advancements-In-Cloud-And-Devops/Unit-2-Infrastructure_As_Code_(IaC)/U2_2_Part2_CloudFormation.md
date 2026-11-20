# 🟠 UNIT II — PART 2: AWS CLOUDFORMATION (THEORY + TEMPLATES)

> **Covered in this document:** what CloudFormation is → how it works (template → stack → resources) → template formats (YAML vs JSON) → full template anatomy (AWSTemplateFormatVersion, Description, Metadata, Parameters, Rules, Mappings, Conditions, Transform, Resources, Outputs) → parameter types, constraints, SSM parameter types → intrinsic functions (Ref, Fn::GetAtt, Sub, Join, Select, Split, FindInMap, If, ImportValue, Base64, GetAZs) and pseudo parameters → dependencies (implicit vs DependsOn) → stack lifecycle and statuses → creating, updating, deleting stacks → update behaviors (no interruption / some interruption / replacement) → change sets → rollback behavior and failure handling → DeletionPolicy, UpdateReplacePolicy, stack policies, termination protection → drift detection → nested stacks, cross-stack references, StackSets → helper scripts (cfn-init, cfn-signal, CreationPolicy, WaitCondition) → custom resources → IaC generator and other newer features → CLI cheat sheet → three complete templates (S3, EC2 web server, VPC) → common errors → viva → master sheet.

---

Absolutely mate. 😤🔥 Part 1 gave the idea. Now the **AWS-native IaC tool: CloudFormation** (Practical 3).

---

# 1️⃣ WHAT IS CLOUDFORMATION?

> **AWS CloudFormation** is an AWS service that lets you model and provision AWS resources by writing a **template** (YAML or JSON) and then creating a **stack** from it. CloudFormation figures out the order, creates the resources, tracks them, and can update or delete them as a unit.

Key facts:

```
Declarative         → you describe WHAT, CloudFormation decides HOW
AWS only (core)     → manages AWS resources (plus third-party via the CloudFormation Registry/extensions)
Free to use         → you pay only for the resources it creates (handler operations on third-party types may be charged)
Managed by AWS      → no state file to host: AWS stores stack state for you
Regional            → a stack lives in one region and one account
```

## 1.1 Core vocabulary ⭐

| Term | Meaning |
|---|---|
| **Template** | The text file (YAML/JSON) describing resources |
| **Stack** | A running instance of a template: the group of resources managed as one unit |
| **Resource** | One AWS thing (EC2 instance, S3 bucket, VPC...) |
| **Parameters** | Inputs given at deploy time |
| **Outputs** | Values returned after deploy (e.g., public IP, bucket name) |
| **Change set** | A preview of what an update will do (like `terraform plan`) |
| **Stack events** | Log of what CloudFormation is doing and any errors |
| **Drift** | Difference between stack definition and actual resource configuration |
| **StackSet** | Deploy the same stack to many accounts/regions |

```
TEMPLATE (blueprint) ──create-stack──▶ STACK (the built house) ──▶ RESOURCES
          one template → many stacks (dev, test, prod) using different parameters
```

## 1.2 How it works behind the scenes

```
1. You upload template (directly or from S3)
2. CloudFormation validates syntax and parameters
3. Builds a dependency graph from references (Ref, GetAtt, DependsOn)
4. Calls AWS service APIs to create resources in order (in parallel when independent)
5. Records each resource's physical ID in the stack
6. If ANY resource fails → rolls back (deletes what it created) by default
7. Status: CREATE_COMPLETE (or ROLLBACK_COMPLETE on failure)
8. Outputs become available
```

---

# 2️⃣ TEMPLATE FORMATS: YAML vs JSON

| | YAML | JSON |
|---|---|---|
| Readability | ✅ Cleaner | Verbose, braces and quotes |
| Comments | ✅ `# comment` | ❌ Not supported |
| Short functions | ✅ `!Ref`, `!Sub` | Must use `{"Ref": ...}`, `{"Fn::Sub": ...}` |
| Recommendation | **Use YAML** | Legacy/generated templates |

Same snippet:

```yaml
BucketName: !Sub "${AWS::StackName}-data"
```
```json
"BucketName": { "Fn::Sub": "${AWS::StackName}-data" }
```

⚠️ YAML uses **spaces only** (no tabs); indentation is meaning.

---

# 3️⃣ TEMPLATE ANATOMY ⭐⭐⭐ (DRAW IN EXAMS)

```yaml
AWSTemplateFormatVersion: "2010-09-09"   # optional; the only valid value
Description: What this template does     # optional text

Metadata:        # optional: extra info (e.g., parameter grouping for console UI)
Parameters:      # optional: inputs
Rules:           # optional: validate parameter combinations
Mappings:        # optional: static lookup tables (region → AMI, env → size)
Conditions:      # optional: true/false logic (create resource only in prod)
Transform:       # optional: macros (AWS::Serverless-2016-10-31 for SAM, AWS::Include)
Resources:       # REQUIRED: the only mandatory section
Outputs:         # optional: values to display/export
```

💡 Memory trick: **"A D M P R M C T R O"** or just remember: **Resources is the only required section.**

## 3.1 Resources (the heart)

```yaml
Resources:
  LogicalID:                 # name inside the template (letters/numbers only)
    Type: AWS::Service::ResourceType
    Properties:
      PropertyName: value
    DependsOn: OtherLogicalID      # optional
    DeletionPolicy: Retain         # optional attributes
    Condition: IsProd              # optional
```

| Part | Meaning |
|---|---|
| **Logical ID** | Name used inside the template for references (`WebServer`) |
| **Physical ID** | Real ID AWS assigns (`i-0abc123...`, bucket name) |
| **Type** | Resource type, format `AWS::<Service>::<Resource>` e.g. `AWS::EC2::Instance`, `AWS::S3::Bucket`, `AWS::EC2::SecurityGroup`, `AWS::IAM::Role`, `AWS::RDS::DBInstance`, `AWS::Lambda::Function` |
| **Properties** | Settings for that resource (see AWS docs "Resource and property types reference") |

## 3.2 Parameters (inputs)

```yaml
Parameters:
  InstanceType:
    Type: String
    Default: t3.micro
    AllowedValues: [t3.micro, t3.small, t3.medium]
    Description: EC2 size
  KeyName:
    Type: AWS::EC2::KeyPair::KeyName       # console shows a dropdown of your key pairs
  VpcId:
    Type: AWS::EC2::VPC::Id
  SubnetId:
    Type: AWS::EC2::Subnet::Id
  LatestAmiId:
    Type: AWS::SSM::Parameter::Value<AWS::EC2::Image::Id>
    Default: /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64
  DBPassword:
    Type: String
    NoEcho: true                  # hides value in console/logs (still prefer Secrets Manager)
    MinLength: 12
  Environment:
    Type: String
    AllowedValues: [dev, prod]
    Default: dev
```

| Parameter type | Use |
|---|---|
| `String`, `Number`, `CommaDelimitedList`, `List<Number>` | Basic |
| **AWS-specific types** (`AWS::EC2::KeyPair::KeyName`, `AWS::EC2::VPC::Id`, `AWS::EC2::Subnet::Id`, `AWS::EC2::SecurityGroup::Id`...) | Validated against your account; dropdown in console |
| **SSM parameter types** (`AWS::SSM::Parameter::Value<...>`) | Pull values (like the latest AMI ID) from Systems Manager Parameter Store at deploy time → **no hard-coded AMI IDs** ✅ |

Constraints: `AllowedValues`, `AllowedPattern` (regex), `MinLength/MaxLength`, `MinValue/MaxValue`, `ConstraintDescription`, `NoEcho`.

## 3.3 Mappings (static lookup table)

```yaml
Mappings:
  EnvSizeMap:
    dev:  { Size: t3.micro }
    prod: { Size: t3.large }
  RegionAmi:
    ap-south-1: { AMI: ami-0aaa1111111111111 }     # illustrative IDs only
    us-east-1:  { AMI: ami-0bbb2222222222222 }

# usage
InstanceType: !FindInMap [EnvSizeMap, !Ref Environment, Size]
```

Mappings are fixed values written in the template. (Modern practice prefers SSM parameter types for AMIs.)

## 3.4 Conditions (if/else)

```yaml
Conditions:
  IsProd: !Equals [!Ref Environment, prod]

Resources:
  ProdOnlyAlarm:
    Type: AWS::CloudWatch::Alarm
    Condition: IsProd             # created only when Environment=prod
    Properties: { ... }

  Server:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: !If [IsProd, t3.large, t3.micro]
```

Condition functions: `Fn::Equals`, `Fn::And`, `Fn::Or`, `Fn::Not`, `Fn::If`.

## 3.5 Outputs

```yaml
Outputs:
  WebsiteURL:
    Description: Public URL of the web server
    Value: !Sub "http://${WebServer.PublicIp}"
  BucketName:
    Value: !Ref DataBucket
    Export:
      Name: !Sub "${AWS::StackName}-BucketName"    # lets OTHER stacks import it
```

Outputs show on the console/CLI after create and enable **cross-stack references**.

## 3.6 Transform and Rules (just know them)

```
Transform: AWS::Serverless-2016-10-31   → AWS SAM: write short serverless syntax that expands to CloudFormation
Transform: AWS::LanguageExtensions      → extra functions (e.g., Fn::ForEach loops, Fn::Length)
Rules: validate parameter combinations (e.g., prod must use a larger instance)
```

---

# 4️⃣ INTRINSIC FUNCTIONS AND PSEUDO PARAMETERS ⭐⭐

## 4.1 Intrinsic functions (built-in helpers)

| Function | Short form | Purpose | Example |
|---|---|---|---|
| **Ref** | `!Ref` | Returns a parameter's value, or a resource's main ID | `!Ref WebServer` → instance ID |
| **Fn::GetAtt** | `!GetAtt` | Returns an attribute of a resource | `!GetAtt WebServer.PublicIp` |
| **Fn::Sub** | `!Sub` | String substitution with `${Var}` | `!Sub "${AWS::StackName}-bucket"` |
| **Fn::Join** | `!Join` | Join strings with a delimiter | `!Join ["-", [app, !Ref Environment]]` |
| **Fn::Select** | `!Select` | Pick an item from a list | `!Select [0, !GetAZs ""]` |
| **Fn::Split** | `!Split` | Split a string into a list | `!Split [",", "a,b,c"]` |
| **Fn::FindInMap** | `!FindInMap` | Look up in Mappings | see above |
| **Fn::If / Equals / Not / And / Or** | `!If` etc. | Conditions | see above |
| **Fn::GetAZs** | `!GetAZs` | List AZs of a region | `!GetAZs ""` (current region) |
| **Fn::ImportValue** | `!ImportValue` | Use another stack's exported output | `!ImportValue NetworkStack-VpcId` |
| **Fn::Base64** | `!Base64` | Encode text (needed for EC2 UserData) | `!Base64 |` multi-line script |
| **Fn::Cidr** | `!Cidr` | Generate subnet CIDR blocks | `!Cidr [!GetAtt VPC.CidrBlock, 4, 8]` |
| **Fn::Length / ToJsonString** | | Language extensions | |

### Ref vs GetAtt (very common confusion)

```
Ref     → returns the "default" identifier (for EC2 instance: InstanceId; for S3 bucket: bucket name; for parameter: its value)
GetAtt  → returns a specific attribute (EC2 PublicIp, S3 Arn, ALB DNSName)
```

## 4.2 Pseudo parameters (predefined, no need to declare)

| Pseudo parameter | Value |
|---|---|
| `AWS::Region` | Region of the stack (e.g., ap-south-1) |
| `AWS::AccountId` | 12-digit AWS account ID |
| `AWS::StackName` | Name of the stack |
| `AWS::StackId` | Full ARN of the stack |
| `AWS::Partition` | aws / aws-cn / aws-us-gov |
| `AWS::URLSuffix` | amazonaws.com |
| `AWS::NoValue` | Removes a property (used with `Fn::If`) |

Globally unique bucket name trick:

```yaml
BucketName: !Sub "${AWS::StackName}-${AWS::AccountId}-${AWS::Region}-data"
```

(S3 names must be lowercase; stack names with uppercase letters would break this. Or simply omit `BucketName` and let CloudFormation generate one.)

---

# 5️⃣ DEPENDENCIES

## 5.1 Implicit (automatic) dependency ✅

Using `!Ref` or `!GetAtt` creates an order automatically:

```yaml
WebServer:
  Type: AWS::EC2::Instance
  Properties:
    SecurityGroupIds: [!Ref WebSG]      # CloudFormation creates WebSG first
```

## 5.2 Explicit dependency (`DependsOn`)

When there is no reference but order matters:

```yaml
PublicRoute:
  Type: AWS::EC2::Route
  DependsOn: AttachGateway        # route needs the IGW attached to the VPC first
```

---

# 6️⃣ STACK LIFECYCLE AND STATUSES ⭐

```
CREATE_IN_PROGRESS → CREATE_COMPLETE ✅
                   → ROLLBACK_IN_PROGRESS → ROLLBACK_COMPLETE (create failed; stack remains, must be deleted)
UPDATE_IN_PROGRESS → UPDATE_COMPLETE ✅
                   → UPDATE_ROLLBACK_IN_PROGRESS → UPDATE_ROLLBACK_COMPLETE (update failed; reverted to previous good state)
DELETE_IN_PROGRESS → DELETE_COMPLETE (stack gone)  or DELETE_FAILED (some resource couldn't be deleted)
```

| Status | Meaning |
|---|---|
| `CREATE_COMPLETE` | Stack created successfully |
| `ROLLBACK_COMPLETE` | Creation failed and everything was deleted; stack shell remains (can't update; delete it and retry) |
| `UPDATE_COMPLETE` | Update succeeded |
| `UPDATE_ROLLBACK_COMPLETE` | Update failed; stack returned to previous working configuration ✅ (this is the rollback to demonstrate in Practical 3) |
| `UPDATE_ROLLBACK_FAILED` | Rollback itself failed; use "Continue update rollback" after fixing cause |
| `DELETE_FAILED` | Deletion blocked (e.g., non-empty S3 bucket, dependent resource) |
| `REVIEW_IN_PROGRESS` | Stack created via change set but not executed yet |

---

# 7️⃣ CREATING, UPDATING, DELETING STACKS 🛠️

## 7.1 Console route

```
CloudFormation → Create stack → With new resources → Upload a template file (or S3 URL)
→ Specify stack name → fill Parameters → Configure options (tags, IAM role, rollback, termination protection)
→ Review → check "I acknowledge that CloudFormation might create IAM resources" if needed → Submit
→ watch the Events tab → Outputs tab when complete
```

## 7.2 CLI route

```bash
# validate syntax (does not check everything, e.g. permissions or property correctness)
aws cloudformation validate-template --template-body file://template.yaml

# create
aws cloudformation create-stack --stack-name demo-stack \
    --template-body file://template.yaml \
    --parameters ParameterKey=InstanceType,ParameterValue=t3.micro \
    --capabilities CAPABILITY_NAMED_IAM          # only if template creates IAM resources
aws cloudformation wait stack-create-complete --stack-name demo-stack

# or one command for create-or-update (recommended in scripts/pipelines)
aws cloudformation deploy --stack-name demo-stack --template-file template.yaml \
    --parameter-overrides InstanceType=t3.micro --capabilities CAPABILITY_IAM

# inspect
aws cloudformation describe-stacks --stack-name demo-stack
aws cloudformation describe-stack-events --stack-name demo-stack
aws cloudformation list-stack-resources --stack-name demo-stack

# update directly
aws cloudformation update-stack --stack-name demo-stack --template-body file://template.yaml \
    --parameters ParameterKey=InstanceType,ParameterValue=t3.small

# delete
aws cloudformation delete-stack --stack-name demo-stack
aws cloudformation wait stack-delete-complete --stack-name demo-stack
```

## 7.3 Capabilities (a safety acknowledgement)

```
CAPABILITY_IAM / CAPABILITY_NAMED_IAM → "yes, this template creates IAM resources"
CAPABILITY_AUTO_EXPAND                → "yes, this uses macros/transforms/nested apps"
```
If you forget: `InsufficientCapabilitiesException`.

---

# 8️⃣ UPDATE BEHAVIOR: WHAT HAPPENS WHEN YOU CHANGE A PROPERTY? ⭐⭐

Each property has an **update behavior** (documented per property):

| Update behavior | Meaning | Example |
|---|---|---|
| **No interruption** | Updated in place, resource keeps running | Changing instance **tags**; EC2 `SecurityGroupIds` (in a VPC) |
| **Some interruption** | In place, but with a brief outage (stop/start or reboot) | Changing EC2 **InstanceType** (instance is stopped and restarted; EBS-backed) |
| **Replacement** | Creates a **new** resource, then deletes the old one (new physical ID, data on old one lost) | Changing S3 **BucketName**, EC2 **ImageId** (AMI), subnet of an instance |

```
Replacement flow: create NEW resource → update dependents to point to NEW → delete OLD resource
```

💡 Exam point: **always check the change set for `Replacement: True`** before updating production databases or buckets, because replacement may mean data loss.

---

# 9️⃣ CHANGE SETS ⭐⭐ (CLOUDFORMATION'S `terraform plan`)

> A **change set** is a preview of the changes CloudFormation will make to a stack if you apply a new template/parameters. Nothing is changed until you **execute** it.

```
Modify template ──▶ create change set ──▶ REVIEW (Add / Modify / Remove, Replacement?) ──▶ execute (or delete)
```

```bash
aws cloudformation create-change-set --stack-name demo-stack --change-set-name cs1 \
    --template-body file://template-v2.yaml \
    --parameters ParameterKey=InstanceType,ParameterValue=t3.small

aws cloudformation describe-change-set --stack-name demo-stack --change-set-name cs1

aws cloudformation execute-change-set --stack-name demo-stack --change-set-name cs1
# or discard
aws cloudformation delete-change-set --stack-name demo-stack --change-set-name cs1
```

Sample change set output (conceptual):

```
Action   LogicalId    ResourceType            Replacement
Modify   WebServer    AWS::EC2::Instance      False    ← in-place (some interruption)
Add      MyAlarm      AWS::CloudWatch::Alarm  -
Remove   OldBucket    AWS::S3::Bucket         -
```

Limitation to remember: a change set is a **prediction**; it can't guarantee success (e.g., quota limits or conflicts discovered at execution time).

---

# 🔟 ROLLBACK, FAILURE HANDLING AND PROTECTION ⭐⭐

## 10.1 Automatic rollback

```
Create fails → resources created so far are deleted → ROLLBACK_COMPLETE (stack must be deleted before retry)
Update fails → stack reverts to previous working template/params → UPDATE_ROLLBACK_COMPLETE
```

Option when creating: **Disable rollback / "Preserve successfully provisioned resources"** keeps what was built so you can debug (not for production use by default).

## 10.2 Rollback triggers (advanced)

Attach CloudWatch alarms; if an alarm fires during/after update monitoring window, CloudFormation rolls back automatically.

## 10.3 Deletion and replacement policies

```yaml
DataBucket:
  Type: AWS::S3::Bucket
  DeletionPolicy: Retain          # stack deletion leaves the bucket alone
  UpdateReplacePolicy: Retain     # if replaced during update, keep the old one

Database:
  Type: AWS::RDS::DBInstance
  DeletionPolicy: Snapshot        # take a final snapshot before deletion
```

| DeletionPolicy | Result when stack/resource is deleted |
|---|---|
| `Delete` (default for most) | Resource deleted |
| `Retain` | Resource kept |
| `Snapshot` | Snapshot, then delete (RDS, EBS, ElastiCache, etc.) |
| `RetainExceptOnCreate` | Retain, but delete if creation itself failed |

⚠️ An **S3 bucket with objects** cannot be deleted by CloudFormation → `DELETE_FAILED` unless you empty it first.

## 10.3 Stack policy and termination protection

| Feature | Purpose |
|---|---|
| **Stack policy** | JSON policy denying updates to specific resources (e.g., protect the production DB from replacement) |
| **Termination protection** | Prevents accidental deletion of the whole stack |
| **IAM service role** | CloudFormation uses a role you choose (least privilege) instead of your own permissions |

---

# 1️⃣1️⃣ DRIFT DETECTION ⭐

> **Drift detection** compares the **actual** configuration of stack resources against the **template**. It tells you which resources were changed outside CloudFormation.

```bash
aws cloudformation detect-stack-drift --stack-name demo-stack
# returns a StackDriftDetectionId
aws cloudformation describe-stack-drift-detection-status --stack-drift-detection-id <id>
aws cloudformation describe-stack-resource-drifts --stack-name demo-stack \
    --stack-resource-drift-status-filters MODIFIED DELETED
```

Statuses: `IN_SYNC`, `MODIFIED`, `DELETED`, `NOT_CHECKED`. Drift detection **reports**; it does not auto-fix. You fix by updating the template/stack or reverting the manual change.

Demo idea: deploy an S3/SG stack → manually add an inbound rule in the console → run drift detection → see `MODIFIED` with expected vs actual JSON diff.

---

# 1️⃣2️⃣ NESTED STACKS, CROSS-STACK REFERENCES, STACKSETS

## 12.1 Why split templates?

Template size limits (1 MB from S3 / 51,200 bytes direct upload), team ownership, reuse.

## 12.2 Three patterns

| Pattern | How | Use |
|---|---|---|
| **Cross-stack references** | Stack A `Export`s an output; Stack B uses `!ImportValue` | Shared network stack used by many app stacks (loose coupling; can't delete exporting stack while imports exist) |
| **Nested stacks** | A parent template creates child stacks with `AWS::CloudFormation::Stack` and a `TemplateURL` (S3) | Reusable components (e.g., a standard VPC module) managed as one unit |
| **StackSets** | Deploy one template to **multiple accounts and regions** | Org-wide baselines (IAM roles, logging, guardrails) |

```yaml
# Nested stack usage
Resources:
  NetworkStack:
    Type: AWS::CloudFormation::Stack
    Properties:
      TemplateURL: https://my-bucket.s3.amazonaws.com/vpc.yaml
      Parameters:
        CidrBlock: 10.0.0.0/16
  AppStack:
    Type: AWS::CloudFormation::Stack
    Properties:
      TemplateURL: https://my-bucket.s3.amazonaws.com/app.yaml
      Parameters:
        VpcId: !GetAtt NetworkStack.Outputs.VpcId
```

---

# 1️⃣3️⃣ BOOTSTRAPPING EC2: USERDATA, cfn-init, cfn-signal 🧰

| Method | Use |
|---|---|
| **UserData** | Shell script at first boot (simple, procedural) |
| **cfn-init + AWS::CloudFormation::Init metadata** | Declarative configuration of packages/files/services on the instance (idempotent) |
| **cfn-signal + CreationPolicy** | Instance tells CloudFormation "I'm fully configured"; stack waits for the signal before `CREATE_COMPLETE` |
| **WaitCondition** | Older wait mechanism for signaling |

Simple pattern (used in the lab):

```yaml
UserData:
  Fn::Base64: !Sub |
    #!/bin/bash
    dnf install -y nginx
    systemctl enable --now nginx
    echo "<h1>Hello from ${AWS::StackName}</h1>" > /usr/share/nginx/html/index.html
```

Note: `!Sub |` with `${...}` — any literal `${}` for shell must be written `${!Var}`.

---

# 1️⃣4️⃣ CUSTOM RESOURCES AND NEWER FEATURES (AWARENESS)

| Feature | What |
|---|---|
| **Custom resources** | Run your own Lambda during stack events to do things CloudFormation can't (e.g., call an external API) |
| **Macros / Transforms** | Template pre-processing (SAM, Include, LanguageExtensions) |
| **CloudFormation Registry / extensions** | Third-party and private resource types, modules, hooks |
| **CloudFormation Hooks** | Proactive checks that can block non-compliant resources before provisioning |
| **CloudFormation Guard** | Policy-as-code rules (`cfn-guard`) |
| **IaC generator** | Scan existing resources and generate templates (helps bring ClickOps resources under IaC); resource import also exists |
| **Git sync** | Sync a stack from a Git repository |
| **AWS CDK** | Write infra in Python/TypeScript/etc.; `cdk synth` produces CloudFormation templates (Part 6) |
| **AWS SAM** | Shorthand for serverless apps; expands into CloudFormation |
| **cfn-lint** | Linter that catches errors before deployment |

Check the AWS CloudFormation release notes for the newest additions; the service evolves frequently.

---

# 1️⃣5️⃣ COMPLETE TEMPLATE #1: S3 BUCKET (SIMPLEST PRACTICE)

`s3-bucket.yaml`:

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: Simple versioned, encrypted, private S3 bucket

Parameters:
  Environment:
    Type: String
    Default: dev
    AllowedValues: [dev, test, prod]

Resources:
  DataBucket:
    Type: AWS::S3::Bucket
    DeletionPolicy: Delete
    Properties:
      VersioningConfiguration:
        Status: Enabled
      BucketEncryption:
        ServerSideEncryptionConfiguration:
          - ServerSideEncryptionByDefault:
              SSEAlgorithm: AES256
      PublicAccessBlockConfiguration:
        BlockPublicAcls: true
        BlockPublicPolicy: true
        IgnorePublicAcls: true
        RestrictPublicBuckets: true
      Tags:
        - Key: Environment
          Value: !Ref Environment

Outputs:
  BucketName:
    Value: !Ref DataBucket
  BucketArn:
    Value: !GetAtt DataBucket.Arn
```

(No `BucketName` given, so CloudFormation generates a unique one, avoiding name clashes.)

---

# 1️⃣6️⃣ COMPLETE TEMPLATE #2: EC2 WEB SERVER WITH SECURITY GROUP (PRACTICAL 3 CORE) ⭐

`webserver.yaml`:

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: EC2 web server (Nginx) with a security group in an existing VPC/subnet

Parameters:
  VpcId:
    Type: AWS::EC2::VPC::Id
    Description: VPC for the security group
  SubnetId:
    Type: AWS::EC2::Subnet::Id
    Description: A PUBLIC subnet in that VPC
  KeyName:
    Type: AWS::EC2::KeyPair::KeyName
    Description: Existing key pair for SSH
  MyIpCidr:
    Type: String
    Description: Your IP in CIDR form for SSH, e.g. 203.0.113.25/32
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
      GroupDescription: Allow HTTP from anywhere and SSH from my IP
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
          echo "<h1>Deployed by CloudFormation stack ${AWS::StackName} in ${AWS::Region}</h1>" > /usr/share/nginx/html/index.html
      Tags:
        - Key: Name
          Value: !Sub "${AWS::StackName}-web"

Outputs:
  InstanceId:
    Description: EC2 instance ID
    Value: !Ref WebServer
  PublicIP:
    Description: Public IP address
    Value: !GetAtt WebServer.PublicIp
  WebsiteURL:
    Description: Open this in your browser
    Value: !Sub "http://${WebServer.PublicIp}"
```

Notes:
- The instance gets a public IP only if the chosen subnet **auto-assigns public IPs** (default VPC subnets do). Otherwise add a `NetworkInterfaces` block with `AssociatePublicIpAddress: true` or attach an Elastic IP resource.
- The AMI is fetched from the SSM public parameter at deploy time: no hard-coded AMI ID.
- Using `AllowedValues` on `InstanceType` is how the "rollback demo" can use an invalid value later (Part 7 lab).

---

# 1️⃣7️⃣ COMPLETE TEMPLATE #3: VPC WITH PUBLIC SUBNET (NETWORKING AS CODE)

`vpc.yaml` (mirrors Unit I Part 8 manual steps):

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: VPC with one public subnet, internet gateway and route

Parameters:
  VpcCidr:
    Type: String
    Default: 10.0.0.0/16
  PublicSubnetCidr:
    Type: String
    Default: 10.0.1.0/24

Resources:
  VPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: !Ref VpcCidr
      EnableDnsSupport: true
      EnableDnsHostnames: true
      Tags: [{ Key: Name, Value: !Sub "${AWS::StackName}-vpc" }]

  InternetGateway:
    Type: AWS::EC2::InternetGateway
    Properties:
      Tags: [{ Key: Name, Value: !Sub "${AWS::StackName}-igw" }]

  AttachGateway:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      VpcId: !Ref VPC
      InternetGatewayId: !Ref InternetGateway

  PublicSubnet:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: !Ref PublicSubnetCidr
      AvailabilityZone: !Select [0, !GetAZs ""]
      MapPublicIpOnLaunch: true
      Tags: [{ Key: Name, Value: !Sub "${AWS::StackName}-public-subnet" }]

  PublicRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref VPC

  PublicRoute:
    Type: AWS::EC2::Route
    DependsOn: AttachGateway
    Properties:
      RouteTableId: !Ref PublicRouteTable
      DestinationCidrBlock: 0.0.0.0/0
      GatewayId: !Ref InternetGateway

  SubnetRouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PublicSubnet
      RouteTableId: !Ref PublicRouteTable

Outputs:
  VpcId:
    Value: !Ref VPC
    Export: { Name: !Sub "${AWS::StackName}-VpcId" }
  PublicSubnetId:
    Value: !Ref PublicSubnet
    Export: { Name: !Sub "${AWS::StackName}-PublicSubnetId" }
```

This is the **Unit I manual VPC** turned into code in ~60 lines: exactly the point of IaC. Combine with `webserver.yaml` via `!ImportValue` or parameters.

---

# 1️⃣8️⃣ DRY RUN: CREATING THE WEB SERVER STACK

```
Step 1  aws cloudformation deploy → template uploaded, parameters resolved
        (LatestAmiId SSM value fetched: ami-xxxx)
Step 2  Dependency graph:  WebSecurityGroup ──▶ WebServer   (WebServer uses !Ref WebSecurityGroup)
Step 3  CREATE_IN_PROGRESS WebSecurityGroup → CREATE_COMPLETE
Step 4  CREATE_IN_PROGRESS WebServer (launch, boot, UserData runs) → CREATE_COMPLETE
Step 5  Stack CREATE_COMPLETE ; Outputs: PublicIP=13.x.x.x, WebsiteURL=http://13.x.x.x
Step 6  Browser opens URL → page "Deployed by CloudFormation stack ..."
Delete: delete-stack → WebServer deleted first, then SecurityGroup (reverse order) → DELETE_COMPLETE
```

---

# 1️⃣9️⃣ COMMON ERRORS TABLE 🛠️

| Error / Symptom | Cause | Fix |
|---|---|---|
| `Template format error: YAML not well-formed` | Tabs/indent/quote issue | Use spaces, `cfn-lint`, validate-template |
| `Template format error: Unresolved resource dependencies` | `!Ref` to something undefined | Fix logical ID spelling |
| `Parameters: [X] must have values` | Missing parameter | Pass `--parameter-overrides` |
| `InsufficientCapabilitiesException` | IAM resources without capability | Add `--capabilities CAPABILITY_IAM` |
| `ROLLBACK_COMPLETE` on create | A resource failed (see Events: "Status reason") | Read the first `CREATE_FAILED` event; fix; delete stack; recreate |
| `AlreadyExists` for S3 bucket | Hard-coded global name taken | Omit BucketName or use `${AWS::AccountId}` |
| `The security group 'sg-x' does not exist in VPC` | SG and subnet in different VPCs | Use same VPC for both |
| `Value (t3.superlarge) for parameter instanceType is invalid` | Wrong property value | Fix value; this triggers rollback on update |
| `DELETE_FAILED` (S3) | Bucket not empty | Empty bucket, then delete again |
| Instance created but website unreachable | No public IP, SG port 80 missing, Nginx not started | Check subnet public IP setting, SG, UserData log `/var/log/cloud-init-output.log` |
| `Export cannot be deleted as it is in use` | Another stack imports it | Remove import first |
| Stack stuck `UPDATE_ROLLBACK_FAILED` | Rollback couldn't revert | Fix cause, "Continue update rollback" (skip resources if needed) |

---

# 2️⃣0️⃣ BEST PRACTICES ✅

```
Use YAML; keep templates small and modular; lint with cfn-lint
Parameterize environment differences; avoid hard-coded IDs/names (use SSM params, auto-generated names)
Use change sets for every update to production
Enable termination protection + DeletionPolicy/stack policy for critical resources
Tag everything (Project, Owner, Environment)
Use IAM service roles with least privilege
Store templates in Git; deploy through CI/CD (aws cloudformation deploy)
Run drift detection regularly; never fix prod by hand without code follow-up
Never put secrets in templates (use Secrets Manager dynamic references: {{resolve:secretsmanager:...}})
```

---

# 2️⃣1️⃣ VIVA QUESTIONS (PART 2) 🎯

**Q1. What is CloudFormation?** AWS service for modeling and provisioning AWS resources declaratively from YAML/JSON templates managed as stacks.

**Q2. Template vs stack?** Template is the blueprint file; stack is the deployed set of resources created from it.

**Q3. Mandatory section of a template?** `Resources`.

**Q4. Ref vs GetAtt?** Ref returns a parameter value or the resource's default identifier; GetAtt returns a named attribute like PublicIp or Arn.

**Q5. What is a change set?** A preview of changes (add/modify/remove, replacement) before updating a stack.

**Q6. What happens if stack creation fails?** Automatic rollback deletes created resources, leaving `ROLLBACK_COMPLETE`.

**Q7. What is `UPDATE_ROLLBACK_COMPLETE`?** Update failed and the stack returned to its previous working state.

**Q8. Update behaviors?** No interruption, some interruption, replacement.

**Q9. What is drift detection?** Checking whether actual resources differ from the template definition.

**Q10. Nested stacks vs cross-stack references?** Nested stacks are child stacks created by a parent; cross-stack references share exported outputs between independent stacks.

**Q11. What is a StackSet?** Deploys a template across multiple accounts and regions.

**Q12. DeletionPolicy options?** Delete, Retain, Snapshot (and RetainExceptOnCreate).

**Q13. Why use SSM parameter type for AMI?** Always fetch the latest AMI without hard-coding region-specific IDs.

**Q14. What does `Fn::Sub` do?** Substitutes variables inside a string.

**Q15. Is CloudFormation multi-cloud?** No, it is AWS-centric (extensions can manage third-party resources, but it is not a multi-cloud tool like Terraform).

---

# 🧠 PART 2 MASTER SHEET

```
CFN = AWS-native, declarative IaC ; template (YAML/JSON) → stack → resources ; state kept by AWS
Sections: Version, Description, Metadata, Parameters, Rules, Mappings, Conditions, Transform, Resources(required), Outputs
Functions: Ref, GetAtt, Sub, Join, Select, Split, FindInMap, If, ImportValue, Base64, GetAZs ; Pseudo: Region, AccountId, StackName
Dependencies: implicit (Ref/GetAtt) + DependsOn
Statuses: CREATE_COMPLETE, ROLLBACK_COMPLETE, UPDATE_COMPLETE, UPDATE_ROLLBACK_COMPLETE, DELETE_FAILED
Update types: no interruption | some interruption | REPLACEMENT (check change set!)
Change set = preview ; Drift detection = reality vs template ; Rollback = automatic revert
Policies: DeletionPolicy, UpdateReplacePolicy, stack policy, termination protection
Reuse: nested stacks, cross-stack Export/ImportValue, StackSets
CLI: validate-template, create-stack/deploy, create-change-set, execute-change-set, detect-stack-drift, delete-stack
```

---

## 🫡 PART 2 STATUS

You can now read, write, deploy, update, preview, roll back and delete CloudFormation stacks, and you have the exact templates for Practical 3.

**STOP here.** Next up:

> **PART 3 — Terraform from Scratch (HCL, providers, init/plan/apply/destroy, variables, outputs, data sources, state) 🔥**
