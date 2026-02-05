# AWS Deployment Guide

This guide provides step-by-step instructions to deploy the Flask application on AWS Elastic Beanstalk with RDS MySQL.

## Table of Contents

1. [VPC & Networking Setup](#step-1-vpc--networking-setup)
2. [Internet Gateway & Route Tables](#step-2-internet-gateway--route-tables)
3. [Security Groups Configuration](#step-3-security-groups-configuration)
4. [RDS MySQL Setup](#step-4-rds-mysql-setup)
5. [Database Schema Setup](#step-5-database-schema-setup)
6. [Application Preparation](#step-6-application-preparation)
7. [Elastic Beanstalk Deployment](#step-7-elastic-beanstalk-deployment)
8. [Testing the Application](#step-8-testing-the-application)
9. [Additional Configuration](#additional-configuration)

---

## Step 1: VPC & Networking Setup

### Create VPC

1. Navigate to **VPC → Your VPCs → Create VPC**
2. Configure the VPC:

| Setting | Value |
|---------|-------|
| Name | `project-vpc` |
| IPv4 CIDR | `10.0.0.0/16` |
| Tenancy | Default |

### Create Subnets

Create 4 subnets across 2 Availability Zones:

| Subnet Name | CIDR Block | Availability Zone | Type |
|-------------|------------|-------------------|------|
| Public-A | `10.0.1.0/24` | AZ1 (e.g., us-east-1a) | Public |
| Public-B | `10.0.2.0/24` | AZ2 (e.g., us-east-1b) | Public |
| Private-A | `10.0.3.0/24` | AZ1 (e.g., us-east-1a) | Private |
| Private-B | `10.0.4.0/24` | AZ2 (e.g., us-east-1b) | Private |

> **Note**: Having subnets in 2 AZs provides high availability for both the application and database.

---

## Step 2: Internet Gateway & Route Tables

### Create Internet Gateway

1. Go to **VPC → Internet Gateways → Create internet gateway**
2. Name: `project-igw`
3. Attach to `project-vpc`

### Configure Route Tables

#### Public Route Table

1. Create route table: `public-rt`
2. Add route:
   - **Destination**: `0.0.0.0/0`
   - **Target**: `project-igw`
3. Associate subnets: `Public-A`, `Public-B`

#### Private Route Table

1. Create route table: `private-rt`
2. **No internet route needed** (keeps RDS isolated)
3. Associate subnets: `Private-A`, `Private-B`

### Enable Auto-assign Public IP

For public subnets, enable auto-assign public IPv4:

1. Select `Public-A` → Actions → Edit subnet settings
2. Enable **Auto-assign public IPv4 address**
3. Repeat for `Public-B`

---

## Step 3: Security Groups Configuration

Create three security groups in `project-vpc`:

### SG-ALB (Load Balancer)

| Rule Type | Protocol | Port | Source | Description |
|-----------|----------|------|--------|-------------|
| Inbound | TCP | 80 | `0.0.0.0/0` | HTTP traffic |
| Inbound | TCP | 443 | `0.0.0.0/0` | HTTPS traffic |
| Outbound | All | All | `0.0.0.0/0` | All traffic |

### SG-EC2 (Elastic Beanstalk Instances)

| Rule Type | Protocol | Port | Source | Description |
|-----------|----------|------|--------|-------------|
| Inbound | TCP | 80 | SG-ALB | HTTP from Load Balancer |
| Inbound | TCP | 22 | Your IP | SSH access (optional) |
| Outbound | All | All | `0.0.0.0/0` | All traffic |

### SG-RDS (MySQL Database)

| Rule Type | Protocol | Port | Source | Description |
|-----------|----------|------|--------|-------------|
| Inbound | TCP | 3306 | SG-EC2 | MySQL from EB instances |
| Outbound | All | All | `0.0.0.0/0` | All traffic |

> **Tip**: Temporarily add your IP to SG-RDS inbound (port 3306) for initial database setup, then remove it.

---

## Step 4: RDS MySQL Setup

### Create DB Subnet Group

1. Go to **RDS → Subnet groups → Create DB subnet group**
2. Configure:

| Setting | Value |
|---------|-------|
| Name | `project-db-subnet-group` |
| VPC | `project-vpc` |
| Subnets | `Private-A`, `Private-B` |

### Create RDS Instance

1. Go to **RDS → Create database**
2. Configure:

| Setting | Value |
|---------|-------|
| Engine | MySQL |
| Version | 8.0 (or latest) |
| Template | Free tier / Dev/Test |
| DB Instance Identifier | `prodb` |
| Master Username | `admin` |
| Master Password | (choose a strong password) |
| DB Instance Class | `db.t3.micro` |
| Storage | 20 GB gp2 |
| VPC | `project-vpc` |
| Subnet Group | `project-db-subnet-group` |
| Public Access | **No** |
| Security Group | `SG-RDS` |
| Initial Database | `insured` |

3. Wait for the instance to become available (~5-10 minutes)
4. **Copy the RDS Endpoint** - you'll need this for the Flask application

---

## Step 5: Database Schema Setup

### Option A: Direct Connection (Temporary)

> ⚠️ **Security Note**: Only use this for initial setup. Remove the rule after configuration.

1. Add your IP to `SG-RDS` inbound rules (port 3306)
2. Connect from your local machine:

```bash
mysql -h <RDS-ENDPOINT> -u admin -p
```

3. Execute the schema:

```sql
USE insured;

CREATE TABLE claims (
    id INT AUTO_INCREMENT PRIMARY KEY,
    policy_id VARCHAR(255) NOT NULL,
    name VARCHAR(255) NOT NULL,
    dob DATE NOT NULL,
    mobile VARCHAR(20) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

4. **Remove your IP from SG-RDS** after setup is complete.

### Option B: Bastion Host (Recommended for Production)

1. **Launch Bastion EC2** in a public subnet:
   - AMI: Amazon Linux 2023
   - Instance type: t2.micro
   - Subnet: Public-A
   - Security Group: Allow SSH (22) from your IP only

2. **Update SG-RDS**: Allow MySQL (3306) from bastion's security group

3. **SSH into bastion**:

```bash
ssh -i ~/keys/mykey.pem ec2-user@<BASTION_PUBLIC_IP>
```

4. **Install MySQL client**:

```bash
# Amazon Linux 2023
sudo dnf install -y mariadb105

# Amazon Linux 2
sudo yum install -y mariadb

# Ubuntu
sudo apt-get update && sudo apt-get install -y mysql-client
```

5. **Connect to RDS**:

```bash
mysql -h <RDS-ENDPOINT> -u admin -p
```

6. **Create the schema** (same SQL as above)

---

## Step 6: Application Preparation

### Required Files for Deployment

Ensure your deployment package contains:

```
├── application.py
├── requirements.txt
├── static/
│   └── (your static files)
└── templates/
    └── (your template files)
```

### Create Deployment Package

**Important**: Zip the contents, not the folder itself.

```bash
# Navigate to project directory
cd AWS-Flask---RDS-Deployment-on-AWS-Elastic-Beanstalk

# Create zip (Linux/Mac)
zip -r ../deploy.zip . -x "*.git*" -x "Docs/*" -x "*.md"

# Create zip (Windows PowerShell)
Compress-Archive -Path .\* -DestinationPath ..\deploy.zip -Force
```

---

## Step 7: Elastic Beanstalk Deployment

### Create Application

1. Go to **Elastic Beanstalk → Create Application**
2. Configure:

| Setting | Value |
|---------|-------|
| Application name | `flask-insured-app` |
| Platform | Python |
| Platform branch | Python 3.x running on 64bit Amazon Linux 2023 |
| Application code | Upload your code (`deploy.zip`) |

### Configure Environment

1. Click **Configure more options**

#### Network Settings

| Setting | Value |
|---------|-------|
| VPC | `project-vpc` |
| Load balancer subnets | `Public-A`, `Public-B` |
| Instance subnets | `Public-A`, `Public-B` |
| Database subnets | (leave empty - using external RDS) |

#### Instance Settings

| Setting | Value |
|---------|-------|
| Security groups | `SG-EC2` |
| Key pair | Your SSH key (optional) |

#### Load Balancer Settings

| Setting | Value |
|---------|-------|
| Load balancer type | Application Load Balancer |
| Security groups | `SG-ALB` |

#### Software Settings

Add environment variables:

| Name | Value |
|------|-------|
| `DB_HOST` | `<your-rds-endpoint>` |
| `DB_USER` | `admin` |
| `DB_PASSWORD` | `<your-password>` |
| `DB_NAME` | `insured` |

### Deploy

1. Click **Create environment**
2. Wait for deployment (~5-10 minutes)
3. Once ready, you'll see a green health status

---

## Step 8: Testing the Application

### Verify Deployment

1. Get your EB URL from the console: `http://<env-name>.<region>.elasticbeanstalk.com`

2. Test endpoints:

```bash
# Home page
curl http://<EB-URL>/

# Claim form page
curl http://<EB-URL>/claim
```

### Test Claim Submission

```bash
# Submit a claim (if you have a POST API endpoint)
curl -X POST http://<EB-URL>/claim \
  -d "policy_id=P1001&name=John%20Doe&dob=1990-01-15&mobile=9876543210"
```

---

## Additional Configuration

### EB Extensions (.ebextensions)

Create `.ebextensions/01_flask.config` for custom configurations:

```yaml
option_settings:
  aws:elasticbeanstalk:container:python:
    WSGIPath: application:app

  aws:elasticbeanstalk:application:environment:
    DB_HOST: "your-rds-endpoint"
    DB_USER: "admin"
    DB_PASSWORD: "your-password"
    DB_NAME: "insured"
```

### Enable HTTPS

1. Request SSL certificate in AWS Certificate Manager
2. In EB → Configuration → Load Balancer:
   - Add HTTPS listener (port 443)
   - Select your certificate
3. Optionally redirect HTTP to HTTPS

### Using AWS Secrets Manager (Recommended)

Instead of hardcoding credentials, use AWS Secrets Manager:

1. Create a secret in Secrets Manager with your DB credentials
2. Add IAM permissions to the EB instance role
3. Modify `application.py` to fetch secrets using `boto3`

```python
import boto3
import json

def get_db_credentials():
    client = boto3.client('secretsmanager', region_name='us-east-1')
    response = client.get_secret_value(SecretId='your-secret-name')
    return json.loads(response['SecretString'])
```

---

## Troubleshooting

### Common Issues

| Problem | Cause | Solution |
|---------|-------|----------|
| 502 Bad Gateway | WSGI path incorrect | Verify `WSGIPath: application:app` in `.ebextensions` |
| Database connection timeout | Security group misconfigured | Ensure SG-RDS allows inbound from SG-EC2 on port 3306 |
| Environment variables not working | Not set in EB console | Check Configuration → Software → Environment properties |
| Application not starting | Missing dependencies | Verify `requirements.txt` includes all packages |

### Viewing Logs

```bash
# Using EB CLI
eb logs

# Or in AWS Console
Elastic Beanstalk → Environment → Logs → Request Logs
```

### SSH into EB Instance

```bash
# Using EB CLI
eb ssh

# Or manually
ssh -i your-key.pem ec2-user@<instance-public-ip>
```

---

## Clean Up Resources

To avoid charges, delete resources in this order:

1. **Elastic Beanstalk** environment and application
2. **RDS** instance (take a snapshot if needed)
3. **Bastion EC2** instance (if created)
4. **Security Groups** (delete in order: SG-RDS, SG-EC2, SG-ALB)
5. **DB Subnet Group**
6. **VPC** (this will delete subnets, route tables, IGW)

---

## References

- [AWS Elastic Beanstalk Developer Guide](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/)
- [Amazon RDS User Guide](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/)
- [Flask Documentation](https://flask.palletsprojects.com/)
- [PyMySQL Documentation](https://pymysql.readthedocs.io/)
