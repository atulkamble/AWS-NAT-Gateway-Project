A **complete NAT Gateway project on AWS** using AWS CLI commands and step-by-step setup instructions. This includes VPC, subnets, Internet Gateway, route tables, NAT Gateway, and EC2 instances (public & private) to demonstrate outbound internet access from private instances via NAT Gateway.

---

### 🧾 **Project Overview**

* **Goal**: Allow private EC2 instance to access the internet (e.g., yum, apt updates) via NAT Gateway.
* **Use Case**: Secure environment with private workloads needing outbound access.

---

### 🧱 **Architecture Diagram**

```
                      +------------------------+
                      |    AWS Internet        |
                      +-----------+------------+
                                  |
                        +---------v---------+
                        |  Internet Gateway  |
                        +---------+---------+
                                  |
              +------------------+-------------------+
              |                                      |
         +----v----+                           +-----v-----+
         | Subnet  |                           | Subnet    |
         | Public  |                           | Private   |
         +----+----+                           +-----+-----+
              |                                      |
         +----v----+                           +-----v-----+
         | EC2-Pub |                           | EC2-Priv  |
         +---------+                           +-----------+
              |
        +-----v-----+
        | NAT GW    |
        +-----------+
```

---

## 🔧 Step-by-Step Setup with AWS CLI

---

### 1️⃣ Create VPC

```bash
aws ec2 create-vpc --cidr-block 10.0.0.0/16 --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=nat-vpc}]'
```

---

### 2️⃣ Create Subnets

```bash
# Public Subnet
aws ec2 create-subnet --vpc-id <vpc-id> --cidr-block 10.0.1.0/24 --availability-zone ap-south-1a --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=public-subnet}]'

# Private Subnet
aws ec2 create-subnet --vpc-id <vpc-id> --cidr-block 10.0.2.0/24 --availability-zone ap-south-1a --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=private-subnet}]'
```

---

### 3️⃣ Enable Auto-assign Public IP for Public Subnet

```bash
aws ec2 modify-subnet-attribute --subnet-id <public-subnet-id> --map-public-ip-on-launch
```

---

### 4️⃣ Create and Attach Internet Gateway

```bash
aws ec2 create-internet-gateway --tag-specifications 'ResourceType=internet-gateway,Tags=[{Key=Name,Value=nat-igw}]'

aws ec2 attach-internet-gateway --internet-gateway-id <igw-id> --vpc-id <vpc-id>
```

---

### 5️⃣ Create Elastic IP for NAT Gateway

```bash
aws ec2 allocate-address --domain vpc
```

---

### 6️⃣ Create NAT Gateway

```bash
aws ec2 create-nat-gateway --subnet-id <public-subnet-id> --allocation-id <eip-alloc-id> --tag-specifications 'ResourceType=natgateway,Tags=[{Key=Name,Value=nat-gateway}]'
```

---

### 7️⃣ Create Route Tables

```bash
# Public Route Table
aws ec2 create-route-table --vpc-id <vpc-id> --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=public-rt}]'

# Route to Internet
aws ec2 create-route --route-table-id <public-rt-id> --destination-cidr-block 0.0.0.0/0 --gateway-id <igw-id>

# Associate Public Subnet
aws ec2 associate-route-table --subnet-id <public-subnet-id> --route-table-id <public-rt-id>
```

```bash
# Private Route Table
aws ec2 create-route-table --vpc-id <vpc-id> --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=private-rt}]'

# Route via NAT
aws ec2 create-route --route-table-id <private-rt-id> --destination-cidr-block 0.0.0.0/0 --nat-gateway-id <nat-gateway-id>

# Associate Private Subnet
aws ec2 associate-route-table --subnet-id <private-subnet-id> --route-table-id <private-rt-id>
```

---

### 8️⃣ Launch EC2 Instances

```bash
# Key pair
aws ec2 create-key-pair --key-name mykey --query 'KeyMaterial' --output text > mykey.pem
chmod 400 mykey.pem

# Security Group
aws ec2 create-security-group --group-name nat-sg --description "Allow SSH and HTTP" --vpc-id <vpc-id>

aws ec2 authorize-security-group-ingress --group-id <sg-id> \
--protocol tcp --port 22 --cidr 0.0.0.0/0

# EC2 Public
aws ec2 run-instances --image-id ami-0f58b397bc5c1f2e8 \
--count 1 --instance-type t2.micro \
--key-name mykey --subnet-id <public-subnet-id> \
--security-group-ids <sg-id> \
--associate-public-ip-address \
--tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=ec2-public}]'

# EC2 Private
aws ec2 run-instances --image-id ami-0f58b397bc5c1f2e8 \
--count 1 --instance-type t2.micro \
--key-name mykey --subnet-id <private-subnet-id> \
--security-group-ids <sg-id> \
--tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=ec2-private}]'
```

---

### 🔎 Test the NAT Gateway

1. **Connect to EC2 Public** using SSH.
2. From there, SSH into **EC2 Private** (use private IP).
3. On private instance, try:

   ```bash
   curl https://www.google.com
   sudo yum update -y
   ```
   → This confirms internet access through NAT Gateway.
---
