# 📘 AWS শেখার সম্পূর্ণ গাইড (Bangla Version)

## 🎯 Overview
এই গাইডে AWS এর প্রধান সার্ভিসগুলো হাতে-কলমে শেখার জন্য step-by-step টিউটোরিয়াল দেওয়া হয়েছে।

---

## 1️⃣ **Load Balancing (লোড ব্যালেন্সিং)**

### 🎯 কী এবং কেন?
- একাধিক EC2 instance এ ট্রাফিক ভাগ করে দেওয়া
- High availability এবং fault tolerance নিশ্চিত করা

### 📚 শেখার পয়েন্ট:
- **ELB Types**: Application LB, Network LB, Gateway LB
- **Target Groups**: EC2, Lambda, IP addresses একসাথে manage করা
- **Health Checks**: Unhealthy instances automatically remove করা
- **Listeners & Rules**: Port এবং protocol based routing

### 🛠️ Hands-on Exercise:
```bash
# 1. Create 2 EC2 instances
aws ec2 run-instances --image-id ami-0abcdef1234567890 --count 2 --instance-type t2.micro

# 2. Create Application Load Balancer
aws elbv2 create-load-balancer --name my-alb --subnets subnet-12345 subnet-67890

# 3. Create Target Group
aws elbv2 create-target-group --name my-targets --protocol HTTP --port 80 --vpc-id vpc-12345

# 4. Register instances to target group
aws elbv2 register-targets --target-group-arn arn:aws:elasticloadbalancing:... --targets Id=i-1234567890abcdef0
```

### 💡 Real Example:
```
User Request → ALB → Target Group → [EC2-1, EC2-2, EC2-3]
```

---

## 2️⃣ **ACM (AWS Certificate Manager)**

### 🎯 কী এবং কেন?
- SSL/TLS certificates free তে পাওয়া এবং auto-renewal
- HTTPS security নিশ্চিত করা

### 📚 শেখার পয়েন্ট:
- **Public Certificate**: Internet-facing applications এর জন্য
- **Private Certificate**: Internal applications এর জন্য
- **Domain Validation**: Email বা DNS record দিয়ে verify করা
- **Integration**: ALB, CloudFront এর সাথে attach করা

### 🛠️ Hands-on Exercise:
```bash
# 1. Request SSL certificate
aws acm request-certificate --domain-name example.com --validation-method DNS

# 2. Get certificate details
aws acm describe-certificate --certificate-arn arn:aws:acm:...

# 3. Attach to Load Balancer
aws elbv2 create-listener --load-balancer-arn arn:aws:elasticloadbalancing:... \
  --protocol HTTPS --port 443 --certificates CertificateArn=arn:aws:acm:...
```

### 💡 Real Example:
```
https://myapp.com → ACM Certificate → ALB → EC2 Instances
```

---

## 3️⃣ **Route 53 (DNS Service)**

### 🎯 কী এবং কেন?
- Domain name কে IP address এ convert করা
- Traffic routing এবং health monitoring

### 📚 শেখার পয়েন্ট:
- **Hosted Zones**: Domain এর DNS records manage করা
- **Record Types**: A, AAAA, CNAME, MX, TXT
- **Routing Policies**: Simple, Weighted, Latency-based, Failover
- **Health Checks**: Endpoint monitoring এবং automatic failover

### 🛠️ Hands-on Exercise:
```bash
# 1. Create hosted zone
aws route53 create-hosted-zone --name example.com --caller-reference $(date +%s)

# 2. Create A record pointing to ALB
aws route53 change-resource-record-sets --hosted-zone-id Z123456789 --change-batch '{
  "Changes": [{
    "Action": "CREATE",
    "ResourceRecordSet": {
      "Name": "www.example.com",
      "Type": "A",
      "AliasTarget": {
        "DNSName": "my-alb-123456789.us-east-1.elb.amazonaws.com",
        "EvaluateTargetHealth": false,
        "HostedZoneId": "Z35SXDOTRQ7X7K"
      }
    }
  }]
}'
```

### 💡 Real Example:
```
www.example.com → Route 53 → ALB → Target Group → EC2
```

---

## 4️⃣ **S3 (Simple Storage Service)**

### 🎯 কী এবং কেন?
- Unlimited file storage in cloud
- Static website hosting এবং backup solution

### 📚 শেখার পয়েন্ট:
- **Buckets**: Container for objects
- **Objects**: Files with metadata
- **Storage Classes**: Standard, IA, Glacier, Deep Archive
- **Permissions**: Bucket policies, ACLs, IAM roles
- **Lifecycle Rules**: Automatic data transition এবং deletion

### 🛠️ Hands-on Exercise:
```bash
# 1. Create S3 bucket
aws s3 mb s3://my-website-bucket-12345

# 2. Upload website files
aws s3 cp index.html s3://my-website-bucket-12345/
aws s3 cp style.css s3://my-website-bucket-12345/

# 3. Enable static website hosting
aws s3 website s3://my-website-bucket-12345 --index-document index.html

# 4. Set public read policy
aws s3api put-bucket-policy --bucket my-website-bucket-12345 --policy '{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "PublicReadGetObject",
    "Effect": "Allow",
    "Principal": "*",
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::my-website-bucket-12345/*"
  }]
}'
```

### 💡 Real Example:
```html
<!-- index.html -->
<!DOCTYPE html>
<html>
<head>
    <title>My AWS Website</title>
</head>
<body>
    <h1>Hello from S3!</h1>
</body>
</html>
```

---

## 5️⃣ **CloudFront (CDN)**

### 🎯 কী এবং কেন?
- Global content delivery network
- Fast loading এবং reduced latency

### 📚 শেখার পয়েন্ট:
- **Distributions**: CDN configuration
- **Origins**: S3 bucket বা ALB
- **Edge Locations**: Worldwide caching points
- **Cache Behaviors**: Different rules for different paths
- **SSL/TLS**: ACM certificate integration

### 🛠️ Hands-on Exercise:
```bash
# 1. Create CloudFront distribution
aws cloudfront create-distribution --distribution-config '{
  "CallerReference": "'$(date +%s)'",
  "Origins": {
    "Quantity": 1,
    "Items": [{
      "Id": "S3-my-website-bucket-12345",
      "DomainName": "my-website-bucket-12345.s3.amazonaws.com",
      "S3OriginConfig": {
        "OriginAccessIdentity": ""
      }
    }]
  },
  "DefaultCacheBehavior": {
    "TargetOriginId": "S3-my-website-bucket-12345",
    "ViewerProtocolPolicy": "redirect-to-https",
    "MinTTL": 0,
    "ForwardedValues": {
      "QueryString": false,
      "Cookies": {"Forward": "none"}
    }
  },
  "Comment": "My website CDN",
  "Enabled": true
}'
```

### 💡 Real Example:
```
User (Bangladesh) → CloudFront Edge (Singapore) → S3 (US-East-1)
                     ↓ (Cached)
User (Bangladesh) → CloudFront Edge (Singapore) ✅ Fast!
```

---

## 6️⃣ **RDS (Relational Database Service)**

### 🎯 কী এবং কেন?
- Managed database service
- Automatic backups, patching, monitoring

### 📚 শেখার পয়েন্ট:
- **DB Engines**: MySQL, PostgreSQL, Oracle, SQL Server
- **Multi-AZ**: High availability setup
- **Read Replicas**: Read performance scaling
- **Parameter Groups**: Database configuration
- **Security Groups**: Network access control

### 🛠️ Hands-on Exercise:
```bash
# 1. Create RDS MySQL instance
aws rds create-db-instance \
  --db-instance-identifier myapp-db \
  --db-instance-class db.t3.micro \
  --engine mysql \
  --master-username admin \
  --master-user-password MySecurePassword123 \
  --allocated-storage 20 \
  --vpc-security-group-ids sg-12345678

# 2. Connect from EC2
mysql -h myapp-db.abcdefghijk.us-east-1.rds.amazonaws.com -u admin -p

# 3. Create database and table
CREATE DATABASE myapp;
USE myapp;
CREATE TABLE users (
  id INT AUTO_INCREMENT PRIMARY KEY,
  name VARCHAR(100),
  email VARCHAR(100)
);
```

### 💡 Real Example:
```php
// Laravel .env
DB_CONNECTION=mysql
DB_HOST=myapp-db.abcdefghijk.us-east-1.rds.amazonaws.com
DB_PORT=3306
DB_DATABASE=myapp
DB_USERNAME=admin
DB_PASSWORD=MySecurePassword123
```

---

## 7️⃣ **Lambda (Serverless Functions)**

### 🎯 কী এবং কেন?
- Code run করা without managing servers
- Event-driven এবং pay-per-execution

### 📚 শেখার পয়েন্ট:
- **Function**: Code package with runtime
- **Triggers**: API Gateway, S3, DynamoDB, CloudWatch
- **Environment Variables**: Configuration management
- **IAM Roles**: Permissions for AWS services
- **Layers**: Shared code এবং dependencies

### 🛠️ Hands-on Exercise:

#### Python Lambda Function:
```python
# lambda_function.py
import json

def lambda_handler(event, context):
    name = event.get('name', 'World')
    
    return {
        'statusCode': 200,
        'body': json.dumps({
            'message': f'Hello {name}!',
            'event': event
        })
    }
```

#### Deploy করা:
```bash
# 1. Create deployment package
zip function.zip lambda_function.py

# 2. Create Lambda function
aws lambda create-function \
  --function-name hello-world \
  --runtime python3.9 \
  --role arn:aws:iam::123456789012:role/lambda-execution-role \
  --handler lambda_function.lambda_handler \
  --zip-file fileb://function.zip

# 3. Test function
aws lambda invoke \
  --function-name hello-world \
  --payload '{"name": "AWS"}' \
  response.json
```

### 💡 Real Example:
```
API Gateway → Lambda → DynamoDB
     ↓
{
  "statusCode": 200,
  "body": "Data saved successfully"
}
```

---

## 8️⃣ **EKS (Elastic Kubernetes Service)**

### 🎯 কী এবং কেন?
- Managed Kubernetes cluster
- Container orchestration এবং scaling

### 📚 শেখার পয়েন্ট:
- **Control Plane**: Kubernetes API server (AWS managed)
- **Node Groups**: EC2 instances for running pods
- **Pods**: Smallest deployable units
- **Services**: Load balancing for pods
- **Ingress**: External access to services

### 🛠️ Hands-on Exercise:

#### Create EKS Cluster:
```bash
# 1. Install eksctl
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin

# 2. Create cluster
eksctl create cluster \
  --name my-cluster \
  --version 1.24 \
  --region us-east-1 \
  --nodegroup-name standard-workers \
  --node-type t3.medium \
  --nodes 2 \
  --nodes-min 1 \
  --nodes-max 3
```

#### Deploy Application:
```yaml
# nginx-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.20
        ports:
        - containerPort: 80

---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
  type: LoadBalancer
```

```bash
# Deploy to EKS
kubectl apply -f nginx-deployment.yaml

# Check status
kubectl get pods
kubectl get services
```

### 💡 Real Example:
```
Internet → ALB → EKS Service → Nginx Pods [Pod1, Pod2, Pod3]
```

---

## 🔧 **Bonus: Infrastructure as Code**

### Terraform Example:
```hcl
# main.tf
provider "aws" {
  region = "us-east-1"
}

resource "aws_instance" "web" {
  ami           = "ami-0abcdef1234567890"
  instance_type = "t2.micro"
  
  tags = {
    Name = "HelloWorld"
  }
}

resource "aws_s3_bucket" "website" {
  bucket = "my-website-bucket-${random_string.suffix.result}"
}

resource "random_string" "suffix" {
  length  = 8
  special = false
  upper   = false
}
```

```bash
# Deploy infrastructure
terraform init
terraform plan
terraform apply
```

---

## 📊 **Cost Optimization Tips**

1. **EC2**: Use Spot Instances for non-critical workloads
2. **S3**: Use appropriate storage classes
3. **RDS**: Use Reserved Instances for production
4. **Lambda**: Optimize memory allocation
5. **CloudWatch**: Set up billing alerts

---

## 🎯 **Next Steps**

1. **Practice**: Create a full-stack application using these services
2. **Certification**: Consider AWS Solutions Architect Associate
3. **Advanced Topics**: ECS, API Gateway, DynamoDB, SQS/SNS
4. **Monitoring**: CloudWatch, X-Ray, CloudTrail
5. **Security**: WAF, Shield, GuardDuty

---

## 📚 **Useful Commands Cheat Sheet**

```bash
# AWS CLI Configuration
aws configure

# List all S3 buckets
aws s3 ls

# List EC2 instances
aws ec2 describe-instances

# Get Lambda functions
aws lambda list-functions

# EKS cluster info
eksctl get cluster

# Check CloudFormation stacks
aws cloudformation list-stacks
```

---

## 🔗 **Resources**

- [AWS Free Tier](https://aws.amazon.com/free/)
- [AWS Documentation](https://docs.aws.amazon.com/)
- [AWS CLI Reference](https://docs.aws.amazon.com/cli/)
- [Terraform AWS Provider](https://registry.terraform.io/providers/hashicorp/aws/)

---

**Happy Learning! 🚀**

> এই গাইড follow করে step-by-step practice করলে AWS এর core services গুলো ভালোভাবে শিখতে পারবেন। প্রতিটি service এর জন্য hands-on practice করা খুবই গুরুত্বপূর্ণ।