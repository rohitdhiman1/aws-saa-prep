# VPC Custom Build Lab

**Objective:** Build a highly available, custom VPC from scratch entirely through the AWS Console to understand the components and routing logic natively.

## Lab Architecture Requirements
- **1 x Custom VPC** (e.g. `10.0.0.0/16`)
- **2 x Availability Zones** (e.g., `us-east-1a`, `us-east-1b`)
- **2 x Public Subnets** (1 per AZ)
- **2 x Private Subnets** (1 per AZ)
- **1 x Internet Gateway** (IGW) attached to the VPC
- **1 x Custom Route Table** for public subnets (routing `0.0.0.0/0` -> IGW)
- **1 x NAT Gateway** located in a Public Subnet
- **1 x Custom Route Table** for private subnets (routing `0.0.0.0/0` -> NAT Gateway)

---

## Step-by-Step Guide

### 1. Create the Custom VPC
- Go to **VPC Console** -> **Your VPCs** -> **Create VPC**.
- *Select "VPC only"* (do not use the "VPC and more" wizard so you can build the pieces manually).
- **Name tag:** `Custom-VPC`
- **IPv4 CIDR block:** `10.0.0.0/16`
- **Tenancy:** Default

### 2. Create the Subnets
Navigate to **Subnets** -> **Create subnet**.

1. **Public Subnet A**
   - **VPC ID:** `Custom-VPC`
   - **Subnet name:** `Public-Subnet-A`
   - **Availability Zone:** Select your first AZ (e.g., `us-east-1a`)
   - **IPv4 CIDR block:** `10.0.1.0/24`

2. **Public Subnet B**
   - (Select "Add new subnet")
   - **Subnet name:** `Public-Subnet-B`
   - **Availability Zone:** Select your second AZ (e.g., `us-east-1b`)
   - **IPv4 CIDR block:** `10.0.2.0/24`

3. **Private Subnet A**
   - (Select "Add new subnet")
   - **Subnet name:** `Private-Subnet-A`
   - **Availability Zone:** Select your first AZ (e.g., `us-east-1a`)
   - **IPv4 CIDR block:** `10.0.10.0/24`

4. **Private Subnet B**
   - (Select "Add new subnet")
   - **Subnet name:** `Private-Subnet-B`
   - **Availability Zone:** Select your second AZ (e.g., `us-east-1b`)
   - **IPv4 CIDR block:** `10.0.11.0/24`

*Note: For the public subnets, select them from the list, click Actions -> Edit subnet settings -> Enable "Auto-assign public IPv4 address".*

### 3. Attach an Internet Gateway
- Navigate to **Internet Gateways** -> **Create internet gateway**.
- **Name tag:** `Custom-IGW`
- Once created, select it -> **Actions** -> **Attach to VPC**. Select `Custom-VPC`.

### 4. Configure Public Route Table
- Navigate to **Route tables**. Note the "Main" route table already exists for your VPC. Leave it alone.
- Click **Create route table**.
- **Name:** `Public-RT`
- **VPC:** `Custom-VPC`
- Once created, click the `Public-RT` -> **Routes** tab -> **Edit routes**.
   - Add route: Destination `0.0.0.0/0`, Target `Custom-IGW`. Safe.
- Go to the **Subnet associations** tab -> **Edit subnet associations**.
   - Check `Public-Subnet-A` and `Public-Subnet-B`. Save.

*Both public subnets can now route out to the internet.*

### 5. Create a NAT Gateway
- Navigate to **NAT gateways** -> **Create NAT gateway**.
- **Name:** `Custom-NAT`
- **Subnet:** Select `Public-Subnet-A` (NAT GWs must live in a public subnet to reach the internet).
- **Connectivity type:** Public.
- Click **Allocate Elastic IP**.
- Create NAT gateway. (It will take a few minutes to transition from Pending to Available).

### 6. Configure Private Route Table
- Navigate to **Route tables**. You can use the "Main" route table for private subnets, or create a dedicated private one to be safe. Let's create a dedicated one.
- Click **Create route table**.
- **Name:** `Private-RT`
- **VPC:** `Custom-VPC`
- Once created, click the `Private-RT` -> **Routes** tab -> **Edit routes**.
   - Add route: Destination `0.0.0.0/0`, Target `nat-*******` (`Custom-NAT`). Save.
- Go to the **Subnet associations** tab -> **Edit subnet associations**.
   - Check `Private-Subnet-A` and `Private-Subnet-B`. Save.

*Both private subnets now route their internet bound traffic to the NAT gateway sitting in the public subnet.*

---

## Validation / Testing Commands (Optional EC2 Deploy)

To prove this works, you can launch two small EC2 instances (e.g., Amazon Linux 2023, `t2.micro`):

1. **Test Instance 1 (Public Subnet):** Ensure it receives a public IP. SSH/Session Manager into it, run `ping google.com`. It should work.
2. **Test Instance 2 (Private Subnet):** Ensure it has NO public IP. Connect to it via Session Manager (ensure VPC endpoints are set up) OR SSH from Instance 1. Once connected, run `ping google.com`. It should successfully route through the NAT Gateway.

---

## Cleanup!
*Note: If you are using the **A Cloud Guru (ACG) 4-hour AWS Sandbox**, the environment will auto-destruct when the timer expires. However, if you are doing this in your personal account, perform the steps below immediately to avoid billing shocks (especially NAT Gateways!)*

1. Terminate any EC2 instances.
2. Delete the **NAT Gateway** (NAT GWs bill by the hour approx $0.045/hr).
3. Release the **Elastic IP** associated with the NAT GW.
4. Detach and delete the **Internet Gateway**.
5. Delete the **VPC** (this will clean up subnets and route tables).
