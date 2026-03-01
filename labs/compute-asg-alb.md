# Lab: Auto Scaling Group + Application Load Balancer

*Phase 2 · Week 3 · Concept: [concepts/compute.md](../concepts/compute.md)*

---

## Goal

Launch a scalable web tier: EC2 instances in an ASG behind an ALB, with target tracking scaling.

---

## Architecture

```
Internet
    │
   ALB (public subnets, 2 AZs)
    │
  Target Group
    │
  ASG (private subnets, 2 AZs)
  ├── EC2 (AZ-a)
  └── EC2 (AZ-b)
```

---

## Tasks

### 1. Prepare Launch Template
- [ ] Create a launch template: Amazon Linux 2023, t3.micro.
- [ ] User data script — install and start a simple HTTP server:
  ```bash
  #!/bin/bash
  yum install -y httpd
  systemctl start httpd
  echo "<h1>Hello from $(hostname -f)</h1>" > /var/www/html/index.html
  ```
- [ ] IAM instance profile: `SSMManagedInstanceCore` role (for SSM session access).
- [ ] Security group: allow HTTP (80) from ALB security group only.

### 2. Create ALB
- [ ] Create an internet-facing ALB in public subnets (2 AZs).
- [ ] Create a target group (HTTP, port 80, instance type).
- [ ] Add listener on port 80 forwarding to the target group.
- [ ] ALB security group: allow inbound HTTP from `0.0.0.0/0`.

### 3. Create Auto Scaling Group
- [ ] Create ASG using the launch template.
- [ ] VPC: private subnets in 2 AZs.
- [ ] Min: 1, Desired: 2, Max: 4.
- [ ] Attach to the ALB target group.
- [ ] Health check: ELB type (ALB health check).

### 4. Add Target Tracking Scaling
- [ ] Add a Target Tracking policy: metric = `ALBRequestCountPerTarget`, target = 100.
- [ ] (Alternative) CPU utilisation target tracking at 50%.

### 5. Test
- [ ] Copy ALB DNS name; open in browser — see instance hostname.
- [ ] Refresh multiple times — observe different hostnames (load balancing).
- [ ] Generate load (CloudShell `ab` or `hey`) to trigger scale-out.
- [ ] Watch ASG activity in console — new instances launched.
- [ ] Stop load — observe scale-in after cooldown.

---

## Key Things to Observe

- ALB distributes requests across healthy instances.
- ASG replaces an unhealthy instance automatically.
- Target tracking adds instances before CPU maxes out (proactive).
- Cooldown prevents thrashing.

---

## Clean Up

- Delete ASG (set desired to 0 first or delete directly).
- Delete ALB + target group.
- Delete launch template.
- Delete security groups.
