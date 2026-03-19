# Java Application — AWS 3-Tier Architecture

A production-grade Java web application deployed on AWS using a 3-tier architecture with high availability, auto scaling, RDS MySQL, and CloudWatch monitoring.

## Architecture

![AWS Architecture](https://imgur.com/b9iHwVc.png)

![3-Tier Architecture Diagram](https://imgur.com/3XF0tlJ.png)

## Architecture Overview

### Tier 1 — Presentation (Frontend)
- Nginx web servers in Auto Scaling Group
- Public-facing Network Load Balancer
- CloudFront Distribution for static content

### Tier 2 — Application (Backend)
- Apache Tomcat servers in Auto Scaling Group
- Internal Network Load Balancer
- Session management with Amazon ElastiCache

### Tier 3 — Data
- Amazon RDS MySQL in Multi-AZ configuration
- Automated backups and point-in-time recovery
- Read replicas for read-heavy workloads

### Network Design
- Two VPCs (192.168.0.0/16 and 172.32.0.0/16)
- Public and private subnets across multiple AZs
- Transit Gateway for inter-VPC communication

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | Nginx, CloudFront |
| Backend | Java, Spring Boot, Apache Tomcat |
| Database | Amazon RDS MySQL (Multi-AZ) |
| Caching | Amazon ElastiCache |
| Build | Maven, JFrog Artifactory |
| Code Quality | SonarCloud |
| Infrastructure | AWS VPC, EC2, ALB, ASG, RDS |
| Monitoring | CloudWatch, CloudWatch Agent |
| Security | AWS WAF, Shield, GuardDuty, Secrets Manager |

## Project Structure

```
.
├── Java-Login-App/         # Java Spring Boot application source
├── infrastructure/         # AWS infrastructure setup scripts
└── README.md
```

## Prerequisites

- AWS Account with CLI configured
- Java 11+
- Maven 3.6+
- Git

```bash
# Install AWS CLI
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
aws configure
```

## Infrastructure Setup

### 1. VPC and Networking

```bash
# Create primary VPC
aws ec2 create-vpc \
    --cidr-block 192.168.0.0/16 \
    --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=PrimaryVPC}]' \
    --region us-east-1

# Create public subnet
aws ec2 create-subnet \
    --vpc-id vpc-xxx \
    --cidr-block 192.168.1.0/24 \
    --availability-zone us-east-1a

# Create private subnet
aws ec2 create-subnet \
    --vpc-id vpc-xxx \
    --cidr-block 192.168.2.0/24 \
    --availability-zone us-east-1b

# Create and attach Internet Gateway
aws ec2 create-internet-gateway
aws ec2 attach-internet-gateway --vpc-id vpc-xxx --internet-gateway-id igw-xxx

# Create NAT Gateway
aws ec2 create-nat-gateway \
    --subnet-id subnet-xxx \
    --allocation-id eipalloc-xxx
```

### 2. Security Groups

```bash
# Frontend security group
aws ec2 create-security-group \
    --group-name FrontendSG \
    --description "Security group for frontend servers" \
    --vpc-id vpc-xxx

# Allow HTTP/HTTPS
aws ec2 authorize-security-group-ingress \
    --group-id sg-xxx --protocol tcp --port 80 --cidr 0.0.0.0/0

aws ec2 authorize-security-group-ingress \
    --group-id sg-xxx --protocol tcp --port 443 --cidr 0.0.0.0/0
```

### 3. RDS MySQL (Multi-AZ)

```bash
aws rds create-db-instance \
    --db-instance-identifier prod-mysql \
    --db-instance-class db.t3.medium \
    --engine mysql \
    --master-username admin \
    --master-user-password "YourSecurePassword" \
    --allocated-storage 20 \
    --multi-az \
    --vpc-security-group-ids sg-xxx \
    --db-subnet-group-name your-db-subnet-group
```

```sql
-- Initialize database
CREATE DATABASE javaapp;
USE javaapp;

CREATE TABLE users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    username VARCHAR(50) NOT NULL UNIQUE,
    password VARCHAR(255) NOT NULL,
    email VARCHAR(100) NOT NULL UNIQUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_username ON users(username);
CREATE INDEX idx_email ON users(email);
```

### 4. Auto Scaling Group

```bash
# Create launch template
aws ec2 create-launch-template \
    --launch-template-name WebServerTemplate \
    --launch-template-data '{
        "ImageId": "ami-xxx",
        "InstanceType": "t3.micro",
        "SecurityGroupIds": ["sg-xxx"]
    }'

# Create ASG
aws autoscaling create-auto-scaling-group \
    --auto-scaling-group-name WebServerASG \
    --launch-template LaunchTemplateName=WebServerTemplate,Version='$Latest' \
    --min-size 2 \
    --max-size 6 \
    --desired-capacity 2 \
    --vpc-zone-identifier "subnet-xxx,subnet-yyy" \
    --health-check-type ELB \
    --health-check-grace-period 300
```

## Application Setup

### Build

```bash
# Clone the repo
git clone https://github.com/Shubhamharanale7/Java-App-AWS-3-Tier-Architecture.git
cd Java-Login-App

# Build
mvn clean package -DskipTests

# Run tests
mvn test
```

### Deploy to Tomcat

```bash
# Configure Tomcat as a service
sudo tee /etc/systemd/system/tomcat.service << EOF
[Unit]
Description=Apache Tomcat
After=network.target

[Service]
Type=forking
Environment=JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64
Environment=CATALINA_HOME=/opt/tomcat
ExecStart=/opt/tomcat/bin/startup.sh
ExecStop=/opt/tomcat/bin/shutdown.sh
Restart=always

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl start tomcat
sudo systemctl enable tomcat
```

### Nginx Configuration

```nginx
upstream backend {
    server internal-nlb-xxx.elb.amazonaws.com:8080;
}

server {
    listen 80;
    server_name example.com;

    location / {
        proxy_pass http://backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

## Monitoring (CloudWatch)

```bash
# Custom memory metric
MEMORY_USAGE=$(free | grep Mem | awk '{print $3/$2 * 100.0}')
aws cloudwatch put-metric-data \
    --metric-name MemoryUsage \
    --namespace CustomMetrics \
    --value $MEMORY_USAGE
```

## Security Best Practices

- Network ACLs + Security Groups (defense in depth)
- VPC Flow Logs enabled
- AWS WAF for application protection
- AWS Shield for DDoS protection
- AWS GuardDuty for threat detection
- AWS Secrets Manager for credentials
- Encryption at rest and in transit (SSL/TLS)
- Regular automated backups (RDS)

## Troubleshooting

```bash
# Test DB connectivity
telnet database-endpoint 3306

# Check load balancer health
aws elbv2 describe-target-health \
    --target-group-arn arn:aws:elasticloadbalancing:region:account-id:targetgroup/xxx

# Monitor performance
top -bn1
free -m
df -h
```

## Cleanup

```bash
# Delete ASG
aws autoscaling delete-auto-scaling-group --auto-scaling-group-name WebServerASG --force-delete

# Delete RDS
aws rds delete-db-instance --db-instance-identifier prod-mysql --skip-final-snapshot

# Delete VPC resources
aws ec2 delete-vpc --vpc-id vpc-xxx
```

---

**Built by [Shubham Haranale](https://github.com/Shubhamharanale7)**

[![GitHub](https://img.shields.io/badge/GitHub-181717?style=for-the-badge&logo=github&logoColor=white)](https://github.com/Shubhamharanale7)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-%230077B5.svg?style=for-the-badge&logo=linkedin&logoColor=white)](https://linkedin.com/in/shubhamharanale7)
