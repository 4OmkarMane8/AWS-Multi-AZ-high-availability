# AWS-Multi-AZ-high-availability
AWS highly available web application using ALB, Auto Scaling, EC2, VPC and private subnets.


## 🎯 Project Overview
 
This project builds a **three-tier, highly available, fault-tolerant AWS architecture** suitable for production environments. It demonstrates core AWS services including:
 
- **VPC & Networking**: Multi-AZ subnet design with public and private tiers
- **Load Balancing**: Application Load Balancer (ALB) for traffic distribution
- **Auto Scaling**: Dynamic instance management based on demand
- **Security**: IAM roles, security groups, and network ACLs
- **Storage**: EBS volumes and S3 bucket integration
- **Monitoring**: CloudWatch metrics and health checks
  
 
---


🏗️ Architecture


<img width="1111" height="842" alt="Screenshot 2026-09-15 223024" src="https://github.com/user-attachments/assets/dc4306ec-0ef1-4a9f-94f8-e92524d43c21" />



### Key Components
 
| Component | Purpose | Details |
|-----------|---------|---------|
| **VPC** | Network boundary | CIDR: 10.0.0.0/24, Multi-AZ enabled |
| **Public Subnets** | Internet-facing resources | ALB, NAT Gateway |
| **Private Subnets** | Protected application tier | EC2 instances running Apache |
| **Internet Gateway** | Internet connectivity | Routes traffic from public subnets |
| **NAT Gateway** | Outbound internet access | Allows private instances to reach internet |
| **ALB** | Load distribution | Health checks, multi-AZ deployment |
| **Auto Scaling Group** | Dynamic scaling | Min: 2, Max: 3, Target CPU: 50% |
| **Security Groups** | Firewall rules | Restrict traffic by port and source |
| **S3 Bucket** | Object storage | File storage and backup |
 
---
 

## ✅ Prerequisites
 
Before starting, ensure you have:
 
- **AWS Account** with the following permissions:
  - IAM (create users, roles, policies)
  - VPC (create VPC, subnets, route tables, NACLs)
  - EC2 (instances, security groups, load balancers, auto scaling)
  - S3 (bucket creation and management)
  - CloudWatch (monitoring and logging)
  - EBS (volume and snapshot management)
- **AWS CLI** (optional, for automation)
```bash
  # Install AWS CLI v2
  # https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html
```

### Phase 1: IAM Setup (15-20 minutes)
 
#### 1.1 Create IAM User
 
```
AWS Console → IAM → Users → Create User
├── Username: <YourName>
├── Enable MFA
└── Attach Policy: Cloudops-team
```
 
**Custom Policy (Cloudops-team):**
 
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "iam:*",
        "ec2:*",
        "rds:*",
        "cloudwatch:*",
        "sns:*",
        "s3:*"
      ],
      "Resource": "*"
    }
  ]
}
```
 
#### 1.2 Create IAM Role
 
```
AWS Console → IAM → Roles → Create Role
├── Service: EC2
├── Policy: AmazonS3FullAccess
├── Role Name: EC2_S3_Integration
└── Create Role
```
 
---
 
### Phase 2: VPC & Networking (30-40 minutes)
 
#### 2.1 Create VPC
 
```
VPC Console → Create VPC
├── Name: project1
├── IPv4 CIDR: 10.0.0.0/24
└── Create
```
 
#### 2.2 Create Subnets
 
| Subnet | CIDR | AZ |
|--------|------|-----|
| Public1 | 10.0.0.0/26 | us-east-1a |
| Public2 | 10.0.0.64/26 | us-east-1b |
| Private1 | 10.0.0.128/26 | us-east-1a |
| Private2 | 10.0.0.192/26 | us-east-1b |
 
```
VPC Console → Subnets → Create Subnet
├── VPC: project1
├── Subnet Name: <from table>
├── Availability Zone: <from table>
├── IPv4 CIDR: <from table>
└── Create Subnet
```
 
#### 2.3 Create Internet Gateway
 
```
VPC Console → Internet Gateways → Create
├── Name: project1-IGW
└── Attach to VPC: project1
```

 #### 2.4 Create Route Tables
 
**Public Route Table:**
 
```
VPC Console → Route Tables → Create Route Table
├── Name: Public-RT
├── VPC: project1
├── Subnet Associations: Public1, Public2
├── Routes:
│   ├── 10.0.0.0/24 → Local
│   └── 0.0.0.0/0 → Internet Gateway
└── Create
```
 
**Private Route Table:**
 
```
VPC Console → Route Tables → Create Route Table
├── Name: Private-RT
├── VPC: project1
├── Subnet Associations: Private1, Private2
├── Routes (will update after NAT Gateway):
│   ├── 10.0.0.0/24 → Local
│   └── 0.0.0.0/0 → NAT Gateway (after creation)
└── Create
```
 
#### 2.5 Create NAT Gateway
 
```
VPC Console → NAT Gateways → Create NAT Gateway
├── Subnet: Public1
├── Elastic IP: Allocate
└── Create
 
After creation, update Private-RT:
├── Routes → Add Route
├── Destination: 0.0.0.0/0
├── Target: NAT Gateway (from above)
└── Save
```
 
#### 2.6 Create Network ACLs
 
**Public NACL:**
 
```
VPC Console → NACLs → Create Network ACL
├── Name: public-nacl
├── VPC: project1
├── Inbound Rules:
│   ├── 100: ALL, ALL, 0.0.0.0/0, ALLOW
├── Outbound Rules:
│   ├── 100: ALL, ALL, 0.0.0.0/0, ALLOW
├── Subnet Associations: Public1, Public2
└── Create
```
 
**Private NACL:**
 
```
VPC Console → NACLs → Create Network ACL
├── Name: private-nacl
├── VPC: project1
├── Inbound Rules:
│   ├── 100: ALL, ALL, 10.0.0.0/24, ALLOW
├── Outbound Rules:
│   ├── 100: ALL, ALL, 0.0.0.0/0, ALLOW
├── Subnet Associations: Private1, Private2
└── Create
```
 
---

### Phase 3: Security Groups (15 minutes)
 
#### 3.1 Create SG-EC2-ALB
 
```
EC2 Console → Security Groups → Create Security Group
├── Name: SG-EC2-ALB
├── Description: Security group for ALB
├── VPC: project1
├── Inbound Rules:
│   ├── HTTP (80) from 0.0.0.0/0
├── Outbound Rules: Allow All
└── Create
```
 
#### 3.2 Create SG-EC2-Private
 
```
EC2 Console → Security Groups → Create Security Group
├── Name: SG-EC2-Private
├── Description: Security group for private EC2 instances
├── VPC: project1
├── Inbound Rules:
│   ├── HTTP (80) from SG-EC2-ALB
├── Outbound Rules: Allow All
└── Create
```
 
---
 
### Phase 4: Launch Template & ALB (25-35 minutes)
 
#### 4.1 Create Launch Template
 
```
EC2 Console → Launch Templates → Create Launch Template
├── Name: project2
├── AMI: Ubuntu (latest)
├── Instance Type: t3.micro
├── Key Pair: <your-key-pair>
├── Storage: 20 GB
├── Security Group: SG-EC2-Private
├── IAM Instance Profile: EC2_S3_Integration
├── User Data:
│   #!/bin/bash
│   apt update -y
│   apt install apache2 -y
│   systemctl start apache2
│   systemctl enable apache2
│   rm -f /var/www/html/index.html
│   PRIVATE_IP=$(hostname -I | awk '{print $1}')
│   echo "This is $PRIVATE_IP" > /var/www/html/index.html
│   systemctl restart apache2
└── Create
```
 
#### 4.2 Create ALB & Target Group
 
```
EC2 Console → Load Balancers → Create Load Balancer
├── Type: Application Load Balancer
├── Name: project2-ALB
├── Scheme: Internet-facing
├── VPC: project1
├── Subnets: Public1, Public2
├── Security Groups: SG-EC2-ALB
├── Listener Port: 80
└── Next: Configure Routing
 
Create Target Group:
├── Name: project2-TG
├── Protocol: HTTP, Port: 80
├── VPC: project1
├── Health Check Path: /
├── Create
 
Finish ALB creation with project2-TG
```
 
---
 
### Phase 5: Auto Scaling (15-20 minutes)
 
```
EC2 Console → Auto Scaling Groups → Create Auto Scaling Group
├── Name: Project2
├── Launch Template: project2
├── VPC: project1
├── Subnets: Private1, Private2
├── Load Balancing: Attach to project2-TG
├── Enable ELB Health Checks: ✓
├── Desired Capacity: 2
├── Minimum Capacity: 2
├── Maximum Capacity: 3
├── Scaling Policy:
│   ├── Target Tracking Scaling Policy
│   ├── Metric Type: CPU Utilization
│   ├── Target Value: 50
│   └── Instance Warmup: 300 seconds
└── Create
```
 
---
 
### Phase 6: S3 Bucket Creation (5 minutes)
 
```
S3 Console → Create Bucket
├── Bucket Name: project1-storage-<unique-id>
├── Region: <same as EC2>
├── Block Public Access: ✓ (Enabled)
└── Create
```
 
---
 
### Phase 7: Operational Tasks (20-30 minutes)
 
#### 7.1 Create AMI
 
```
EC2 Console → Instances
├── Select running instance
├── Image and templates → Create Image
├── Image Name: project2-custom-ami
└── Create
```
 
#### 7.2 Create Snapshot
 
```
EC2 Console → Volumes
├── Select volume
├── Create Snapshot
├── Description: project2-snapshot
└── Create
```
 
#### 7.3 Create Volume from Snapshot
 
```
EC2 Console → Snapshots
├── Select snapshot
├── Create Volume from Snapshot
├── AZ: <select>
└── Create
```
 
#### 7.4 Copy to Another Region
 
```
EC2 Console → Snapshots
├── Select snapshot
├── Copy Snapshot
├── Destination Region: <other region>
└── Copy
```
 
---
 ## 🔍 Project Components
 
### Core Services
 
**Amazon VPC**
- Isolated network environment
- CIDR: 10.0.0.0/24
- Multi-AZ design for high availability
**Amazon EC2**
- Instance Type: t3.micro (1 vCPU, 1GB RAM)
- AMI: Ubuntu (latest)
- Auto-managed by Auto Scaling Group
**Application Load Balancer**
- Internet-facing
- Health checks every 30 seconds
- Sticky sessions support
**Auto Scaling Group**
- Dynamic scaling based on CPU (50% target)
- Min: 2, Max: 3 instances
- Automatic replacement of unhealthy instances
**Amazon S3**
- Global object storage
- Block public access enabled
- Integrated with EC2 via IAM role
**Amazon CloudWatch**
- Metrics: CPU, Network, Status Checks
- Health checks monitoring
- Custom dashboards and alarms
**AWS Identity & Access Management**
- IAM User: Cloudops team member
- IAM Role: EC2 to S3 integration
- MFA enabled for security
---


 ## ✅ Testing & Validation
 
### Test 1: Access Application
 
```bash
# Get ALB DNS
AWS Console → Load Balancers → project2-ALB → Copy DNS
 
# Test in browser
https://project2-alb-xxx.us-east-1.elb.amazonaws.com
# Should see: "This is <PRIVATE_IP>"
```
 
### Test 2: Verify Load Balancing
 
```bash
# Refresh browser multiple times
# You should see different private IPs from different instances
# This confirms traffic is being distributed
```
 
### Test 3: Test Auto-Healing
 
```
AWS Console → EC2 → Instances
1. Select a running instance
2. Terminate the instance
3. Watch Auto Scaling Group replace it
4. Verify new instance appears in Target Group
5. Refresh ALB in browser - should still work
```
 
### Test 4: Test Auto-Scaling
 
```
CloudWatch → Metrics → EC2
1. Simulate load on an instance:
   ssh -i key.pem ubuntu@<instance-ip>
   sudo apt install stress
   stress --cpu 2 --timeout 300s
 
2. Watch Auto Scaling Group metrics
3. Verify new instance launches when CPU > 50%
4. Verify instance terminates when CPU < 50%
```


 ## 🎓 Key Learnings
 
### Concepts Covered
 
1. **High Availability**
   - Multi-AZ deployment across Availability Zones
   - Redundant components eliminate single points of failure
   - Automatic failover with health checks
2. **Load Balancing**
   - ALB distributes traffic across instances
   - Health checks ensure only healthy instances receive traffic
   - Sticky sessions maintain user sessions
3. **Auto Scaling**
   - Dynamic scaling based on demand (CPU utilization)
   - Automatic replacement of unhealthy instances
   - Cost optimization through right-sizing
4. **Network Design**
   - Public subnets for internet-facing resources
   - Private subnets for application servers
   - NAT Gateway for secure outbound access
5. **Security**
   - IAM roles instead of credentials
   - Security groups for instance-level firewall
   - NACLs for subnet-level protection
   - Least privilege access
6. **Infrastructure as Code**
   - Launch Templates for consistent deployments
   - Reusable configurations
   - Version control for infrastructure
7. **Monitoring & Troubleshooting**
   - CloudWatch metrics for visibility
   - Health checks for proactive monitoring
   - Logs for debugging
---
 
 




