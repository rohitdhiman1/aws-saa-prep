# Lab: Systems Manager — Parameter Store, Session Manager & Automation

*Phase 5 · Security · Concept: [concepts/security.md](../concepts/security.md)*

---

## Goal

Use SSM Parameter Store to manage configuration and secrets, connect to an EC2 instance via Session Manager (no SSH, no bastion), and run an SSM Automation document to patch instances. Understand when to use Parameter Store vs Secrets Manager.

---

## Architecture

```
SSM Parameter Store                    SSM Session Manager
┌──────────────────┐                   ┌──────────────┐
│ /app/config/db   │ (String)          │ EC2 Instance │ ← SSM Agent
│ /app/config/port │ (String)          │ (private     │   (no SSH key,
│ /app/secret/key  │ (SecureString)    │  subnet, no  │    no port 22,
└──────────────────┘                   │  public IP)  │    no bastion)
        │                              └──────┬───────┘
        ▼                                     │
Lambda reads params at runtime         You connect via
                                       SSM Session Manager
```

---

## Part A — SSM Parameter Store Basics

**1. Create parameters (hierarchy)**
- [ ] Systems Manager → Parameter Store → Create parameter.

| Name | Type | Value |
|---|---|---|
| `/app/config/db-host` | String | `mydb.cluster-xyz.us-east-1.rds.amazonaws.com` |
| `/app/config/db-port` | String | `3306` |
| `/app/config/environment` | String | `production` |
| `/app/secret/db-password` | **SecureString** | `SuperSecret123!` |
| `/app/secret/api-key` | **SecureString** | `sk-abc123xyz` |

- [ ] For SecureString, use the default AWS managed KMS key (`aws/ssm`).
- [ ] Note the hierarchy: `/app/config/...` for plain config, `/app/secret/...` for encrypted values.

**2. Retrieve parameters via CLI**
- [ ] In CloudShell:
  ```bash
  # Get a single parameter
  aws ssm get-parameter --name /app/config/db-host

  # Get a SecureString (encrypted by default)
  aws ssm get-parameter --name /app/secret/db-password
  # Value shows as encrypted ciphertext

  # Decrypt it
  aws ssm get-parameter --name /app/secret/db-password --with-decryption
  # Value shows plaintext
  ```

**3. Get parameters by path (hierarchy fetch)**
- [ ] ```bash
  # Get all config params
  aws ssm get-parameters-by-path --path /app/config

  # Get all secrets (decrypted)
  aws ssm get-parameters-by-path --path /app/secret --with-decryption

  # Get everything under /app
  aws ssm get-parameters-by-path --path /app --recursive --with-decryption
  ```
- [ ] This hierarchy is powerful — a Lambda function can fetch all its config with one API call using the path prefix.

**4. Parameter versioning**
- [ ] Update `/app/config/environment` to `staging`.
  ```bash
  aws ssm put-parameter --name /app/config/environment --value staging --overwrite
  ```
- [ ] Get the version history:
  ```bash
  aws ssm get-parameter-history --name /app/config/environment
  ```
- [ ] Note: version 1 = `production`, version 2 = `staging`. You can reference a specific version: `/app/config/environment:1`.

---

## Part B — Parameter Store vs Secrets Manager

| Feature | Parameter Store | Secrets Manager |
|---|---|---|
| Cost | Free (Standard), $0.05/10K API calls (Advanced) | $0.40/secret/month + $0.05/10K API calls |
| Auto-rotation | No | Yes (built-in Lambda rotation for RDS, Redshift, DocumentDB) |
| Max size | 4 KB (Standard), 8 KB (Advanced) | 64 KB |
| Hierarchy / paths | Yes (`/app/config/db`) | No (flat names) |
| Cross-account access | No (same account only) | Yes (resource-based policy) |
| CloudFormation dynamic ref | `{{resolve:ssm:/path}}` | `{{resolve:secretsmanager:name}}` |
| Best for | App config, feature flags, non-rotating secrets | Database credentials, API keys that need rotation |

**Exam decision:**
| Scenario | Use |
|---|---|
| "Store database connection string" | Either works; Parameter Store is cheaper |
| "Credentials that must rotate every 30 days" | Secrets Manager |
| "Hierarchical config for microservices" | Parameter Store (path-based) |
| "Cross-account secret sharing" | Secrets Manager (resource policy) |
| "Cheapest option for non-sensitive config" | Parameter Store Standard (free) |

---

## Part C — Session Manager (Bastion-Free EC2 Access)

**1. Create an EC2 instance in a private subnet (no public IP, no SSH key)**
- [ ] EC2 → Launch instance.
- [ ] Name: `ssm-lab-instance`. AMI: Amazon Linux 2023.
- [ ] Instance type: `t3.micro`.
- [ ] Key pair: **Proceed without a key pair** (no SSH).
- [ ] Network: your VPC, **private subnet**, Auto-assign public IP: **Disable**.
- [ ] Security group: create new → **no inbound rules at all** (no port 22).
- [ ] IAM instance profile: create or select a role with `AmazonSSMManagedInstanceCore` policy.
- [ ] Launch.

**2. Verify SSM Agent connectivity**
- [ ] Systems Manager → Fleet Manager → your instance should appear as "Online".
- [ ] If it doesn't appear (private subnet, no internet):
  - Option A: NAT Gateway in the VPC (instance reaches SSM endpoints via internet).
  - Option B: **VPC Endpoints** for SSM (no internet needed):
    - Create 3 Interface Endpoints: `com.amazonaws.<region>.ssm`, `com.amazonaws.<region>.ssmmessages`, `com.amazonaws.<region>.ec2messages`.
    - Attach to the private subnet. Enable private DNS.
  - [ ] After endpoints are created, the instance should appear in Fleet Manager within 5 minutes.

**3. Connect via Session Manager**
- [ ] Systems Manager → Session Manager → Start session → select `ssm-lab-instance`.
- [ ] You get a shell — no SSH key, no bastion host, no port 22, no public IP.
- [ ] Run:
  ```bash
  whoami          # ssm-user
  curl ifconfig.me  # fails (no internet) — this is a private, locked-down instance

  # Read a parameter from inside the instance
  aws ssm get-parameter --name /app/config/db-host --region us-east-1
  ```

**4. Session Manager advantages over bastion hosts**

| Bastion Host | Session Manager |
|---|---|
| Requires public subnet + EIP | No public IP needed |
| Requires SSH key management | No keys |
| Requires port 22 open in SG | No inbound ports |
| Additional EC2 cost | Free (SSM agent is pre-installed) |
| Audit: SSH logs only | Full CloudTrail + Session logs (S3 or CloudWatch) |

**Exam answer:** "Connect to EC2 in a private subnet without a bastion" → Session Manager + SSM VPC Endpoints.

---

## Part D — SSM Automation (Run Command)

**1. Run a command on the instance**
- [ ] Systems Manager → Run Command → Run command.
- [ ] Command document: `AWS-RunShellScript`.
- [ ] Targets: select your instance.
- [ ] Command:
  ```bash
  yum check-update
  uname -r
  cat /etc/os-release
  ```
- [ ] Run. View output when complete.

**2. Patch Manager (observation)**
- [ ] Systems Manager → Patch Manager → Configure patching.
- [ ] Observe: you can create a **patch baseline** (which patches to approve), a **maintenance window** (when to patch), and target instances by tags.
- [ ] Patch Manager uses SSM Automation under the hood — `AWS-RunPatchBaseline` document.
- [ ] Don't configure — just observe the options.

**Exam pattern:** "Automate OS patching across 100 instances" → SSM Patch Manager with maintenance windows.

---

## Part E — Lambda Reading from Parameter Store

- [ ] Lambda → Create function → `ssm-reader` → Python 3.12.
- [ ] Execution role: attach `AmazonSSMReadOnlyAccess` policy (and `kms:Decrypt` for SecureStrings).
- [ ] Code:
  ```python
  import json
  import boto3

  ssm = boto3.client('ssm')

  def lambda_handler(event, context):
      # Fetch all config under /app
      response = ssm.get_parameters_by_path(
          Path='/app',
          Recursive=True,
          WithDecryption=True
      )
      params = {p['Name']: p['Value'] for p in response['Parameters']}
      return {
          'statusCode': 200,
          'body': json.dumps(params)
      }
  ```
- [ ] Test → should return all 5 parameters with decrypted values.
- [ ] This pattern avoids hardcoding config and secrets in environment variables or code.

---

## Key Things to Observe

- Parameter Store hierarchy (`/app/config/...`) enables fetching all related params with one API call.
- SecureString uses KMS encryption — the caller needs both `ssm:GetParameter` and `kms:Decrypt` permissions.
- Session Manager eliminates bastion hosts — no SSH keys, no port 22, no public IP. Audit via CloudTrail.
- SSM VPC Endpoints allow Session Manager to work in fully private subnets with no internet access.
- Parameter Store is free (Standard tier). Secrets Manager costs $0.40/secret/month but adds automatic rotation.
- SSM Run Command and Patch Manager automate operational tasks without SSH access.

---

## Clean Up

- Lambda → delete `ssm-reader`.
- EC2 → terminate `ssm-lab-instance`. Delete the security group.
- If you created VPC Endpoints for SSM → delete them (they cost ~$0.01/hr each).
- SSM Parameter Store → delete all 5 parameters.
- IAM → delete the SSM instance profile role (if created for this lab).
