# Cloud-Native Refactoring of a Multi-Tier Web Application on AWS Using Elastic Beanstalk, RDS, ElastiCache, Amazon MQ, and CloudFront

![architecture](./images/project-banner.png)

## Project Overview

This project demonstrates how to refactor a traditional multi-tier Java web application from self-managed infrastructure into a cloud-native architecture using AWS managed services. The application stack consists of:

* Java/Tomcat web application (VProfile)
* MySQL database
* Memcached caching layer
* RabbitMQ messaging service

The refactored architecture replaces self-managed components with AWS managed services:

| Traditional Component   | AWS Managed Service     |
| ----------------------- | ----------------------- |
| Tomcat on EC2           | Elastic Beanstalk       |
| MySQL on EC2            | Amazon RDS              |
| Memcached on EC2        | Amazon ElastiCache      |
| RabbitMQ on EC2         | Amazon MQ               |
| DNS                     | Route 53 / External DNS |
| Global Content Delivery | Amazon CloudFront       |
| SSL Certificates        | AWS Certificate Manager |

---

# Architecture

![architecture](./images/architecture-drawing.png)

---

# Prerequisites

Before starting:

* AWS Account
* AWS CLI configured
* Git installed
* Java 17+
* Maven 3.9+
* Visual Studio Code (optional)
* Registered Domain Name (optional)
* SSL Certificate in AWS ACM (for HTTPS)

---

# Step 1: Create Backend Security Group

Navigate to:

```text
EC2 → Security Groups → Create Security Group
```

Create:

```text
Name: vprofile-rearch-backend-sg
```

After the security group is created:

1. Open the security group.
2. Select **Edit Inbound Rules**.
3. Add a rule:

| Type        | Source              |
| ----------- | ------------------- |
| All Traffic | Same Security Group |

This allows communication between backend services.

---

# Step 2: Create EC2 Key Pair

Navigate to:

```text
EC2 → Key Pairs → Create Key Pair
```

Create:

```text
Name: vprofile-rearch-key
Format: PEM
```

Download and save securely.

---

# Step 3: Create Amazon RDS MySQL

## Create Parameter Group

```text
RDS → Parameter Groups → Create
```

Settings:

```text
Name: vprofile-rds-rearch-paragroup
Engine: MySQL 8.4
Type: DB Parameter Group
```

---

## Create Subnet Group

```text
RDS → Subnet Groups → Create
```

Settings:

```text
Name: vprofile-rds-rearch-subgroup
VPC: Default VPC
Subnets: All Available
```

---

## Create Database

```text
RDS → Create Database
```

Configuration:

```text
Engine: MySQL 8.4
Template: Dev/Test
Deployment: Single AZ
```

Instance:

```text
Identifier: vprofile-rds-rearch
Username: admin
Password: Auto-generated
```

Storage:

```text
GP3
20 GB
```

Networking:

```text
Private Access
Backend Security Group
```

Database Name:

```text
accounts
```

Record:

* Endpoint
* Username
* Password

---

# Step 4: Create Amazon ElastiCache

## Create Parameter Group

```text
ElastiCache → Parameter Groups
```

```text
Name: vprofile-rearch-cache-paragroup
Family: Memcached 1.6
```

---

## Create Subnet Group

```text
ElastiCache → Subnet Groups
```

```text
Name: vprofile-rearch-cache-subgroup
```

---

## Create Memcached Cluster

```text
Engine: Memcached
Version: 1.6.x
```

Settings:

```text
Cluster Name: vprofile-rearch-cache
Node Type: T3 Micro
Nodes: 1
Port: 11211
```

Networking:

```text
Subnet Group: vprofile-rearch-cache-subgroup
Security Group: Backend Security Group
```

Disable:

```text
Encryption in Transit
```

Record:

```text
Configuration Endpoint
```

---

# Step 5: Create Amazon MQ

Navigate to:

```text
Amazon MQ → Create Broker
```

Configuration:

```text
Broker Engine: RabbitMQ
Deployment: Single Instance
```

Settings:

```text
Name: vprofile-rearch-rabbitmq
Username: rabbit
Password: Secure Password
Version: RabbitMQ 3.13
```

Networking:

```text
Private Access
Default VPC
Backend Security Group
```

Record:

* Endpoint
* Username
* Password

---

# Step 6: Initialise Amazon RDS

## Launch Temporary EC2

Create:

```text
Ubuntu 24.04
T2 Micro
```

Create a security group for the instance with:

| Protocol | Port | Source |
| -------- | ---- | ------ |
| SSH      | 22   | My IP  |

This allows secure SSH access from your workstation. 

After the instance launches, connect using SSH:

```bash
ssh -i <key-file.pem> ubuntu@<public-ip>
```

Once connected:

Install tools:

```bash
sudo apt update
sudo apt install mysql-client git -y
```

---

## Allow MySQL Access

Add rule to backend security group:

| Type  | Port |
| ----- | ---- |
| MySQL | 3306 |

Source:

```text
Temporary EC2 Security Group
```

---

## Clone Source Code

```bash
git clone https://github.com/OlumideOlumayegun/cloud-native-refactoring-of-web-app.git
cd cloud-native-refactoring-of-web-app
```

---

## Import Database

```bash
mysql -h <RDS-ENDPOINT> -u admin -p accounts < src/main/resources/db_backup.sql
```

Verify:

```sql
SHOW TABLES;
```

Terminate temporary EC2.

---

# Step 7: Create Elastic Beanstalk IAM Role

Navigate to:

```text
IAM → Roles → Create Role
```

Use:

```text
AWS Service → EC2
```

Attach:

* AWS Elastic Beanstalk Custom Platform for EC2 Role: ***```AWSElasticBeanstalkCustomPlatformforEC2Role```***
* AWS Elastic Beanstalk Web Tier: ***```AWSElasticBeanstalkWebTier```***
* AWS Elastic Beanstalk Administrator Access: ***```AdministratorAcess-AWSElasticBeanstalk```***
* SNS permissions for notification: ***```AWSElasticBeanstalkRoleSNS```***

Name:

```text
vprofile-rearch-bean-role
```

---

# Step 8: Create Elastic Beanstalk Environment

Navigate to:

```text
Elastic Beanstalk → Create Application
```

Settings:

```text
Application: VProfile-Rearch
Environment: Web Server
```

Platform:

```text
Tomcat 10
Corretto 21
```

---

## Service Access

Select:

```text
Beanstalk Service Role
vprofile-rearch-bean-role
```

Key Pair:

```text
vprofile-rearch-key
```

---

## Networking

```text
Default VPC
Public IP Enabled
All Availability Zones
```

Do not create an RDS instance.

---

## Instances

```text
Type: T2 Micro
Min Instances: 2
Max Instances: 4
```

Root Volume:

```text
GP3
```

---

## Load Balancer

```text
Application Load Balancer
Port 80
Public
```

Enable:

```text
Session Stickiness
```

---

## Monitoring

```text
Enhanced Health Reporting
```

---

## Deployment Strategy

```text
Rolling
Batch Size: 50%
```

Launch environment.

Wait until status becomes:

```text
Healthy
```

---

# Step 9: Configure Security Group Access

Locate:

```text
Elastic Beanstalk Instance Security Group
```

Add rule to:

```text
vprofile-rearch-backend-sg
```

Rule:

| Type        | Source                            |
| ----------- | --------------------------------- |
| All Traffic | Beanstalk Instance Security Group |

Do NOT use Load Balancer Security Group.

---

# Step 10: Build Application Artifact

Clone repository locally:

```bash
git clone https://github.com/OlumideOlumayegun/cloud-native-refactoring-of-web-app.git
cd cloud-native-refactoring-of-web-app

```

---

## Update application.properties

Replace:

### Database

```properties
db.host=<RDS-ENDPOINT>
db.user=admin
db.password=<PASSWORD>
```

### Memcached

```properties
memcached.host=<ELASTICACHE-ENDPOINT>
memcached.port=11211
```

### RabbitMQ

```properties
rabbitmq.host=<AMAZONMQ-ENDPOINT>
rabbitmq.port=5671
rabbitmq.user=rabbit
rabbitmq.password=<PASSWORD>
```

---

## Build Artifact

Verify:

```bash
mvn -version
```

Build:

```bash
mvn install
```

Artifact:

```text
target/vprofile-v2.war
```

---

# Step 11: Deploy to Elastic Beanstalk

Navigate to:

```text
Elastic Beanstalk → Upload and Deploy
```

Upload:

```text
target/vprofile-v2.war
```

Version:

```text
vprofile-rearch-v1.0
```

Deploy.

Monitor:

```text
Events
Target Group Health
```

Wait for deployment completion.

---

# Step 12: Enable HTTPS

Navigate to:

```text
Elastic Beanstalk → Configuration
```

Add Listener:

```text
HTTPS
Port: 443
```

Select ACM certificate.

Apply changes.

---

# Step 13: Configure DNS

At GoDaddy or Route 53:

Create:

```text
CNAME
```

Example:

```text
vprofile.example.com
```

Point to:

```text
Elastic Beanstalk Endpoint
```

Verify HTTPS access.

---

# Step 14: Create CloudFront Distribution

Navigate to:

```text
CloudFront → Create Distribution
```

Origin:

```text
Elastic Beanstalk Load Balancer
```

Settings:

```text
Match Viewer Protocol
HTTP + HTTPS
All HTTP Methods
```

Create Distribution.

---

# Step 15: Add Custom Domain to CloudFront

Configure:

```text
Alternate Domain Name (CNAME)
```

Example:

```text
vprofile.example.com
```

Associate ACM Certificate.

Deploy.

---

# Step 16: Update DNS to CloudFront

Replace previous CNAME:

```text
vprofile.example.com
```

New target:

```text
xxxxxxxx.cloudfront.net
```

Wait for propagation.

---

# Step 17: Validate Application

Access:

```text
https://vprofile.example.com
```

Login:

```text
Username: admin_vp
Password: admin_vp
```

Verify:

* Database connectivity
* RabbitMQ connectivity
* Memcached functionality
* Session persistence
* HTTPS certificate

---

# Step 18: Verify CloudFront

Open:

```text
F12 → Network → Headers
```

Look for:

```text
Via: cloudfront.net
```

This confirms requests are being served through CloudFront edge locations.

---

# Project Outcomes

By completing this project, you have:

✅ Refactored a traditional multi-tier application into a cloud-native architecture

✅ Replaced self-managed infrastructure with AWS managed services

✅ Implemented Elastic Beanstalk application hosting

✅ Deployed Amazon RDS, ElastiCache, and Amazon MQ

✅ Configured Auto Scaling and Load Balancing

✅ Secured the application with HTTPS and ACM

✅ Delivered content globally through CloudFront

✅ Applied DevOps deployment strategies including rolling deployments

✅ Built a scalable, resilient, production-style AWS application platform suitable for real-world enterprise workloads.
