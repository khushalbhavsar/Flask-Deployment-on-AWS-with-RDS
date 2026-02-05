# AWS Architecture Documentation

## System Architecture Diagram

```mermaid
flowchart TB
    subgraph Internet["🌐 Internet"]
        Users["👥 Users"]
    end

    subgraph AWS["☁️ AWS Cloud"]
        IGW["🚪 Internet Gateway"]
        
        subgraph VPC["🔒 VPC (10.0.0.0/16)"]
            subgraph PublicSubnets["📤 Public Subnets"]
                subgraph PubA["Public Subnet A<br/>10.0.1.0/24 | AZ-1"]
                    EC2_A["🖥️ EC2 Instance<br/>Flask App"]
                end
                subgraph PubB["Public Subnet B<br/>10.0.2.0/24 | AZ-2"]
                    EC2_B["🖥️ EC2 Instance<br/>Flask App"]
                end
            end
            
            ALB["⚖️ Application Load Balancer"]
            
            subgraph PrivateSubnets["🔐 Private Subnets"]
                subgraph PriA["Private Subnet A<br/>10.0.3.0/24 | AZ-1"]
                    RDS_Primary["🗄️ RDS MySQL<br/>Primary"]
                end
                subgraph PriB["Private Subnet B<br/>10.0.4.0/24 | AZ-2"]
                    RDS_Standby["🗄️ RDS MySQL<br/>Standby (Multi-AZ)"]
                end
            end
            
            subgraph SecurityGroups["🛡️ Security Groups"]
                SG_ALB["SG-ALB<br/>Port 80, 443"]
                SG_EC2["SG-EC2<br/>Port 80 from ALB"]
                SG_RDS["SG-RDS<br/>Port 3306 from EC2"]
            end
        end
        
        subgraph EB["🚀 Elastic Beanstalk"]
            AutoScaling["📈 Auto Scaling Group"]
            HealthCheck["❤️ Health Monitoring"]
        end
    end

    Users -->|HTTPS| IGW
    IGW --> ALB
    ALB --> EC2_A
    ALB --> EC2_B
    EC2_A -->|Port 3306| RDS_Primary
    EC2_B -->|Port 3306| RDS_Primary
    RDS_Primary -.->|Sync| RDS_Standby
    
    EB -.-> EC2_A
    EB -.-> EC2_B
    
    SG_ALB -.-> ALB
    SG_EC2 -.-> EC2_A
    SG_EC2 -.-> EC2_B
    SG_RDS -.-> RDS_Primary
    SG_RDS -.-> RDS_Standby
```

---

## Data Flow Diagram

```mermaid
sequenceDiagram
    participant User
    participant ALB as Application Load Balancer
    participant EC2 as Flask App (EC2)
    participant RDS as MySQL RDS

    User->>ALB: 1. HTTPS Request
    ALB->>EC2: 2. Route to healthy instance
    EC2->>RDS: 3. Database query (Port 3306)
    RDS-->>EC2: 4. Query results
    EC2-->>ALB: 5. HTTP Response
    ALB-->>User: 6. HTTPS Response
```

---

## Network Architecture

```mermaid
graph TB
    subgraph VPC["VPC: 10.0.0.0/16"]
        subgraph AZ1["Availability Zone 1"]
            PublicA["Public Subnet A<br/>10.0.1.0/24"]
            PrivateA["Private Subnet A<br/>10.0.3.0/24"]
        end
        
        subgraph AZ2["Availability Zone 2"]
            PublicB["Public Subnet B<br/>10.0.2.0/24"]
            PrivateB["Private Subnet B<br/>10.0.4.0/24"]
        end
    end
    
    IGW[Internet Gateway] --> PublicA
    IGW --> PublicB
    
    PublicA -.->|Internal| PrivateA
    PublicB -.->|Internal| PrivateB
```

---

## Component Details

### 1. VPC (Virtual Private Cloud)

| Property | Value |
|----------|-------|
| CIDR Block | `10.0.0.0/16` |
| DNS Hostnames | Enabled |
| DNS Resolution | Enabled |

### 2. Subnets

| Subnet | CIDR | Type | AZ | Purpose |
|--------|------|------|-----|---------|
| Public-A | `10.0.1.0/24` | Public | AZ-1 | EC2 instances, ALB |
| Public-B | `10.0.2.0/24` | Public | AZ-2 | EC2 instances, ALB |
| Private-A | `10.0.3.0/24` | Private | AZ-1 | RDS Primary |
| Private-B | `10.0.4.0/24` | Private | AZ-2 | RDS Standby |

### 3. Security Groups

```mermaid
flowchart LR
    subgraph SG_ALB["SG-ALB"]
        ALB_In["Inbound:<br/>80, 443 from 0.0.0.0/0"]
        ALB_Out["Outbound:<br/>All traffic"]
    end
    
    subgraph SG_EC2["SG-EC2"]
        EC2_In["Inbound:<br/>80 from SG-ALB<br/>22 from Admin IP"]
        EC2_Out["Outbound:<br/>All traffic"]
    end
    
    subgraph SG_RDS["SG-RDS"]
        RDS_In["Inbound:<br/>3306 from SG-EC2"]
        RDS_Out["Outbound:<br/>None required"]
    end
    
    SG_ALB --> SG_EC2 --> SG_RDS
```

### 4. Route Tables

#### Public Route Table
| Destination | Target |
|-------------|--------|
| `10.0.0.0/16` | local |
| `0.0.0.0/0` | Internet Gateway |

#### Private Route Table
| Destination | Target |
|-------------|--------|
| `10.0.0.0/16` | local |

---

## Application Architecture

```mermaid
flowchart TB
    subgraph FlaskApp["Flask Application"]
        Routes["Routes<br/>/  → index<br/>/claim → claim form"]
        DB["Database Layer<br/>PyMySQL"]
        Templates["Templates<br/>Jinja2"]
        Static["Static Files<br/>CSS, Images"]
    end
    
    Routes --> Templates
    Routes --> DB
    Templates --> Static
    
    subgraph Database["RDS MySQL"]
        Claims["claims table<br/>policy_id, name, dob, mobile"]
    end
    
    DB --> Claims
```

---

## Deployment Pipeline

```mermaid
flowchart LR
    Code["📝 Code<br/>application.py"] --> Zip["📦 ZIP Bundle<br/>+ requirements.txt"]
    Zip --> EB["🚀 Elastic Beanstalk<br/>Upload & Deploy"]
    EB --> EC2["🖥️ EC2 Instances<br/>Auto-provisioned"]
    EC2 --> Live["✅ Live Application"]
```

---

## High Availability Design

```mermaid
flowchart TB
    subgraph HA["High Availability Features"]
        direction TB
        
        subgraph Compute["Compute Layer"]
            ASG["Auto Scaling Group<br/>Min: 1, Max: 4"]
            MultiAZ_EC2["Multi-AZ EC2<br/>Instances"]
        end
        
        subgraph Data["Data Layer"]
            MultiAZ_RDS["Multi-AZ RDS<br/>Automatic Failover"]
            Backups["Automated Backups<br/>Point-in-time Recovery"]
        end
        
        subgraph Network["Network Layer"]
            ALB_HA["ALB Health Checks<br/>Automatic Routing"]
            DNS["Route 53 (Optional)<br/>DNS Failover"]
        end
    end
```

---

## Security Architecture

```mermaid
flowchart TB
    subgraph SecurityLayers["Defense in Depth"]
        L1["Layer 1: Network<br/>VPC Isolation"]
        L2["Layer 2: Subnet<br/>Public/Private Separation"]
        L3["Layer 3: Security Groups<br/>Port-level Firewall"]
        L4["Layer 4: Application<br/>Input Validation"]
        L5["Layer 5: Data<br/>Encrypted at Rest"]
    end
    
    L1 --> L2 --> L3 --> L4 --> L5
```

### Security Controls

| Layer | Control | Implementation |
|-------|---------|----------------|
| **Network** | VPC Isolation | Custom VPC with no default rules |
| **Subnet** | Public/Private | RDS in private subnets (no internet access) |
| **Firewall** | Security Groups | Least privilege access between tiers |
| **Transport** | HTTPS | SSL/TLS termination at ALB |
| **Database** | Encryption | RDS encryption at rest |
| **Access** | IAM | Role-based access control |

---

## Cost Optimization

| Component | Cost Factor | Optimization Strategy |
|-----------|-------------|----------------------|
| EC2 | Compute hours | Use t3.micro (free tier) or right-size |
| RDS | Instance + storage | db.t3.micro, single-AZ for dev |
| ALB | LCU hours | Included with Elastic Beanstalk |
| Data Transfer | Outbound data | Use CloudFront for static assets |

---

## Scaling Strategy

```mermaid
flowchart LR
    subgraph Triggers["Scaling Triggers"]
        CPU["CPU > 70%"]
        Requests["Request Count"]
        Latency["Response Time"]
    end
    
    subgraph Actions["Scaling Actions"]
        ScaleOut["Scale Out<br/>Add Instances"]
        ScaleIn["Scale In<br/>Remove Instances"]
    end
    
    CPU --> ScaleOut
    Requests --> ScaleOut
    Latency --> ScaleOut
    
    CPU --> ScaleIn
```

---

## Environment Variables

```mermaid
flowchart LR
    subgraph EB["Elastic Beanstalk Environment"]
        ENV["Environment Variables"]
    end
    
    subgraph App["Flask Application"]
        DB_HOST["DB_HOST"]
        DB_USER["DB_USER"]
        DB_PASSWORD["DB_PASSWORD"]
        DB_NAME["DB_NAME"]
    end
    
    ENV --> DB_HOST
    ENV --> DB_USER
    ENV --> DB_PASSWORD
    ENV --> DB_NAME
```

| Variable | Description | Source |
|----------|-------------|--------|
| `DB_HOST` | RDS endpoint | AWS RDS console |
| `DB_USER` | Database username | RDS configuration |
| `DB_PASSWORD` | Database password | Secrets/secure config |
| `DB_NAME` | Database name | RDS configuration |

---

## Monitoring & Logging

```mermaid
flowchart TB
    subgraph Sources["Log Sources"]
        EC2_Logs["EC2 Instance Logs"]
        ALB_Logs["ALB Access Logs"]
        RDS_Logs["RDS Logs"]
        App_Logs["Application Logs"]
    end
    
    subgraph CloudWatch["Amazon CloudWatch"]
        Metrics["Metrics"]
        Alarms["Alarms"]
        Logs["Log Groups"]
    end
    
    EC2_Logs --> Logs
    ALB_Logs --> Logs
    RDS_Logs --> Logs
    App_Logs --> Logs
    
    Metrics --> Alarms
```

### Key Metrics to Monitor

| Metric | Threshold | Action |
|--------|-----------|--------|
| CPU Utilization | > 70% | Scale out |
| Database Connections | > 80% of max | Investigate |
| Request Latency | > 2s | Investigate |
| Error Rate | > 5% | Alert |
| RDS Storage | > 80% | Increase storage |
