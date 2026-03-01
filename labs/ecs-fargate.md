# Lab: ECS Fargate — Containers, Task Roles & Service Scaling

*Phase 2 · Compute · Concept: [concepts/compute.md](../concepts/compute.md)*

---

## Goal

Deploy a containerised application on ECS Fargate. Understand the two IAM roles (execution role vs task role) by deliberately breaking each and observing the error. Scale the service and put it behind an ALB.

---

## Architecture

```
Internet → ALB → ECS Service (2 Fargate tasks)
                      │
                  Task execution role → pulls image from ECR, writes logs to CloudWatch
                  Task role → app code reads from S3
```

---

## Part A — Create the IAM Roles

**Execution Role** (used by ECS agent, not your app code)
- [ ] IAM → Roles → Create → Trusted entity: `ecs-tasks.amazonaws.com`.
- [ ] Attach: `AmazonECSTaskExecutionRolePolicy` (allows ECR pull + CloudWatch Logs).
- [ ] Name: `ecsTaskExecutionRole`.

**Task Role** (used by your running container code)
- [ ] IAM → Roles → Create → Trusted entity: `ecs-tasks.amazonaws.com`.
- [ ] Attach: `AmazonS3ReadOnlyAccess`.
- [ ] Name: `ecsTaskRole-S3Read`.

---

## Part B — Create an S3 Bucket with a Test File

- [ ] Create bucket: `ecs-task-role-test-<random>`. Upload a file `hello.txt` with content `task role works`.

---

## Part C — Create the ECS Cluster and Task Definition

**1. Create ECS Cluster**
- [ ] ECS → Clusters → Create cluster. Infrastructure: **Fargate only**. Name: `lab-cluster`.

**2. Create Task Definition**
- [ ] ECS → Task definitions → Create → **Fargate**.
- [ ] Task definition family: `lab-task`.
- [ ] **Task execution role:** `ecsTaskExecutionRole` ← ECS infrastructure role.
- [ ] **Task role:** `ecsTaskRole-S3Read` ← your application's role.
- [ ] CPU: 0.25 vCPU. Memory: 0.5 GB.
- [ ] Container: name `web`, image `public.ecr.aws/nginx/nginx:latest` (public nginx).
- [ ] Port mappings: 80.
- [ ] Add environment variable: `S3_BUCKET=ecs-task-role-test-<random>`.
- [ ] Create.

---

## Part D — Run a Task and Verify Task Role Works

**1. Run a standalone task (not a service yet)**
- [ ] ECS → Clusters → `lab-cluster` → Tasks → Run new task.
- [ ] Launch type: Fargate. Task definition: `lab-task`. VPC: your VPC. Subnet: public. Security group: allow HTTP 80.
- [ ] Enable "Auto-assign public IP".
- [ ] Run task. Wait for it to reach RUNNING.

**2. Exec into the container to test the task role**
- [ ] Enable ECS Exec on the task (if not already):
  ```bash
  aws ecs execute-command \
    --cluster lab-cluster \
    --task <task-id> \
    --container web \
    --interactive \
    --command "/bin/sh"
  ```
- [ ] Inside the container:
  ```bash
  # List the S3 bucket using the task role credentials (auto-injected by ECS)
  aws s3 ls s3://ecs-task-role-test-<random>
  ```
  → Should return the file listing. Task role is working.

**3. Break the task role — remove S3 permission**
- [ ] In IAM, detach `AmazonS3ReadOnlyAccess` from `ecsTaskRole-S3Read`.
- [ ] Stop the task. Run a new task (fresh credentials).
- [ ] Exec in and try `aws s3 ls` again → `Access Denied`. App code can't access S3.
- [ ] Reattach the policy.

---

## Part E — Understand the Execution Role vs Task Role Failure Modes

| Break this | Error you'll see |
|---|---|
| Remove execution role | Task fails to pull image from ECR / fails to create log group |
| Remove task role | Container starts fine but app code fails when calling AWS APIs |

To see the execution role error:
- [ ] Create a task definition with **no execution role** set.
- [ ] Try to run a task → it fails at the `PROVISIONING` or `PULLING` stage with an error about credentials for ECR or logs.

---

## Part F — Create an ECS Service Behind an ALB

**1. Create a target group**
- [ ] Target type: **IP** (Fargate uses IP targets, not instance targets). Protocol: HTTP. Port: 80.
- [ ] Health check: path `/`.

**2. Create an ALB**
- [ ] Internet-facing. Public subnets. SG: allow HTTP 80 from `0.0.0.0/0`.
- [ ] Listener HTTP :80 → forward to the target group.

**3. Create the ECS Service**
- [ ] ECS → `lab-cluster` → Services → Create.
- [ ] Launch type: Fargate. Task definition: `lab-task`. Service name: `lab-service`.
- [ ] Desired tasks: **2**.
- [ ] Networking: private subnets (or public with public IP). SG: allow HTTP 80 from ALB SG.
- [ ] Load balancing: ALB → select your ALB and target group.
- [ ] Create.

**4. Test**
- [ ] ALB DNS → browser → nginx default page. Refresh several times — two different IPs in CloudWatch logs (2 tasks running).

**5. Scale**
- [ ] Service → Update → Desired tasks: 4. Observe 4 tasks launch in parallel.
- [ ] Scale back to 2 → ECS drains 2 tasks (ALB stops sending traffic first, then tasks stop).

---

## Key Things to Observe

- **Task execution role** = ECS infrastructure. Needs `AmazonECSTaskExecutionRolePolicy`. Without it, the task can't even start.
- **Task role** = your app code. Grants AWS API access from inside the container. Without it, your app gets `AccessDenied` on AWS calls.
- Fargate target type in ALB must be **IP** (not Instance). Each Fargate task gets its own ENI with a private IP.
- Scaling is fast — Fargate provisions new containers in ~30s, no EC2 warm-up.
- ECS Exec lets you shell into a running Fargate container without SSH.

---

## Clean Up

- Service → Update desired count to 0 → then delete service.
- Delete ALB and target group.
- Delete task definition (deregister).
- Delete ECS cluster.
- Delete S3 bucket.
- Delete both IAM roles.
