# Lab: GuardDuty + EventBridge — Threat Detection & Automated Remediation

*Phase 5 · Security · Concept: [concepts/security.md](../concepts/security.md)*

---

## Goal

Enable GuardDuty, generate a real finding using a known-malicious IP, build an EventBridge rule that fires when a finding appears, and wire it to a Lambda function that automatically blocks the offending IP in a Security Group. This is the exact automated-remediation pattern the exam tests.

---

## Architecture

```
GuardDuty detects suspicious traffic
       │
       ▼
EventBridge rule (source: aws.guardduty, detail-type: GuardDuty Finding)
       │
       ▼
Lambda (reads finding → extracts bad IP → removes/blocks SG inbound rule)
       │
       ▼
Security Group updated — attacker blocked
```

---

## Part A — Enable GuardDuty

- [ ] GuardDuty → Get started → Enable GuardDuty.
  - It immediately starts analysing VPC Flow Logs, CloudTrail, DNS logs — no agents needed.
- [ ] Leave all data sources enabled (default).

GuardDuty generates findings in categories:
- **Recon**: Port scans, credential enumeration attempts.
- **Trojan**: EC2 communicating with known C2 (command-and-control) IPs.
- **Stealth**: CloudTrail disabled, password policy weakened.
- **Backdoor**: EC2 or account used for Bitcoin mining, DDoS bots.

---

## Part B — Generate a Sample Finding

AWS provides a built-in way to generate sample findings without actually attacking anything:

- [ ] GuardDuty → Settings → Sample findings → Generate sample findings.
- [ ] GuardDuty → Findings → You'll see ~40 sample findings appear within ~30 seconds.
- [ ] Click on `UnauthorizedAccess:EC2/SSHBruteForce` — read the finding details:
  - Which EC2 was targeted.
  - What remote IP was the source.
  - Threat intelligence feed that flagged the IP.
  - Severity (Low/Medium/High/Critical).

**Alternatively — trigger a real finding:**
- [ ] Spin up an EC2 with a public IP and a security group allowing SSH from `0.0.0.0/0`.
- [ ] Wait 15–30 minutes — GuardDuty will flag it with `Recon:EC2/PortProbeUnprotectedPort` if port scanners hit it.
- [ ] This is common enough on fresh EC2s with open ports.

---

## Part C — EventBridge Rule on GuardDuty Findings

**1. Create a Lambda function for remediation**
- [ ] Lambda → Create function: `guardduty-remediation`. Runtime: Python 3.12.
- [ ] Execution role: create new role with:
  - `AmazonEC2ReadOnlyAccess` (to describe security groups)
  - A custom inline policy to update security groups:
    ```json
    {
      "Version": "2012-10-17",
      "Statement": [{
        "Effect": "Allow",
        "Action": [
          "ec2:RevokeSecurityGroupIngress",
          "ec2:AuthorizeSecurityGroupIngress"
        ],
        "Resource": "*"
      }]
    }
    ```

**2. Lambda code:**
```python
import boto3, json

ec2 = boto3.client('ec2')

def lambda_handler(event, context):
    # Extract finding details from the EventBridge event
    detail = event.get('detail', {})
    finding_type = detail.get('type', '')
    severity = detail.get('severity', 0)

    print(f"Finding type: {finding_type}, Severity: {severity}")

    # Only remediate High/Critical severity SSH brute force
    if severity < 7 or 'SSHBruteForce' not in finding_type:
        print("Not actioning this finding")
        return {'action': 'skipped', 'reason': 'low severity or different finding type'}

    # Extract the attacking IP
    service = detail.get('service', {})
    action = service.get('action', {})
    remote_ip = (action
                 .get('networkConnectionAction', {})
                 .get('remoteIpDetails', {})
                 .get('ipAddressV4', ''))

    if not remote_ip:
        print("No remote IP found in finding")
        return {'action': 'skipped', 'reason': 'no IP'}

    print(f"Blocking IP: {remote_ip}")

    # Get the target EC2 instance's security group
    resource = detail.get('resource', {})
    instance_id = (resource
                   .get('instanceDetails', {})
                   .get('instanceId', ''))

    if not instance_id:
        print("No instance ID in finding")
        return {'action': 'skipped', 'reason': 'no instance'}

    # Describe the instance to get its security group
    resp = ec2.describe_instances(InstanceIds=[instance_id])
    sg_id = resp['Reservations'][0]['Instances'][0]['SecurityGroups'][0]['GroupId']

    # Revoke the SSH inbound rule for the attacking IP
    try:
        ec2.revoke_security_group_ingress(
            GroupId=sg_id,
            IpPermissions=[{
                'IpProtocol': 'tcp',
                'FromPort': 22,
                'ToPort': 22,
                'IpRanges': [{'CidrIp': f'{remote_ip}/32'}]
            }]
        )
        print(f"Revoked SSH ingress from {remote_ip} on {sg_id}")
        return {'action': 'blocked', 'ip': remote_ip, 'sg': sg_id}
    except Exception as e:
        print(f"Could not revoke rule: {e}")
        return {'action': 'error', 'error': str(e)}
```
- [ ] Deploy.

**3. Create EventBridge rule**
- [ ] EventBridge → Rules → Create rule (default event bus, region: same as GuardDuty).
- [ ] Name: `guardduty-high-severity`.
- [ ] Event source: **AWS events**. Event pattern:
  ```json
  {
    "source": ["aws.guardduty"],
    "detail-type": ["GuardDuty Finding"],
    "detail": {
      "severity": [{ "numeric": [">=", 7] }]
    }
  }
  ```
- [ ] Target: Lambda function → `guardduty-remediation`.
- [ ] Create.

---

## Part D — Test the Pipeline End-to-End

**Using sample findings:**
- [ ] GuardDuty → Settings → Generate sample findings.
- [ ] Wait ~30 seconds.
- [ ] Lambda → `guardduty-remediation` → Monitor → CloudWatch Logs. You'll see log entries for each sample finding.
- [ ] Most sample findings won't have a real instance ID, so Lambda logs "no instance" and skips. That's expected for samples.

**Using a real finding (if you triggered one in Part B):**
- [ ] Real findings have actual instance IDs and remote IPs.
- [ ] Lambda executes, attempts to revoke the SSH inbound rule.
- [ ] Check the EC2's Security Group → Inbound rules → the specific /32 rule was removed.

**Manual test with a fake EventBridge event:**
- [ ] Lambda → Test → Configure test event:
  ```json
  {
    "source": "aws.guardduty",
    "detail-type": "GuardDuty Finding",
    "detail": {
      "type": "UnauthorizedAccess:EC2/SSHBruteForce",
      "severity": 7.5,
      "service": {
        "action": {
          "networkConnectionAction": {
            "remoteIpDetails": {
              "ipAddressV4": "198.51.100.1"
            }
          }
        }
      },
      "resource": {
        "instanceDetails": {
          "instanceId": "i-0abc1234def56789"
        }
      }
    }
  }
  ```
- [ ] Run the test → observe the Lambda log (will fail at `revoke_security_group_ingress` since the instance doesn't exist, but you'll see it got that far).

---

## Part E — GuardDuty Multi-Account (Exam Pattern)

In an AWS Organization, you designate one account as the **GuardDuty administrator account**. All member accounts automatically have GuardDuty enabled and their findings aggregated to the admin account.

- [ ] GuardDuty → Settings → Accounts → Configure accounts through AWS Organizations.
- [ ] Note: You can't opt out individual accounts once GuardDuty is org-enabled by the admin.
- [ ] Findings from all accounts appear in the admin account's GuardDuty console.

**Exam scenario:** "How do you centrally monitor security threats across 200 AWS accounts?"
→ GuardDuty with AWS Organizations delegation.

---

## Key Things to Observe

- GuardDuty is passive — it reads existing logs (VPC Flow Logs, CloudTrail, DNS). No agents to install.
- Enabling GuardDuty does NOT enable VPC Flow Logs or CloudTrail in your account — it uses them if they're already there, or reads via its own internal access.
- Finding severity: 1–3 Low, 4–6 Medium, 7–8.9 High, 9–10 Critical.
- EventBridge is the glue between GuardDuty findings and automated responses (Lambda, SNS, SSM automation).
- GuardDuty can be enabled per-account or org-wide. Org-wide is the exam-preferred pattern for centralised threat detection.
- GuardDuty does NOT block traffic — it only detects and reports. Blocking is your job (via Lambda + SG, NACL, or WAF).

---

## Clean Up

- GuardDuty → Settings → Disable GuardDuty (stops billing immediately).
- Delete EventBridge rule.
- Delete Lambda function.
- Delete IAM role for Lambda.
- Terminate any EC2s created for this lab.
