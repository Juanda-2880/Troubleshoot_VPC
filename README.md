# Module 7 – Activity 7: Troubleshoot a VPC

##  Overview

This activity provides hands-on practice using **VPC Flow Logs** and troubleshooting **Amazon VPC configurations**. The environment consists of two VPCs:

- **VPC1** — Contains the Mom & Pop Café web server in a public subnet (has intentional misconfigurations to resolve).
- **VPC2** — Contains the CLI Host instance used to run AWS CLI commands (no issues).

> VPC1 and VPC2 are **not peered**. The CLI Host connects to VPC1 resources via the internet, just like an external machine would.

---

##  Objectives

After completing this activity, you will be able to:

- Identify VPC configuration issues.
- Troubleshoot VPC configuration issues.
- Enable VPC Flow Logs.
- Analyze flow logs using `grep`.

---

##  Prerequisites

- AWS account with lab credentials (AccessKey / SecretKey).
- SSH client (PuTTY for Windows, Terminal for macOS/Linux).
- Downloaded key pair file (`labsuser.pem` or `labsuser.ppk`).

---

##  Tasks

---

### Task 1.2 – Configure the AWS CLI on the CLI Host

Connected to the CLI Host EC2 instance via SSH and configured AWS CLI credentials using `aws configure`.

**Commands used:**
```bash
# Discover the current region
curl http://169.254.169.254/latest/dynamic/instance-identity/document | grep region

# Configure AWS CLI
aws configure
```

**IMAGES – Task 1 and 1.2:**

<img width="672" height="102" alt="task1" src="https://github.com/user-attachments/assets/337c4a2f-1354-428c-bc32-d6f7b6eefe2c" />

<img width="1600" height="802" alt="task1-1" src="https://github.com/user-attachments/assets/39eb9179-6530-40d8-9a3a-2d026ea7d20a" />

<img width="1056" height="552" alt="task1-2" src="https://github.com/user-attachments/assets/2f0ba730-db86-409a-b7f7-f15c259f82f3" />

<img width="1416" height="685" alt="TASK12" src="https://github.com/user-attachments/assets/27e85cd7-01ed-42be-9d47-4fd9d943dd64" />

<img width="916" height="223" alt="TASK12-1" src="https://github.com/user-attachments/assets/954abd11-0735-4b38-a563-6afefaa5b3f7" />

---

### Task 2 – Enable VPC Flow Logs

Created an S3 bucket to store VPC Flow Logs, then enabled flow logging on VPC1 to capture all IP traffic.

**Commands used:**
```bash
# Create S3 bucket for flow logs
aws s3api create-bucket \
  --bucket flowlog#### \
  --region <region> \
  --create-bucket-configuration LocationConstraint=<region>

# Get VPC1 ID
aws ec2 describe-vpcs \
  --query 'Vpcs[*].[VpcId,Tags[?Key==`Name`].Value,CidrBlock]' \
  --filters "Name=tag:Name,Values='VPC1'"

# Enable VPC Flow Logs
aws ec2 create-flow-logs \
  --resource-type VPC \
  --resource-ids <vpc-id> \
  --traffic-type ALL \
  --log-destination-type s3 \
  --log-destination arn:aws:s3:::<flowlog####>

# Confirm flow log creation
aws ec2 describe-flow-logs
```

**IMAGES – Task 2:**

<img width="557" height="63" alt="TASK2" src="https://github.com/user-attachments/assets/b5cfb296-e91e-4de2-94e2-43880d666fc7" />

<img width="1153" height="137" alt="TASK2-1" src="https://github.com/user-attachments/assets/751e7a52-6533-4258-a397-ff00f5ad6122" />

<img width="1600" height="261" alt="TASK2-2" src="https://github.com/user-attachments/assets/4d2c020d-0e4f-47e0-9f73-b5086d7c56de" />


<img width="1600" height="252" alt="TASK2-3" src="https://github.com/user-attachments/assets/ad0e491b-40b6-4d7d-8bd0-8298044e4faa" />

<img width="1600" height="447" alt="TASK2-4" src="https://github.com/user-attachments/assets/96057ff1-34d3-4c06-935d-127c69871a00" />

---

### Task 3 – Analyze and Troubleshoot Access to Resources

Attempted to access the web server via browser (HTTP) and SSH. Both attempts failed with **ERR_CONNECTION_TIMED_OUT** and **Connection timed out**, respectively.

**Commands used:**
```bash
# Describe the web server instance
aws ec2 describe-instances \
  --filter "Name=ip-address,Values='<WebServerIP>'" \
  --query 'Reservations[*].Instances[*].[State,PrivateIpAddress,InstanceId,SecurityGroups,SubnetId,KeyName]'
```


**IMAGES – Task 3:**

<img width="1600" height="829" alt="TASK3" src="https://github.com/user-attachments/assets/f94fed15-f033-47d5-9ec4-4b9d4eb473fb" />


<img width="1600" height="538" alt="TASK3-1" src="https://github.com/user-attachments/assets/b64f64f4-c49b-43d3-a76c-508895edb49f" />

<img width="571" height="125" alt="TASK3-2" src="https://github.com/user-attachments/assets/21eef56b-544b-4b7b-8a9c-d69fd2a5c575" />


<img width="647" height="77" alt="TASK3-3" src="https://github.com/user-attachments/assets/c72f88bf-d180-4a32-b46e-cee7e426e240" />

<img width="654" height="104" alt="TASK3-4" src="https://github.com/user-attachments/assets/45b2b9ea-d8ea-4f59-8802-0d0e858a5275" />

<img width="657" height="94" alt="TASK3-5" src="https://github.com/user-attachments/assets/7b263696-4714-491e-bc37-4857347ae555" />

---

###  Troubleshooting Challenge #1 – Route Table Issue

**Symptom:** The website was unreachable and SSH timed out even though the instance was running.

**Investigation:**
```bash
# Install nmap and check open ports
sudo yum install -y nmap
nmap <WebServerIP>

# Check security group rules
aws ec2 describe-security-groups --group-ids <WebServerSgId>

# Check route table for the public subnet
aws ec2 describe-route-tables \
  --filter "Name=association.subnet-id,Values='<VPC1PubSubnetID>'"
```

**Root Cause:** The public subnet's route table was **missing a route to the Internet Gateway (IGW)**. Without this route, traffic from/to the internet could not flow.

**Fix:**
```bash
# Add the missing default route to the Internet Gateway
aws ec2 create-route \
  --route-table-id <RouteTableId> \
  --destination-cidr-block 0.0.0.0/0 \
  --gateway-id <InternetGatewayId>
```

**Result:** After adding the route, the website loaded successfully at `http://<WebServerIP>/mompopcafe/`.

**IMAGES – Troubleshooting Challenge #1:**

<img width="1512" height="573" alt="TR1" src="https://github.com/user-attachments/assets/e8b231e9-70c6-4dfa-a042-178abc6a811b" />

<img width="682" height="107" alt="TR1-1" src="https://github.com/user-attachments/assets/8eb4ebde-db10-439c-bd6a-09121f703ba3" />

<img width="1324" height="500" alt="TR1-2" src="https://github.com/user-attachments/assets/5f96a243-a5e2-4f4e-a0b1-d567ce400d19" />


<img width="1348" height="501" alt="TR1-3" src="https://github.com/user-attachments/assets/0637a9ec-a4a0-4703-a36c-7091c4c96182" />


<img width="1271" height="95" alt="TR1-4" src="https://github.com/user-attachments/assets/614a5ea8-7df0-441e-a70b-bb29197ffb8a" />


<img width="1600" height="838" alt="TR1-5" src="https://github.com/user-attachments/assets/d763e37e-2d3a-47e7-b83f-ef0df43f8fc9" />


<img width="1600" height="844" alt="TR1-6" src="https://github.com/user-attachments/assets/485190e9-24aa-4d4c-8c05-01ad52cd1ecf" />

---

###  Troubleshooting Challenge #2 – Network ACL Issue

**Symptom:** The website was now accessible, but SSH to the web server still timed out.

**Investigation:**
```bash
# Check the Network ACL associated with the public subnet
aws ec2 describe-network-acls \
  --filter "Name=association.subnet-id,Values='<VPC1PublicSubnetID>'" \
  --query 'NetworkAcls[*].[NetworkAclId,Entries]'
```

**Root Cause:** A **Network ACL (NACL) rule** was blocking inbound/outbound traffic on port 22 (SSH). Unlike security groups, NACLs are stateless and require explicit rules for both inbound and outbound traffic.

**Fix:**
```bash
# Delete the offending NACL entry that was blocking SSH
aws ec2 delete-network-acl-entry \
  --network-acl-id <NetworkAclId> \
  --rule-number <RuleNumber> \
  --ingress
```

**Result:** After removing the restrictive NACL rule, SSH connected successfully. Running `hostname` confirmed the connection to `web-server`.

**IMAGES – Troubleshooting Challenge #2:**

<img width="1503" height="307" alt="TR2" src="https://github.com/user-attachments/assets/a8934976-13f7-439c-af80-ab7fdd96b5a4" />

<img width="1501" height="549" alt="TR2-1" src="https://github.com/user-attachments/assets/1a113fc3-bd03-4560-bc98-efcf74c07abf" />

<img width="1516" height="590" alt="TR2-2" src="https://github.com/user-attachments/assets/1bb54250-0c11-488d-b3a5-66aeecc7b5d7" />

<img width="788" height="313" alt="TR2-3" src="https://github.com/user-attachments/assets/b29ebd0e-ef72-4b68-a24c-7517511c5d99" />

---

### Task 4 – Analyze Flow Logs

Downloaded the VPC Flow Logs from S3 to the CLI Host, extracted them, and used `grep` to identify and analyze the failed connection attempts.

**Commands used:**
```bash
# Create directory and download flow logs
mkdir flowlogs && cd flowlogs
aws s3 ls
aws s3 cp s3://<flowlog####>/ . --recursive

# Extract compressed logs
gunzip *.gz

# View log structure
head <log-file-name>

# Find all REJECT entries
grep -rn REJECT .

# Count total REJECT records
grep -rn REJECT . | wc -l

# Filter by SSH port (22) and REJECT status
grep -rn ' 22 ' . | grep REJECT

# Refine by source IP address (your local machine)
grep -rn ' 22 ' . | grep REJECT | grep <your-ip-address>

# Confirm network interface matches the web server
aws ec2 describe-network-interfaces \
  --filters "Name=association.public-ip,Values='<WebServerIP>'" \
  --query 'NetworkInterfaces[*].[NetworkInterfaceId,Association.PublicIp]'

# Convert Unix timestamp to human-readable format
date -d @<timestamp>
```

**Key findings:**
- Each failed SSH attempt generated a `REJECT` entry in the flow logs with port `22`.
- The ENI ID in the logs matched the network interface of the web server instance.
- Timestamps confirmed the events occurred during the troubleshooting session.

**IMAGES – Task 4:**

<img width="1519" height="569" alt="TASK4" src="https://github.com/user-attachments/assets/ba975c10-a807-4110-a7d9-353565bc3639" />

<img width="1395" height="301" alt="TASK4-1" src="https://github.com/user-attachments/assets/9713b274-f949-4dcf-9fe9-6b22f356cc48" />

<img width="932" height="106" alt="TASK4-2" src="https://github.com/user-attachments/assets/e6e9fdfc-5fd2-407c-a049-7be1e961193c" />

<img width="1526" height="563" alt="TASK4-3" src="https://github.com/user-attachments/assets/abe90a33-b661-4b4a-9a51-c3b7e0b3ebee" />

<img width="1527" height="543" alt="TASK4-4" src="https://github.com/user-attachments/assets/f932a37c-32fc-484e-ab9b-c6cb7b419441" />

<img width="1600" height="809" alt="TASK4-5" src="https://github.com/user-attachments/assets/f57391af-50ff-42a8-93ea-06a51c80909c" />

<img width="1600" height="139" alt="TASK4-6" src="https://github.com/user-attachments/assets/56d71099-862d-4fc5-b69c-7b3868668d0e" />

<img width="1506" height="284" alt="TASK4-7" src="https://github.com/user-attachments/assets/b27bee85-658e-42b9-b909-9259c05fbc0f" />

<img width="1520" height="166" alt="TASK4-8" src="https://github.com/user-attachments/assets/5a8592ed-0810-4795-984f-4a9323463140" />

<img width="1600" height="206" alt="TASK4-9" src="https://github.com/user-attachments/assets/f378b48f-663e-495f-9347-4cc0a02757db" />

---

##  Summary of Issues Found and Fixed

| # | Issue | Root Cause | Fix Applied |
|---|-------|-----------|-------------|
| 1 | Website unreachable (HTTP & SSH timeout) | Missing route to Internet Gateway in the public route table | Added `0.0.0.0/0` route pointing to the IGW |
| 2 | SSH still blocked after fix #1 | Network ACL rule explicitly denying port 22 traffic | Deleted the restrictive NACL inbound rule |

---

##  Key Concepts Reviewed

- **VPC Flow Logs** — Capture IP traffic metadata for network interfaces within a VPC and store them in S3 or CloudWatch.
- **Route Tables** — Public subnets require a route `0.0.0.0/0 → IGW` to communicate with the internet.
- **Security Groups** — Stateful; only inbound rules need to allow SSH (port 22) and HTTP (port 80).
- **Network ACLs** — Stateless; require explicit allow rules for both inbound **and** outbound traffic. They are evaluated in rule-number order.
- **`grep`** — Powerful CLI tool used to filter and search log files efficiently.


---

*Activity completed as part of the AWS Cloud Foundations course – Module 7.*
