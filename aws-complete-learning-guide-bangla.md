# AWS Complete Learning Guide - সম্পূর্ণ বাংলা গাইড

## 📋 সূচিপত্র
- [AWS কি এবং কেন শিখবেন](#aws-কি-এবং-কেন-শিখবেন)
- [IAM (Identity and Access Management)](#iam-identity-and-access-management)
- [EC2 (Elastic Compute Cloud)](#ec2-elastic-compute-cloud)
- [Security Groups](#security-groups)
- [VPC (Virtual Private Cloud)](#vpc-virtual-private-cloud)
- [Load Balancing (ELB)](#load-balancing-elb)
- [Advanced Services](#advanced-services)
- [DevOps Integration](#devops-integration)
- [Hands-on Projects](#hands-on-projects)

---

## AWS কি এবং কেন শিখবেন

### 🌐 **Amazon Web Services (AWS) কি?**

AWS হলো Amazon এর **cloud computing platform** যা বিভিন্ন ধরনের IT services প্রদান করে।

### 🎯 **কেন AWS শিখবেন?**

- ✅ **Market Leader:** 32% cloud market share
- ✅ **Job Opportunities:** High demand for AWS skills
- ✅ **Cost Effective:** Pay-as-you-use model
- ✅ **Scalability:** Auto scaling capabilities
- ✅ **Global Reach:** 200+ services worldwide
- ✅ **Security:** Enterprise-grade security

### 💰 **AWS Pricing Model:**
```
Free Tier: 12 months free usage
Pay-as-you-go: শুধু যা ব্যবহার করবেন তার জন্য payment
Reserved Instances: Long-term commitment এ discount
Spot Instances: Unused capacity এ 90% discount
```

### 🏗️ **AWS Core Concepts:**
```
Region: Geographic area (us-east-1, ap-south-1)
Availability Zone: Data centers within region
Edge Locations: CDN endpoints
Services: 200+ different services
```

---

## IAM (Identity and Access Management)

### 🔐 **IAM কি?**

IAM হলো AWS এর **security service** যা **"কে কি করতে পারবে"** সেটা control করে।

### 🎯 **IAM Components:**

**1. Users:**
```json
{
  "UserName": "john-developer",
  "Policies": ["DeveloperAccess"],
  "Groups": ["Developers"],
  "AccessKeys": {
    "AccessKeyId": "AKIA...",
    "SecretAccessKey": "wJal..."
  }
}
```

**2. Groups:**
```json
{
  "GroupName": "Developers",
  "Users": ["john", "jane", "bob"],
  "Policies": [
    "AmazonEC2FullAccess",
    "AmazonS3ReadOnlyAccess"
  ]
}
```

**3. Roles:**
```json
{
  "RoleName": "EC2-S3-Access-Role",
  "AssumeRolePolicyDocument": {
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"Service": "ec2.amazonaws.com"},
      "Action": "sts:AssumeRole"
    }]
  }
}
```

**4. Policies:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::my-bucket/*"
    },
    {
      "Effect": "Deny",
      "Action": "s3:DeleteObject",
      "Resource": "*"
    }
  ]
}
```

### 🛡️ **IAM Best Practices:**

**1. Root User Security:**
```bash
# Root user শুধু account setup এর জন্য ব্যবহার করুন
# MFA enable করুন
# Access keys তৈরি করবেন না
# Billing alerts setup করুন
```

**2. Least Privilege Principle:**
```json
// ❌ Bad: Too much permission
{
  "Effect": "Allow",
  "Action": "*",
  "Resource": "*"
}

// ✅ Good: Specific permissions
{
  "Effect": "Allow",
  "Action": [
    "s3:GetObject",
    "s3:PutObject"
  ],
  "Resource": "arn:aws:s3:::my-app-bucket/*"
}
```

**3. MFA (Multi-Factor Authentication):**
```bash
# Virtual MFA device setup
aws iam create-virtual-mfa-device --virtual-mfa-device-name MyMFADevice

# Enable MFA for user
aws iam enable-mfa-device --user-name john --serial-number arn:aws:iam::123456789012:mfa/MyMFADevice
```

### 🔧 **Hands-on: IAM Setup**

**Step 1: Create User**
```bash
# AWS CLI দিয়ে user তৈরি
aws iam create-user --user-name developer-john

# Password set করা
aws iam create-login-profile --user-name developer-john --password MySecurePassword123!
```

**Step 2: Create Group**
```bash
# Group তৈরি
aws iam create-group --group-name Developers

# User কে group এ add করা
aws iam add-user-to-group --user-name developer-john --group-name Developers
```

**Step 3: Attach Policy**
```bash
# Managed policy attach করা
aws iam attach-group-policy --group-name Developers --policy-arn arn:aws:iam::aws:policy/AmazonEC2ReadOnlyAccess

# Custom policy তৈরি ও attach
aws iam create-policy --policy-name S3BucketAccess --policy-document file://s3-policy.json
aws iam attach-group-policy --group-name Developers --policy-arn arn:aws:iam::123456789012:policy/S3BucketAccess
```

---

## EC2 (Elastic Compute Cloud)

### 🖥️ **EC2 কি?**

EC2 হলো AWS এর **virtual server service** যা cloud এ scalable computing capacity প্রদান করে।

### 🎯 **EC2 Key Concepts:**

**1. Instance Types:**
```
t2.micro    - 1 vCPU, 1GB RAM    (Free tier)
t2.small    - 1 vCPU, 2GB RAM    ($0.023/hour)
t2.medium   - 2 vCPU, 4GB RAM    ($0.046/hour)
m5.large    - 2 vCPU, 8GB RAM    ($0.096/hour)
c5.xlarge   - 4 vCPU, 8GB RAM    ($0.17/hour)
r5.large    - 2 vCPU, 16GB RAM   ($0.126/hour)
```

**2. AMI (Amazon Machine Image):**
```
Amazon Linux 2    - AWS optimized Linux
Ubuntu 20.04 LTS   - Popular Linux distribution
Windows Server     - Microsoft Windows
Custom AMI         - Your own configured image
```

**3. Storage Types:**
```
EBS (Elastic Block Store):
- gp3: General purpose SSD (3,000-16,000 IOPS)
- io2: Provisioned IOPS SSD (up to 64,000 IOPS)
- st1: Throughput optimized HDD
- sc1: Cold HDD (lowest cost)

Instance Store:
- Temporary storage
- High performance
- Data lost on stop/terminate
```

### 🚀 **EC2 Launch Process:**

**Step 1: Choose AMI**
```bash
# Available AMIs দেখা
aws ec2 describe-images --owners amazon --filters "Name=name,Values=amzn2-ami-hvm-*" --query 'Images[*].[ImageId,Name]' --output table
```

**Step 2: Choose Instance Type**
```bash
# Instance types দেখা
aws ec2 describe-instance-types --instance-types t2.micro --query 'InstanceTypes[*].[InstanceType,VCpuInfo.DefaultVCpus,MemoryInfo.SizeInMiB]' --output table
```

**Step 3: Configure Instance**
```bash
# Key pair তৈরি
aws ec2 create-key-pair --key-name MyKeyPair --query 'KeyMaterial' --output text > MyKeyPair.pem
chmod 400 MyKeyPair.pem

# Security group তৈরি
aws ec2 create-security-group --group-name MySecurityGroup --description "My security group"
```

**Step 4: Launch Instance**
```bash
# EC2 instance launch
aws ec2 run-instances \
    --image-id ami-0abcdef1234567890 \
    --count 1 \
    --instance-type t2.micro \
    --key-name MyKeyPair \
    --security-groups MySecurityGroup \
    --user-data file://user-data.sh
```

### 📝 **User Data Script:**
```bash
#!/bin/bash
# user-data.sh - Instance launch এর সময় চলবে

# System update
yum update -y

# Install Apache web server
yum install -y httpd

# Start Apache
systemctl start httpd
systemctl enable httpd

# Create simple web page
echo "<h1>Hello from EC2!</h1>" > /var/www/html/index.html
echo "<p>Instance ID: $(curl -s http://169.254.169.254/latest/meta-data/instance-id)</p>" >> /var/www/html/index.html

# Install PHP
yum install -y php php-mysql

# Install Composer
curl -sS https://getcomposer.org/installer | php
mv composer.phar /usr/local/bin/composer

# Install Node.js
curl -sL https://rpm.nodesource.com/setup_16.x | bash -
yum install -y nodejs
```

### 🔧 **EC2 Management:**

**Instance Operations:**
```bash
# Instance start
aws ec2 start-instances --instance-ids i-1234567890abcdef0

# Instance stop
aws ec2 stop-instances --instance-ids i-1234567890abcdef0

# Instance terminate
aws ec2 terminate-instances --instance-ids i-1234567890abcdef0

# Instance status check
aws ec2 describe-instances --instance-ids i-1234567890abcdef0 --query 'Reservations[*].Instances[*].[InstanceId,State.Name,PublicIpAddress]' --output table
```

**SSH Connection:**
```bash
# SSH দিয়ে connect
ssh -i MyKeyPair.pem ec2-user@54.123.45.67

# File transfer
scp -i MyKeyPair.pem file.txt ec2-user@54.123.45.67:~/
scp -i MyKeyPair.pem -r folder/ ec2-user@54.123.45.67:~/
```

---

## Security Groups

### 🛡️ **Security Groups কি?**

Security Groups হলো **virtual firewall** যা EC2 instances এর traffic control করে।

### 🎯 **Security Group Rules:**

**Inbound Rules (Incoming Traffic):**
```json
{
  "IpPermissions": [
    {
      "IpProtocol": "tcp",
      "FromPort": 22,
      "ToPort": 22,
      "IpRanges": [{"CidrIp": "0.0.0.0/0", "Description": "SSH from anywhere"}]
    },
    {
      "IpProtocol": "tcp", 
      "FromPort": 80,
      "ToPort": 80,
      "IpRanges": [{"CidrIp": "0.0.0.0/0", "Description": "HTTP from anywhere"}]
    },
    {
      "IpProtocol": "tcp",
      "FromPort": 443, 
      "ToPort": 443,
      "IpRanges": [{"CidrIp": "0.0.0.0/0", "Description": "HTTPS from anywhere"}]
    }
  ]
}
```

**Outbound Rules (Outgoing Traffic):**
```json
{
  "IpPermissionsEgress": [
    {
      "IpProtocol": "-1",
      "IpRanges": [{"CidrIp": "0.0.0.0/0", "Description": "All traffic"}]
    }
  ]
}
```

### 🔧 **Common Security Group Configurations:**

**1. Web Server Security Group:**
```bash
# Web server security group তৈরি
aws ec2 create-security-group \
    --group-name WebServerSG \
    --description "Security group for web servers"

# HTTP access allow
aws ec2 authorize-security-group-ingress \
    --group-name WebServerSG \
    --protocol tcp \
    --port 80 \
    --cidr 0.0.0.0/0

# HTTPS access allow  
aws ec2 authorize-security-group-ingress \
    --group-name WebServerSG \
    --protocol tcp \
    --port 443 \
    --cidr 0.0.0.0/0

# SSH access (specific IP only)
aws ec2 authorize-security-group-ingress \
    --group-name WebServerSG \
    --protocol tcp \
    --port 22 \
    --cidr 203.0.113.0/24
```

**2. Database Security Group:**
```bash
# Database security group তৈরি
aws ec2 create-security-group \
    --group-name DatabaseSG \
    --description "Security group for database servers"

# MySQL access from web servers only
aws ec2 authorize-security-group-ingress \
    --group-name DatabaseSG \
    --protocol tcp \
    --port 3306 \
    --source-group WebServerSG
```

**3. Load Balancer Security Group:**
```bash
# Load balancer security group
aws ec2 create-security-group \
    --group-name LoadBalancerSG \
    --description "Security group for load balancer"

# Allow HTTP/HTTPS from internet
aws ec2 authorize-security-group-ingress \
    --group-name LoadBalancerSG \
    --protocol tcp \
    --port 80 \
    --cidr 0.0.0.0/0

aws ec2 authorize-security-group-ingress \
    --group-name LoadBalancerSG \
    --protocol tcp \
    --port 443 \
    --cidr 0.0.0.0/0
```

### 🔍 **Security Group Best Practices:**

**1. Principle of Least Privilege:**
```bash
# ❌ Bad: Too open
--cidr 0.0.0.0/0  # All internet access

# ✅ Good: Specific access
--cidr 203.0.113.0/24  # Specific IP range
--source-group sg-12345678  # Specific security group
```

**2. Regular Audit:**
```bash
# Security groups list করা
aws ec2 describe-security-groups --query 'SecurityGroups[*].[GroupId,GroupName,Description]' --output table

# Unused security groups খুঁজে বের করা
aws ec2 describe-security-groups --query 'SecurityGroups[?length(IpPermissions)==`0` && length(IpPermissionsEgress)==`1`].[GroupId,GroupName]' --output table
```

---

## VPC (Virtual Private Cloud)

### 🌐 **VPC কি?**

VPC হলো AWS cloud এর মধ্যে আপনার **নিজস্ব isolated network** যেখানে আপনি complete control রাখেন।

### 🏗️ **VPC Components:**

**1. CIDR Block:**
```
VPC CIDR: 10.0.0.0/16 (65,536 IP addresses)
├── Public Subnet: 10.0.1.0/24 (256 IPs)
├── Private Subnet: 10.0.2.0/24 (256 IPs)
└── Database Subnet: 10.0.3.0/24 (256 IPs)
```

**2. Subnets:**
```json
{
  "PublicSubnet": {
    "CidrBlock": "10.0.1.0/24",
    "AvailabilityZone": "us-east-1a",
    "MapPublicIpOnLaunch": true
  },
  "PrivateSubnet": {
    "CidrBlock": "10.0.2.0/24", 
    "AvailabilityZone": "us-east-1b",
    "MapPublicIpOnLaunch": false
  }
}
```

**3. Route Tables:**
```json
{
  "PublicRouteTable": {
    "Routes": [
      {"DestinationCidrBlock": "10.0.0.0/16", "Target": "local"},
      {"DestinationCidrBlock": "0.0.0.0/0", "Target": "igw-12345678"}
    ]
  },
  "PrivateRouteTable": {
    "Routes": [
      {"DestinationCidrBlock": "10.0.0.0/16", "Target": "local"},
      {"DestinationCidrBlock": "0.0.0.0/0", "Target": "nat-12345678"}
    ]
  }
}
```

### 🚀 **VPC Setup Step by Step:**

**Step 1: Create VPC**
```bash
# VPC তৈরি
aws ec2 create-vpc --cidr-block 10.0.0.0/16 --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=MyVPC}]'

# VPC ID save করা
VPC_ID=$(aws ec2 describe-vpcs --filters "Name=tag:Name,Values=MyVPC" --query 'Vpcs[0].VpcId' --output text)
echo "VPC ID: $VPC_ID"
```

**Step 2: Create Internet Gateway**
```bash
# Internet Gateway তৈরি
aws ec2 create-internet-gateway --tag-specifications 'ResourceType=internet-gateway,Tags=[{Key=Name,Value=MyIGW}]'

# IGW ID save করা
IGW_ID=$(aws ec2 describe-internet-gateways --filters "Name=tag:Name,Values=MyIGW" --query 'InternetGateways[0].InternetGatewayId' --output text)

# VPC এর সাথে attach করা
aws ec2 attach-internet-gateway --vpc-id $VPC_ID --internet-gateway-id $IGW_ID
```

**Step 3: Create Subnets**
```bash
# Public Subnet তৈরি
aws ec2 create-subnet \
    --vpc-id $VPC_ID \
    --cidr-block 10.0.1.0/24 \
    --availability-zone us-east-1a \
    --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=PublicSubnet}]'

PUBLIC_SUBNET_ID=$(aws ec2 describe-subnets --filters "Name=tag:Name,Values=PublicSubnet" --query 'Subnets[0].SubnetId' --output text)

# Private Subnet তৈরি
aws ec2 create-subnet \
    --vpc-id $VPC_ID \
    --cidr-block 10.0.2.0/24 \
    --availability-zone us-east-1b \
    --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=PrivateSubnet}]'

PRIVATE_SUBNET_ID=$(aws ec2 describe-subnets --filters "Name=tag:Name,Values=PrivateSubnet" --query 'Subnets[0].SubnetId' --output text)
```

**Step 4: Create NAT Gateway**
```bash
# Elastic IP allocate করা
aws ec2 allocate-address --domain vpc --tag-specifications 'ResourceType=elastic-ip,Tags=[{Key=Name,Value=NATGatewayEIP}]'

EIP_ALLOCATION_ID=$(aws ec2 describe-addresses --filters "Name=tag:Name,Values=NATGatewayEIP" --query 'Addresses[0].AllocationId' --output text)

# NAT Gateway তৈরি (Public subnet এ)
aws ec2 create-nat-gateway \
    --subnet-id $PUBLIC_SUBNET_ID \
    --allocation-id $EIP_ALLOCATION_ID \
    --tag-specifications 'ResourceType=nat-gateway,Tags=[{Key=Name,Value=MyNATGateway}]'

NAT_GATEWAY_ID=$(aws ec2 describe-nat-gateways --filter "Name=tag:Name,Values=MyNATGateway" --query 'NatGateways[0].NatGatewayId' --output text)
```

**Step 5: Configure Route Tables**
```bash
# Public Route Table তৈরি
aws ec2 create-route-table --vpc-id $VPC_ID --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=PublicRouteTable}]'

PUBLIC_RT_ID=$(aws ec2 describe-route-tables --filters "Name=tag:Name,Values=PublicRouteTable" --query 'RouteTables[0].RouteTableId' --output text)

# Internet Gateway route add করা
aws ec2 create-route --route-table-id $PUBLIC_RT_ID --destination-cidr-block 0.0.0.0/0 --gateway-id $IGW_ID

# Public subnet associate করা
aws ec2 associate-route-table --subnet-id $PUBLIC_SUBNET_ID --route-table-id $PUBLIC_RT_ID

# Private Route Table তৈরি
aws ec2 create-route-table --vpc-id $VPC_ID --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=PrivateRouteTable}]'

PRIVATE_RT_ID=$(aws ec2 describe-route-tables --filters "Name=tag:Name,Values=PrivateRouteTable" --query 'RouteTables[0].RouteTableId' --output text)

# NAT Gateway route add করা
aws ec2 create-route --route-table-id $PRIVATE_RT_ID --destination-cidr-block 0.0.0.0/0 --nat-gateway-id $NAT_GATEWAY_ID

# Private subnet associate করা
aws ec2 associate-route-table --subnet-id $PRIVATE_SUBNET_ID --route-table-id $PRIVATE_RT_ID
```

### 🔧 **VPC Best Practices:**

**1. Multi-AZ Design:**
```bash
# Multiple Availability Zones ব্যবহার করুন
us-east-1a: Public Subnet (10.0.1.0/24), Private Subnet (10.0.2.0/24)
us-east-1b: Public Subnet (10.0.3.0/24), Private Subnet (10.0.4.0/24)
us-east-1c: Public Subnet (10.0.5.0/24), Private Subnet (10.0.6.0/24)
```

**2. Network ACLs:**
```bash
# Network ACL তৈরি (additional security layer)
aws ec2 create-network-acl --vpc-id $VPC_ID

# Rules add করা
aws ec2 create-network-acl-entry \
    --network-acl-id acl-12345678 \
    --rule-number 100 \
    --protocol tcp \
    --port-range From=80,To=80 \
    --cidr-block 0.0.0.0/0 \
    --rule-action allow
```

---

## Load Balancing (ELB)

### ⚖️ **Elastic Load Balancer কি?**

ELB হলো AWS এর **managed load balancing service** যা incoming traffic কে multiple targets এ distribute করে।

### 🎯 **ELB Types:**

**1. Application Load Balancer (ALB):**
```
Layer: 7 (Application Layer)
Protocol: HTTP, HTTPS
Features: 
- Path-based routing
- Host-based routing
- WebSocket support
- HTTP/2 support
Use Case: Web applications, microservices
```

**2. Network Load Balancer (NLB):**
```
Layer: 4 (Transport Layer)  
Protocol: TCP, UDP, TLS
Features:
- Ultra-high performance
- Static IP addresses
- Preserve source IP
Use Case: Gaming, IoT, real-time applications
```

**3. Gateway Load Balancer (GWLB):**
```
Layer: 3 (Network Layer)
Protocol: GENEVE
Features:
- Third-party appliances
- Transparent network gateway
Use Case: Firewalls, intrusion detection
```

### 🚀 **Application Load Balancer Setup:**

**Step 1: Create Target Group**
```bash
# Target Group তৈরি
aws elbv2 create-target-group \
    --name MyTargetGroup \
    --protocol HTTP \
    --port 80 \
    --vpc-id $VPC_ID \
    --health-check-path /health \
    --health-check-interval-seconds 30 \
    --health-check-timeout-seconds 5 \
    --healthy-threshold-count 2 \
    --unhealthy-threshold-count 3

TARGET_GROUP_ARN=$(aws elbv2 describe-target-groups --names MyTargetGroup --query 'TargetGroups[0].TargetGroupArn' --output text)
```

**Step 2: Register Targets**
```bash
# EC2 instances কে target group এ add করা
aws elbv2 register-targets \
    --target-group-arn $TARGET_GROUP_ARN \
    --targets Id=i-1234567890abcdef0,Port=80 Id=i-0987654321fedcba0,Port=80
```

**Step 3: Create Load Balancer**
```bash
# Application Load Balancer তৈরি
aws elbv2 create-load-balancer \
    --name MyALB \
    --subnets $PUBLIC_SUBNET_ID $PUBLIC_SUBNET_ID_2 \
    --security-groups $LOAD_BALANCER_SG_ID \
    --scheme internet-facing \
    --type application \
    --ip-address-type ipv4

LOAD_BALANCER_ARN=$(aws elbv2 describe-load-balancers --names MyALB --query 'LoadBalancers[0].LoadBalancerArn' --output text)
```

**Step 4: Create Listener**
```bash
# HTTP Listener তৈরি
aws elbv2 create-listener \
    --load-balancer-arn $LOAD_BALANCER_ARN \
    --protocol HTTP \
    --port 80 \
    --default-actions Type=forward,TargetGroupArn=$TARGET_GROUP_ARN

# HTTPS Listener (SSL certificate সহ)
aws elbv2 create-listener \
    --load-balancer-arn $LOAD_BALANCER_ARN \
    --protocol HTTPS \
    --port 443 \
    --certificates CertificateArn=arn:aws:acm:us-east-1:123456789012:certificate/12345678-1234-1234-1234-123456789012 \
    --default-actions Type=forward,TargetGroupArn=$TARGET_GROUP_ARN
```

### 🎯 **Advanced Load Balancer Features:**

**1. Path-based Routing:**
```bash
# Different paths এর জন্য different target groups
aws elbv2 create-rule \
    --listener-arn $LISTENER_ARN \
    --priority 100 \
    --conditions Field=path-pattern,Values="/api/*" \
    --actions Type=forward,TargetGroupArn=$API_TARGET_GROUP_ARN

aws elbv2 create-rule \
    --listener-arn $LISTENER_ARN \
    --priority 200 \
    --conditions Field=path-pattern,Values="/admin/*" \
    --actions Type=forward,TargetGroupArn=$ADMIN_TARGET_GROUP_ARN
```

**2. Host-based Routing:**
```bash
# Different domains এর জন্য different target groups
aws elbv2 create-rule \
    --listener-arn $LISTENER_ARN \
    --priority 100 \
    --conditions Field=host-header,Values="api.example.com" \
    --actions Type=forward,TargetGroupArn=$API_TARGET_GROUP_ARN

aws elbv2 create-rule \
    --listener-arn $LISTENER_ARN \
    --priority 200 \
    --conditions Field=host-header,Values="admin.example.com" \
    --actions Type=forward,TargetGroupArn=$ADMIN_TARGET_GROUP_ARN
```

**3. Health Checks:**
```bash
# Custom health check configuration
aws elbv2 modify-target-group \
    --target-group-arn $TARGET_GROUP_ARN \
    --health-check-path /api/health \
    --health-check-interval-seconds 15 \
    --health-check-timeout-seconds 10 \
    --healthy-threshold-count 2 \
    --unhealthy-threshold-count 5 \
    --matcher HttpCode=200,201
```

---

## Advanced Services

### 📦 **S3 (Simple Storage Service)**

**Basic S3 Operations:**
```bash
# Bucket তৈরি
aws s3 mb s3://my-unique-bucket-name-12345

# File upload
aws s3 cp file.txt s3://my-unique-bucket-name-12345/
aws s3 cp folder/ s3://my-unique-bucket-name-12345/folder/ --recursive

# File download
aws s3 cp s3://my-unique-bucket-name-12345/file.txt ./
aws s3 sync s3://my-unique-bucket-name-12345/ ./local-folder/

# Static website hosting
aws s3 website s3://my-unique-bucket-name-12345 --index-document index.html --error-document error.html
```

### 🌐 **Route 53 (DNS Service)**

**DNS Configuration:**
```bash
# Hosted zone তৈরি
aws route53 create-hosted-zone --name example.com --caller-reference $(date +%s)

# A record তৈরি
aws route53 change-resource-record-sets --hosted-zone-id Z123456789 --change-batch '{
  "Changes": [{
    "Action": "CREATE",
    "ResourceRecordSet": {
      "Name": "www.example.com",
      "Type": "A",
      "TTL": 300,
      "ResourceRecords": [{"Value": "54.123.45.67"}]
    }
  }]
}'
```

### 📊 **CloudWatch (Monitoring)**

**Monitoring Setup:**
```bash
# Custom metric তৈরি
aws cloudwatch put-metric-data \
    --namespace "MyApp/Performance" \
    --metric-data MetricName=ResponseTime,Value=250,Unit=Milliseconds

# Alarm তৈরি
aws cloudwatch put-metric-alarm \
    --alarm-name "HighCPUUtilization" \
    --alarm-description "Alarm when CPU exceeds 70%" \
    --metric-name CPUUtilization \
    --namespace AWS/EC2 \
    --statistic Average \
    --period 300 \
    --threshold 70 \
    --comparison-operator GreaterThanThreshold \
    --evaluation-periods 2
```

---

## DevOps Integration

### 🏗️ **Infrastructure as Code (Terraform)**

**Terraform Configuration:**
```hcl
# main.tf
provider "aws" {
  region = "us-east-1"
}

# VPC
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name = "MainVPC"
  }
}

# Internet Gateway
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "MainIGW"
  }
}

# Public Subnet
resource "aws_subnet" "public" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.1.0/24"
  availability_zone       = "us-east-1a"
  map_public_ip_on_launch = true

  tags = {
    Name = "PublicSubnet"
  }
}

# Security Group
resource "aws_security_group" "web" {
  name_prefix = "web-sg"
  vpc_id      = aws_vpc.main.id

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# EC2 Instance
resource "aws_instance" "web" {
  ami                    = "ami-0abcdef1234567890"
  instance_type          = "t2.micro"
  key_name              = "MyKeyPair"
  vpc_security_group_ids = [aws_security_group.web.id]
  subnet_id             = aws_subnet.public.id

  user_data = <<-EOF
              #!/bin/bash
              yum update -y
              yum install -y httpd
              systemctl start httpd
              systemctl enable httpd
              echo "<h1>Hello from Terraform!</h1>" > /var/www/html/index.html
              EOF

  tags = {
    Name = "WebServer"
  }
}

# Output
output "instance_public_ip" {
  value = aws_instance.web.public_ip
}
```

**Terraform Commands:**
```bash
# Initialize
terraform init

# Plan
terraform plan

# Apply
terraform apply

# Destroy
terraform destroy
```

---

## Hands-on Projects

### 🎯 **Project 1: Simple Web Application**

**Architecture:**
```
Internet → ALB → EC2 (Web Server) → RDS (Database)
                ↓
            CloudWatch (Monitoring)
```

**Implementation Steps:**
```bash
# 1. Create VPC with public/private subnets
# 2. Launch EC2 instances in public subnet
# 3. Create RDS in private subnet
# 4. Setup Application Load Balancer
# 5. Configure CloudWatch monitoring
# 6. Setup Auto Scaling (optional)
```

### 🎯 **Project 2: Scalable WordPress Site**

**Architecture:**
```
Route 53 → CloudFront → ALB → EC2 Auto Scaling Group → RDS Multi-AZ
                                ↓
                            EFS (Shared Storage)
                                ↓
                            S3 (Media Files)
```

### 🎯 **Project 3: Microservices Architecture**

**Architecture:**
```
API Gateway → Lambda Functions → DynamoDB
     ↓              ↓              ↓
   SQS          CloudWatch      S3
```

---

## 🎯 Learning Roadmap

### **Week 1-2: Fundamentals**
```
✅ AWS Account setup
✅ IAM (Users, Groups, Roles, Policies)
✅ EC2 (Launch, Connect, Manage)
✅ Security Groups basics
```

### **Week 3-4: Networking**
```
✅ VPC concepts
✅ Subnets (Public/Private)
✅ Route Tables
✅ Internet Gateway & NAT Gateway
```

### **Week 5-6: Load Balancing & Scaling**
```
✅ Application Load Balancer
✅ Target Groups
✅ Auto Scaling Groups
✅ CloudWatch monitoring
```

### **Week 7-8: Advanced Services**
```
✅ S3 (Storage)
✅ RDS (Database)
✅ Route 53 (DNS)
✅ CloudFront (CDN)
```

### **Week 9-10: DevOps Integration**
```
✅ Infrastructure as Code (Terraform)
✅ CI/CD with CodePipeline
✅ Container services (ECS/EKS)
✅ Serverless (Lambda)
```

---

## 🎯 Quick Reference

### Essential AWS CLI Commands:
```bash
# Configure AWS CLI
aws configure

# EC2 operations
aws ec2 describe-instances
aws ec2 start-instances --instance-ids i-1234567890abcdef0
aws ec2 stop-instances --instance-ids i-1234567890abcdef0

# S3 operations
aws s3 ls
aws s3 cp file.txt s3://bucket-name/
aws s3 sync ./folder s3://bucket-name/folder/

# IAM operations
aws iam list-users
aws iam create-user --user-name newuser
aws iam attach-user-policy --user-name newuser --policy-arn arn:aws:iam::aws:policy/ReadOnlyAccess
```

### Cost Optimization Tips:
```
✅ Use Free Tier services
✅ Stop unused EC2 instances
✅ Use Reserved Instances for long-term
✅ Monitor with CloudWatch
✅ Set up billing alerts
✅ Use Spot Instances for non-critical workloads
```

এই comprehensive guide follow করে আপনি AWS এর core services master করতে পারবেন এবং cloud infrastructure professionally manage করতে পারবেন।