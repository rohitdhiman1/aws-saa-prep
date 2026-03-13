# Infrastructure as Code — CloudFormation, SAM, CDK

*Phase 5 · Week 10 · [roadmap day-by-day](../docs/roadmap.md)*

---

## What It Is

AWS services for defining, provisioning, and managing infrastructure as code (IaC). CloudFormation is the foundation; SAM and CDK build on top of it for serverless and general-purpose development respectively.

---

## How It Works

### CloudFormation

Declarative IaC service. You define resources in a **template** (JSON or YAML), and CloudFormation provisions them as a **stack**.

#### Template Anatomy

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: "My stack"

Parameters:       # Input values (instance type, env name, etc.)
Mappings:         # Static lookup tables (AMI per region, etc.)
Conditions:       # Conditional resource creation (if prod, create larger instance)
Resources:        # REQUIRED — the AWS resources to create
Outputs:          # Values to export (VPC ID, ALB DNS, etc.)
```

- **Resources** is the only required section.
- **Pseudo-parameters** — `AWS::Region`, `AWS::AccountId`, `AWS::StackName` (auto-populated).
- **Intrinsic functions** — `!Ref`, `!GetAtt`, `!Sub`, `!Join`, `!Select`, `!If`, `!ImportValue`, `Fn::Base64`.

#### Stacks & Stack Sets

| Concept | What |
|---|---|
| **Stack** | Single unit of deployment (one template → one stack) |
| **Nested stacks** | Stack that includes other stacks via `AWS::CloudFormation::Stack`. Reuse common patterns (VPC, ALB, SG). |
| **Cross-stack references** | Stack A exports an output → Stack B imports it with `!ImportValue`. Loose coupling between stacks. |
| **Stack Sets** | Deploy a stack across multiple accounts and regions in one operation (requires AWS Organisations or self-managed permissions). |

#### Change Sets

- Preview changes before applying them. Shows: Add, Modify, Remove, Replace.
- **Replace** = resource will be **deleted and recreated** (e.g., changing an RDS instance identifier). Data loss risk.
- **Update behaviours:**
  - **No interruption** — update in place (e.g., adding a tag).
  - **Some interruption** — brief downtime (e.g., changing instance type with stop/start).
  - **Replacement** — new resource created, old one deleted (e.g., changing RDS engine).

#### Drift Detection

- Detect when actual resource config has diverged from the template (manual console changes, CLI modifications).
- Shows: IN_SYNC, MODIFIED, DELETED, NOT_CHECKED.
- Does not auto-fix; you must update the template or the resource.

#### Rollback & Failure Handling

- **On create failure** — default: rollback all resources (delete stack). Option: `--disable-rollback` to preserve for debugging.
- **On update failure** — automatic rollback to previous working state.
- **Stack policies** — protect specific resources from unintended updates (e.g., prevent deletion of production RDS).
- **Termination protection** — prevent accidental stack deletion.
- **DeletionPolicy:**
  - `Delete` (default) — resource deleted with the stack.
  - `Retain` — keep the resource after stack deletion (e.g., S3 bucket with data).
  - `Snapshot` — create a snapshot before deletion (supported: RDS, EBS, ElastiCache, Redshift, Neptune).

#### cfn-init, cfn-signal & CreationPolicy

- **cfn-init** — install packages, create files, start services on EC2 during launch (more structured than UserData).
- **cfn-signal** — EC2 sends a success/failure signal back to CloudFormation.
- **CreationPolicy** — CloudFormation waits for the signal before marking the resource as CREATE_COMPLETE. Timeout configurable.
- **WaitCondition** — similar to CreationPolicy but for external resources or more complex orchestration.

---

### SAM — Serverless Application Model

An extension of CloudFormation specifically for **serverless** applications.

#### Key Points

- SAM templates are CloudFormation templates with a `Transform: AWS::Serverless-2016-10-31` header.
- **Simplified resource types:**
  - `AWS::Serverless::Function` → Lambda + IAM role + event source
  - `AWS::Serverless::Api` → API Gateway
  - `AWS::Serverless::SimpleTable` → DynamoDB table
  - `AWS::Serverless::HttpApi` → HTTP API (cheaper than REST)
  - `AWS::Serverless::LayerVersion` → Lambda Layer
  - `AWS::Serverless::StateMachine` → Step Functions
- **SAM CLI** — `sam build`, `sam local invoke`, `sam deploy` (local testing with Docker).
- SAM is transformed into standard CloudFormation at deploy time.
- **SAM Policy Templates** — pre-built IAM policies (e.g., `DynamoDBCrudPolicy`, `S3ReadPolicy`).

> **Exam tip:** SAM appears when the question mentions "serverless deployment" or "Lambda + API Gateway packaging." The answer is SAM, not raw CloudFormation.

---

### CDK — Cloud Development Kit

Define infrastructure using **programming languages** (TypeScript, Python, Java, C#, Go). CDK synthesises CloudFormation templates.

#### Key Concepts

- **App** — top-level construct; represents the entire CDK application.
- **Stack** — unit of deployment (maps 1:1 to a CloudFormation stack).
- **Construct** — building block. Three levels:
  - **L1** — direct CloudFormation mapping (`CfnBucket`).
  - **L2** — AWS-curated with sensible defaults (`Bucket` with encryption enabled by default).
  - **L3 (Patterns)** — complete architectures (e.g., `LambdaRestApi` = Lambda + API Gateway + IAM).
- **`cdk synth`** — generate CloudFormation template.
- **`cdk deploy`** — deploy the stack.
- **`cdk diff`** — preview changes (like a change set).
- **`cdk bootstrap`** — create the CDK toolkit stack (S3 bucket + IAM roles) in the target account/region.

> **Exam tip:** CDK appears when the question mentions "define infrastructure in Python/TypeScript" or "developer-friendly IaC." CDK synthesises to CloudFormation.

---

### CloudFormation vs SAM vs CDK vs Terraform

| Feature | CloudFormation | SAM | CDK | Terraform |
|---|---|---|---|---|
| Language | YAML/JSON | YAML (extended) | TypeScript/Python/etc. | HCL |
| Scope | All AWS services | Serverless focus | All AWS services | Multi-cloud |
| Abstraction | Low (verbose) | Medium (serverless shortcuts) | High (L2/L3 constructs) | Medium |
| Local testing | No | Yes (SAM CLI + Docker) | No (but `cdk diff`) | No |
| State management | AWS-managed (stack) | AWS-managed | AWS-managed | Self-managed or Terraform Cloud |
| Output | Direct deployment | → CloudFormation | → CloudFormation | Direct deployment |

---

## Key Configs & Limits

| Item | Limit |
|---|---|
| Max resources per stack | 500 |
| Max stacks per account per region | 2,000 (soft limit) |
| Template max size (S3) | 1 MB |
| Template max size (direct upload) | 51,200 bytes |
| Max parameters per template | 200 |
| Max outputs per template | 200 |
| Max mappings per template | 200 |
| Nested stack depth | Unlimited (but practical limit ~5) |
| Stack Set max concurrent accounts | 1 per region (default) |

---

## When To Use

| Scenario | Solution |
|---|---|
| Provision any AWS resource declaratively | CloudFormation |
| Deploy Lambda + API Gateway + DynamoDB | SAM |
| Local Lambda testing before deploy | SAM CLI |
| Define infra in Python/TypeScript with defaults | CDK |
| Deploy same stack to 50 accounts | CloudFormation Stack Sets |
| Reuse a VPC pattern across stacks | Nested stacks or cross-stack references |
| Preview what will change before deploying | Change Sets (CFN) or `cdk diff` (CDK) |
| Detect manual config changes | Drift detection |
| Protect RDS from accidental deletion | DeletionPolicy: Snapshot + Stack Policy |
| Multi-cloud infrastructure | Terraform (not AWS-native) |

---

## Exam Traps

- **Resources is the only required section** in a CloudFormation template. Parameters, Outputs, Mappings, Conditions are optional.
- **Change Set does not execute changes** — it only previews them. You must explicitly execute the change set.
- **Replacement means data loss** — if CloudFormation shows "Replacement" for an RDS instance, the old one will be deleted. Use DeletionPolicy: Snapshot.
- **Nested stacks vs cross-stack references** — nested = lifecycle coupled (deploy together); cross-stack = independent lifecycle (deploy separately).
- **Stack Sets require trusted access** in AWS Organisations, or self-managed IAM roles in each target account.
- **SAM ≠ a separate service** — SAM templates are CloudFormation templates with a transform. They deploy through CloudFormation.
- **CDK ≠ a separate service** — CDK generates CloudFormation templates. The actual deployment is still CloudFormation.
- **Drift detection does not fix drift** — it only reports it. You must reconcile manually.
- **`!Ref` returns different things** — for parameters: the value. For resources: the physical ID (e.g., EC2 instance ID, S3 bucket name).
- **DeletionPolicy: Retain** means the resource stays even after the stack is deleted — you must manually clean it up.
- **cfn-signal + CreationPolicy** — without these, CloudFormation considers EC2 CREATE_COMPLETE as soon as the instance starts, even if the app hasn't finished installing.

---

## Related Files

| File | Purpose |
|---|---|
| `concepts/compute.md` | Lambda, ECS, Elastic Beanstalk deployment |
| `concepts/messaging.md` | Step Functions state machines (SAM can define these) |
| `concepts/governance.md` | Control Tower uses CloudFormation StackSets internally |
| `cheatsheets/compute.md` | Quick-reference for deployment options |
