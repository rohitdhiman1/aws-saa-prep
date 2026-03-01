# Practice Questions: Phase 1 — IAM & VPC

*20 questions · Target score: ≥ 70% · Complete end of Week 2*

---

## Instructions

Answer each question before reading the answer. Write your choice, then compare.
Log wrong answers in `docs/bugbase.md` with the correct reasoning.

---

**Q1.**
A company's security policy requires that no IAM user should ever have direct access to S3. Instead, access must be granted only through specific applications. Which mechanism best enforces this?

A) Attach an S3 deny policy to all IAM users
B) Use an S3 bucket policy that denies all principals except specific IAM roles
C) Enable MFA on all IAM users
D) Use AWS Config to alert when users access S3

**Answer: B** — A bucket policy with `"Principal": {"AWS": "arn:aws:iam::ACCOUNT:role/AppRole"}` and a default Deny on `*` ensures only approved roles (used by applications) can access the bucket. User-level deny (A) is fragile and requires updating every user. MFA (C) doesn't restrict S3 access. Config (D) is detective, not preventive.

---

**Q2.**
A developer needs temporary credentials to access DynamoDB from an on-premises application. The application authenticates users with the company's SAML 2.0 identity provider. What should the solutions architect configure?

A) Create an IAM user for each on-premises user
B) Configure IAM Identity Center with an Active Directory connector
C) Use `AssumeRoleWithSAML` to exchange SAML assertions for temporary AWS credentials
D) Create long-lived access keys and store them in the application config file

**Answer: C** — `sts:AssumeRoleWithSAML` is the standard way to federate a SAML IdP into AWS. The SAML assertion from the IdP is exchanged for short-lived STS credentials scoped to an IAM role. Creating per-user IAM users (A) doesn't scale. IAM Identity Center (B) is for human SSO, not machine/API access. Long-lived keys (D) are a security anti-pattern.

---

**Q3.**
An AWS Organization has a root OU, a Production OU, and a Dev OU. An SCP attached to the root OU denies `ec2:TerminateInstances`. A separate SCP on the Production OU allows `ec2:TerminateInstances`. An IAM admin user in a Production account tries to terminate an instance. What happens?

A) The termination succeeds — the Production OU allow overrides the root deny
B) The termination fails — the root deny cannot be overridden by a child OU allow
C) The termination succeeds — IAM admin users are exempt from SCPs
D) It depends on whether the account has the default SCP attached

**Answer: B** — SCPs use the same evaluation logic as IAM: an explicit Deny always wins. The root OU deny cascades to all child OUs and accounts. A child OU SCP cannot override a parent Deny. SCPs apply to all principals including root users (except the management account itself). IAM admins are not exempt.

---

**Q4.**
A team accidentally left TCP port 22 open to `0.0.0.0/0` in an EC2 security group. They want to restrict SSH to only the corporate IP range `203.0.113.0/24`. What is the MOST secure way to fix this?

A) Add an inbound rule allowing `203.0.113.0/24` on port 22, then delete the `0.0.0.0/0` rule
B) Delete the security group and create a new one
C) Add a NACL deny rule for `0.0.0.0/0` on port 22
D) Enable VPC Flow Logs to detect unauthorized SSH attempts

**Answer: A** — Add the specific rule first, then remove the broad rule. Order matters operationally — removing the broad rule first would temporarily block all SSH including legitimate admin access. A new SG (B) requires reassigning it to all instances. NACL (C) would also block the corporate range unless a higher-priority allow rule exists. Flow Logs (D) are detective only.

---

**Q5.**
A solutions architect is designing access control for a multi-account environment. Developers in Account A need read-only access to S3 buckets in Account B. What is the MOST secure and scalable approach?

A) Create IAM users in Account B and share credentials with the Account A developers
B) In Account B, create an IAM role with S3 read permissions and a trust policy that allows Account A. In Account A, grant developers permission to `sts:AssumeRole` for that role.
C) Make the S3 buckets in Account B public
D) Use VPC Peering between Account A and Account B

**Answer: B** — Cross-account role assumption is the canonical pattern. The role in Account B trusts Account A, and Account A's IAM policies control which users/roles can assume it. Credentials are temporary (STS) and scoped to the role's permissions. Shared IAM user credentials (A) are a security anti-pattern. Public buckets (C) eliminate access control entirely. VPC Peering (D) is for network connectivity, not IAM authorization.

---

**Q6.**
A junior developer needs to deploy Lambda functions but must not be allowed to modify IAM roles or policies. Which IAM feature should the solutions architect use?

A) Service Control Policy
B) Permission boundary
C) Resource-based policy
D) Inline policy

**Answer: B** — A permission boundary defines the maximum permissions an IAM entity can have. Even if the developer's IAM policies allow IAM actions, the permission boundary can cap the effective permissions to exclude IAM write actions. SCPs (A) apply at the account level, not per-user. Resource-based policies (C) are on resources like S3, Lambda. Inline policies (D) are attached to identities and don't cap maximum permissions.

---

**Q7.**
A company's EC2 instances in a private subnet need to download OS updates from the internet. The VPC has an Internet Gateway (IGW) attached. What additional component is required?

A) Assign public IP addresses to the EC2 instances
B) Create a NAT Gateway in a public subnet and add a `0.0.0.0/0 → NAT GW` route to the private route table
C) Create a VPC endpoint for the package repository
D) Attach the IGW directly to the private subnet

**Answer: B** — Private subnet instances have no public IP, so they can't communicate directly with the IGW. A NAT Gateway in a public subnet provides outbound-only internet access for private instances. The private route table must have a default route pointing to the NAT GW. Public IPs (A) would make instances public, which contradicts the private subnet design. VPC endpoints (C) only work for AWS services. IGW attaches to the VPC, not individual subnets.

---

**Q8.**
An application in Subnet A (10.0.1.0/24) cannot reach an application in Subnet B (10.0.2.0/24) in the same VPC. Both subnets have the default local route (`10.0.0.0/16 → local`). Security groups allow all traffic between the two instances. What is the most likely cause?

A) VPC Peering is not configured
B) A NACL on one of the subnets is blocking traffic
C) The instances are in different Availability Zones
D) The IGW is not attached

**Answer: B** — Both subnets are in the same VPC so the local route handles routing — no peering needed. Security groups are stateful and both allow traffic. NACLs are stateless and evaluate inbound AND outbound rules separately. A NACL that blocks the return traffic (ephemeral ports 1024–65535) on the source subnet would silently drop the response. Different AZs (C) don't prevent communication — routing still works. IGW (D) is only needed for internet access.

---

**Q9.**
Which statement correctly describes the difference between Security Groups and Network ACLs?

A) Security Groups are stateless; NACLs are stateful
B) Security Groups can deny specific IPs; NACLs can only allow
C) Security Groups are stateful and operate at the instance level; NACLs are stateless and operate at the subnet level
D) NACLs are evaluated before Security Groups at the subnet boundary

**Answer: C** — Security Groups are stateful (return traffic is automatically allowed), apply to ENIs/instances, and only support allow rules. NACLs are stateless (return traffic must be explicitly allowed), apply to subnets, and support both allow and deny rules with numbered rule evaluation. A and B are reversed.

---

**Q10.**
A company is setting up VPC Peering between VPC-A (10.0.0.0/16) and VPC-B (172.16.0.0/16). After accepting the peering request, instances in VPC-A still cannot reach instances in VPC-B. What is the MOST likely missing step?

A) Enabling DNS resolution for the peering connection
B) Adding routes to both VPCs' route tables pointing to the peering connection
C) Attaching an IGW to both VPCs
D) Creating a VPN connection between the VPCs

**Answer: B** — VPC Peering does not automatically update route tables. You must manually add a route in VPC-A's route table (`172.16.0.0/16 → pcx-xxx`) and a route in VPC-B's route table (`10.0.0.0/16 → pcx-xxx`). Forgetting one side is the most common mistake. DNS resolution (A) helps with DNS but not basic IP connectivity. IGW (C) is for internet access. VPN (D) is an alternative to peering, not a supplement.

---

**Q11.**
An IAM policy contains both an explicit Allow for `s3:GetObject` on a bucket and an explicit Deny for `s3:GetObject` on the same bucket. What is the effective permission?

A) Allow — explicit allows take priority
B) Deny — explicit denies always take priority over allows
C) The last statement in the policy wins
D) It depends on whether it's an inline or managed policy

**Answer: B** — IAM policy evaluation logic: explicit Deny > explicit Allow > implicit Deny (default). An explicit Deny always overrides any Allow regardless of where the Allow comes from (same policy, other policies, resource policies). Policy type (inline vs managed) does not change this logic.

---

**Q12.**
A Lambda function needs to access a DynamoDB table. The function is not inside a VPC. What is the correct way to grant the function access to DynamoDB?

A) Attach an IAM user access key to the Lambda environment variables
B) Create an IAM role with DynamoDB permissions and assign it as the Lambda execution role
C) Add the Lambda function ARN to the DynamoDB resource policy
D) Configure a VPC endpoint for DynamoDB in the Lambda's account

**Answer: B** — Lambda execution roles are the standard pattern. AWS automatically provides temporary credentials to the Lambda function based on the execution role. Environment variable access keys (A) are a security anti-pattern — keys can be exposed in logs and don't rotate. DynamoDB doesn't have resource-based policies in the same way S3 does (C). VPC endpoints (D) are for network routing, not authorization, and irrelevant here since Lambda is not in a VPC.

---

**Q13.**
A company has a VPC with a public subnet and a private subnet. An EC2 instance in the private subnet needs to call the DynamoDB API without routing traffic through the internet. What should the solutions architect configure?

A) NAT Gateway in the public subnet
B) VPC Gateway Endpoint for DynamoDB
C) Interface Endpoint for DynamoDB
D) Direct Connect to the AWS backbone

**Answer: B** — DynamoDB supports Gateway endpoints, which are free and route traffic through AWS's internal network by adding a route to the subnet's route table. No NAT GW needed, no internet traffic. Gateway endpoints exist for S3 and DynamoDB only. Interface endpoints (C) also work for DynamoDB but cost money per hour + data. NAT GW (A) routes traffic to the public DynamoDB endpoint via internet, which works but costs money and uses internet bandwidth. Direct Connect (D) is for on-premises connectivity.

---

**Q14.**
An organization uses AWS Organizations. The security team wants to prevent all member accounts from creating IAM users — only IAM roles should be used. The management account must still be able to create IAM users for break-glass scenarios. Which solution achieves this with minimum effort?

A) Apply an SCP to the root OU denying `iam:CreateUser`
B) Apply an SCP to all member account OUs denying `iam:CreateUser`, leaving the management account unrestricted
C) Create an IAM policy in each member account denying `iam:CreateUser`
D) Use AWS Config rules to auto-delete IAM users when created

**Answer: B** — SCPs do not apply to the management account by design — it is always exempt from SCPs. So applying the deny SCP to the root OU (A) would still not affect the management account (which is correct), but attaching to member OUs is the more precise/intended way. B is more explicit about the design intent. Per-account IAM policies (C) require manual management in each account and can be removed by account admins. Config (D) is reactive.

---

**Q15.**
A solutions architect is reviewing a NACL rule set for a public subnet. The rules are:

- Rule 100: Allow inbound TCP 80 from 0.0.0.0/0
- Rule 200: Allow inbound TCP 443 from 0.0.0.0/0
- Rule `*`: Deny all

Clients report that HTTP requests succeed but responses are sometimes dropped. What is missing?

A) An allow rule for outbound TCP 80
B) An allow rule for outbound TCP 443
C) An outbound allow rule for ephemeral ports (1024–65535)
D) An inbound allow rule for TCP 22

**Answer: C** — NACLs are stateless. The inbound allow lets the request in, but the web server's response goes back to the client on a random ephemeral port (1024–65535). Without an outbound allow for those ports, the response is dropped by the Deny `*` rule. This is the classic NACL exam trap. A and B allow specific outbound ports (the wrong direction for responses). SSH (D) is unrelated.

---

**Q16.**
A company's compliance team requires that all API calls made with root account credentials are logged and the security team is alerted immediately. Which combination of services achieves this?

A) CloudTrail + CloudWatch Logs metric filter + SNS alarm
B) AWS Config + CloudWatch Events
C) GuardDuty + Security Hub
D) VPC Flow Logs + Lambda

**Answer: A** — CloudTrail records all API activity including root usage. A CloudWatch Logs metric filter on the CloudTrail log group can match root account events (`$.userIdentity.type = "Root"`), trigger a CloudWatch Alarm, which publishes to an SNS topic for immediate notification. This is the documented AWS pattern. Config (B) evaluates resource configurations, not API call patterns. GuardDuty (C) detects threats but with some latency. VPC Flow Logs (D) capture network traffic, not API calls.

---

**Q17.**
Which of the following statements about NAT Gateways is correct? (Select TWO)

A) A NAT Gateway must be placed in a private subnet
B) A NAT Gateway must be placed in a public subnet
C) A NAT Gateway requires an Elastic IP address
D) A NAT Gateway can be used by instances in the same subnet
E) A NAT Gateway is highly available within a single Availability Zone

**Answer: B, C** — NAT Gateways must sit in a public subnet (where the IGW route exists) so they can forward traffic to the internet. They require an Elastic IP as their public address. They are not used by instances in the same subnet (instances use the subnet's route table, not the NAT GW — NAT GW is for instances in other subnets). NAT GW is HA within its AZ only — for multi-AZ resilience, create one per AZ.

---

**Q18.**
A solutions architect needs to restrict an IAM role so that even if the role has `AdministratorAccess`, it cannot create or delete S3 buckets. The restriction must survive if someone updates the role's attached policies. Which solution achieves this?

A) Add an inline policy to the role denying `s3:CreateBucket` and `s3:DeleteBucket`
B) Use a permission boundary on the role that excludes `s3:CreateBucket` and `s3:DeleteBucket`
C) Apply an SCP to the account denying `s3:CreateBucket` and `s3:DeleteBucket`
D) Use a Service Control Policy on the OU

**Answer: B** — A permission boundary defines the maximum permissions an IAM entity can ever have, regardless of what policies are attached. Even if someone later attaches `AdministratorAccess`, the boundary caps the effective permissions to exclude the S3 bucket operations. An inline deny (A) works but can be removed by someone with IAM admin access. SCP (C/D) applies account-wide to all principals — if the requirement is role-specific, permission boundaries are the right tool.

---

**Q19.**
A company wants to allow its on-premises Active Directory users to log in to the AWS Management Console without creating IAM users. Users should inherit permissions based on their AD group membership. Which solution should the solutions architect implement?

A) Create IAM users mirroring each AD user, then manually sync group memberships
B) Set up AWS IAM Identity Center (SSO) with an AD Connector, map AD groups to permission sets
C) Use AWS Directory Service to create a new Managed Microsoft AD and have users log in with those credentials instead
D) Configure `AssumeRoleWithWebIdentity` with an OIDC provider

**Answer: B** — IAM Identity Center (formerly AWS SSO) with AD Connector federates existing on-premises AD into AWS. AD groups are mapped to permission sets (which map to IAM roles), and users log in via the SSO portal using their existing AD credentials. No IAM user creation needed. Option A doesn't scale. Option C creates a separate AD, it doesn't use the existing on-premises one. Option D is for OIDC (Google, Cognito), not SAML/AD.

---

**Q20.**
An EC2 instance in a public subnet has a public IP. It can reach the internet but cannot receive inbound connections on port 443. The security group has an inbound rule allowing TCP 443 from `0.0.0.0/0`. What is the most likely cause?

A) The route table is missing a route to the IGW
B) A NACL rule is blocking inbound TCP 443 before the security group is evaluated
C) The instance's operating system firewall (`iptables`) is blocking port 443
D) The public IP has not been associated with an Elastic IP

**Answer: B** — NACLs are evaluated at the subnet boundary before security groups. If a NACL has a numbered deny rule for port 443 with a lower rule number than the allow rule, traffic is dropped before reaching the security group. The route table (A) would affect all traffic, not just 443. OS firewall (C) is possible but the exam expects AWS-layer answers first. Not having an Elastic IP (D) doesn't affect connectivity for EC2 instances with auto-assigned public IPs.

---

## Score Log

| Date | Score | Notes |
|---|---|---|
| | /20 | |
