# Infrastructure Design

**เวอร์ชัน:** 1.0  
**วันที่:** 22 มิถุนายน 2026  
**สถานะ:** Draft  
**เอกสารที่เกี่ยวข้อง:** `03-Software-Design-Specification.md`, `02-System-Architecture-Document.md`

---

## 📋 สารบัญ

1. [Cloud Provider](#1-cloud-provider)
2. [Network Topology](#2-network-topology)
3. [VPC](#3-vpc)
4. [Subnets](#4-subnets)
5. [Load Balancer](#5-load-balancer)
6. [CDN](#6-cdn)
7. [Object Storage](#7-object-storage)
8. [Container Registry](#8-container-registry)
9. [Kubernetes Cluster](#9-kubernetes-cluster)
10. [Secrets Management](#10-secrets-management)
11. [CI/CD](#11-cicd)
12. [Monitoring](#12-monitoring)
13. [Backup Strategy](#13-backup-strategy)
14. [Cost Estimation](#14-cost-estimation)

---

## 1. Cloud Provider

### 1.1 Provider Selection

**Selected Provider: Amazon Web Services (AWS)**

| Criteria | AWS | Azure | GCP | Decision Factor |
|----------|-----|-------|-----|-----------------|
| Market Share | 32% | 23% | 11% | ✅ Ecosystem maturity |
| Kubernetes Support | EKS (Excellent) | AKS (Good) | GKE (Excellent) | ✅ EKS with Fargate |
| Managed Kafka | MSK (Excellent) | Event Hubs (Good) | Pub/Sub (Good) | ✅ MSK for Kafka |
| Managed Redis | ElastiCache (Excellent) | Cache for Redis (Good) | Memorystore (Good) | ✅ ElastiCache |
| Managed PostgreSQL | Aurora/RDS (Excellent) | Flexible Server (Good) | Cloud SQL (Good) | ✅ Aurora PostgreSQL |
| AI/ML Services | SageMaker (Excellent) | Azure ML (Good) | Vertex AI (Good) | ✅ SageMaker |
| Global Presence | 33 Regions | 60+ Regions | 36 Regions | ✅ APAC coverage |
| Compliance | SOC2, ISO27001, GDPR | SOC2, ISO27001, GDPR | SOC2, ISO27001, GDPR | ✅ All compliant |
| Cost | Medium | Medium | Low-Medium | ⚠️ Competitive |

### 1.2 Multi-Cloud Strategy

```
┌─────────────────────────────────────────────────────────────────┐
│                    MULTI-CLOUD STRATEGY                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  PRIMARY: AWS (Production)                                       │
│  ├─ Region: ap-southeast-1 (Singapore)                          │
│  ├─ Availability Zones: 3 AZs                                   │
│  └─ Services: EKS, RDS, ElastiCache, MSK, S3                   │
│                                                                  │
│  DISASTER RECOVERY: AWS (Secondary Region)                       │
│  ├─ Region: ap-northeast-1 (Tokyo)                              │
│  ├─ Strategy: Pilot Light                                       │
│  └─ RTO: 4 hours, RPO: 1 hour                                   │
│                                                                  │
│  DEVELOPMENT/STAGING: AWS                                        │
│  ├─ Region: ap-southeast-1                                      │
│  └─ Separate AWS Account (Dev Account)                          │
│                                                                  │
│  FUTURE CONSIDERATION: GCP (AI/ML Workloads)                     │
│  ├─ Use Case: Vertex AI for advanced ML                         │
│  └─ Integration: Via MuleSoft or Direct API                     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 1.3 AWS Account Structure

```
AWS Organization
├── Management Account (111111111111)
│   └── AWS Organizations, Billing, CloudTrail
│
├── Security Account (222222222222)
│   ├── GuardDuty
│   ├── Security Hub
│   ├── IAM Identity Center
│   └── KMS Keys
│
├── Shared Services Account (333333333333)
│   ├── ECR (Container Registry)
│   ├── CodePipeline
│   ├── CodeBuild
│   └── Shared VPC Endpoints
│
├── Production Account (444444444444)
│   ├── EKS Cluster
│   ├── RDS Aurora
│   ├── ElastiCache
│   ├── MSK
│   ├── S3 Buckets
│   └── CloudFront
│
├── Staging Account (555555555555)
│   └── Mirror of Production (smaller scale)
│
└── Development Account (666666666666)
    └── Developer environments
```

### 1.4 Infrastructure as Code (IaC)

```yaml
# Tool Selection
Primary IaC: Terraform (OpenTofu)
  Version: 1.7+
  State Backend: S3 + DynamoDB (locking)
  Modules: Custom modules in private registry
  
Secondary IaC: AWS CDK (TypeScript)
  Use Case: Complex EKS configurations
  Language: TypeScript
  
Configuration Management: Ansible
  Use Case: EC2 instance configuration
  Playbooks: Stored in Git

Policy as Code: OPA (Open Policy Agent)
  Use Case: Kubernetes policies, IAM policies
  
GitOps: ArgoCD
  Use Case: Kubernetes application deployment
  Repository: Git-based manifests
```

---

## 2. Network Topology

### 2.1 High-Level Network Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    INTERNET                                       │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
─────────────────────────────────────────────────────────────────┐
│              AWS GLOBAL ACCELERATOR (Optional)                   │
│         - Static IP addresses                                   │
│         - Edge locations worldwide                              │
│         - Health checks and failover                            │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    CLOUDFRONT CDN                                │
│         - Static assets (React frontend)                        │
│         - API caching                                           │
│         - DDoS protection (AWS Shield)                          │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│              WEB APPLICATION FIREWALL (WAF)                      │
│         - OWASP Top 10 protection                               │
│         - Rate limiting                                         │
│         - IP reputation lists                                   │
│         - Geo-blocking                                          │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                  PUBLIC SUBNET (DMZ)                             │
│  ┌──────────────────┐  ┌──────────────────┐                    │
│  │   ALB (Internet) │  │   NAT Gateway    │                    │
│  │   - React App    │  │   - Outbound     │                    │
│  │   - API Gateway  │  │     Traffic      │                    │
│  └──────────────────┘  └──────────────────┘                    │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│               PRIVATE SUBNET (Application)                       │
│  ┌──────────────────────────────────────────────────────────  │
│  │              EKS CLUSTER (Kubernetes)                     │  │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐  │  │
│  │  │ NestJS   │ │   Go     │ │  Python  │ │  Redis   │  │  │
│  │  │ Services │ │ Services │ │ Services │ │  Proxy   │  │  │
│  │  └──────────┘ └──────────┘ └──────────┘ └──────────┘  │  │
│  ──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│              ISOLATED SUBNET (Data Layer)                        │
│  ┌──────────┐  ┌──────────┐  ──────────┐  ┌──────────┐      │
│  │  Aurora  │  │  MSK     │  │  Elastic │  │   S3     │      │
│  │PostgreSQL│  │ (Kafka)  │  │  Cache   │  │ Endpoints│      │
│  └──────────┘  └──────────┘  └──────────┘  └──────────      │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 Network Security Groups

```hcl
# Security Group: ALB (Public)
resource "aws_security_group" "alb" {
  name        = "revenue-mgmt-alb-sg"
  description = "Security group for Application Load Balancer"
  vpc_id      = aws_vpc.main.id

  ingress {
    description = "HTTPS from Internet"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    description = "HTTP from Internet (redirect to HTTPS)"
    from_port   = 80
    to_port     = 80
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

# Security Group: EKS Nodes (Private)
resource "aws_security_group" "eks_nodes" {
  name        = "revenue-mgmt-eks-nodes-sg"
  description = "Security group for EKS worker nodes"
  vpc_id      = aws_vpc.main.id

  # Allow traffic from ALB
  ingress {
    description = "From ALB"
    from_port   = 30000
    to_port     = 32768
    protocol    = "tcp"
    security_groups = [aws_security_group.alb.id]
  }

  # Allow inter-node communication
  ingress {
    description = "From EKS nodes"
    from_port   = 0
    to_port     = 65535
    protocol    = "tcp"
    self        = true
  }

  # Allow SSH from bastion (if needed)
  ingress {
    description = "SSH from Bastion"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    security_groups = [aws_security_group.bastion.id]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# Security Group: Aurora PostgreSQL (Isolated)
resource "aws_security_group" "aurora" {
  name        = "revenue-mgmt-aurora-sg"
  description = "Security group for Aurora PostgreSQL"
  vpc_id      = aws_vpc.main.id

  ingress {
    description = "PostgreSQL from EKS"
    from_port   = 5432
    to_port     = 5432
    protocol    = "tcp"
    security_groups = [aws_security_group.eks_nodes.id]
  }

  # No egress needed (database doesn't initiate connections)
}

# Security Group: ElastiCache Redis (Isolated)
resource "aws_security_group" "redis" {
  name        = "revenue-mgmt-redis-sg"
  description = "Security group for ElastiCache Redis"
  vpc_id      = aws_vpc.main.id

  ingress {
    description = "Redis from EKS"
    from_port   = 6379
    to_port     = 6379
    protocol    = "tcp"
    security_groups = [aws_security_group.eks_nodes.id]
  }
}

# Security Group: MSK Kafka (Isolated)
resource "aws_security_group" "msk" {
  name        = "revenue-mgmt-msk-sg"
  description = "Security group for MSK Kafka"
  vpc_id      = aws_vpc.main.id

  ingress {
    description = "Kafka from EKS"
    from_port   = 9098
    to_port     = 9098
    protocol    = "tcp"
    security_groups = [aws_security_group.eks_nodes.id]
  }

  ingress {
    description = "Kafka inter-broker"
    from_port   = 9094
    to_port     = 9094
    protocol    = "tcp"
    self        = true
  }
}
```

### 2.3 VPC Endpoints (PrivateLink)

```hcl
# S3 Gateway Endpoint (Free)
resource "aws_vpc_endpoint" "s3" {
  vpc_id       = aws_vpc.main.id
  service_name = "com.amazonaws.ap-southeast-1.s3"
  route_table_ids = [
    aws_route_table.private_app.id,
    aws_route_table.private_data.id,
  ]
}

# ECR Interface Endpoint
resource "aws_vpc_endpoint" "ecr" {
  vpc_id              = aws_vpc.main.id
  service_name        = "com.amazonaws.ap-southeast-1.ecr.api"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = aws_subnet.private_app[*].id
  security_group_ids  = [aws_security_group.vpc_endpoints.id]
  private_dns_enabled = true
}

# ECR DKR Interface Endpoint
resource "aws_vpc_endpoint" "ecr_dkr" {
  vpc_id              = aws_vpc.main.id
  service_name        = "com.amazonaws.ap-southeast-1.ecr.dkr"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = aws_subnet.private_app[*].id
  security_group_ids  = [aws_security_group.vpc_endpoints.id]
  private_dns_enabled = true
}

# Secrets Manager Interface Endpoint
resource "aws_vpc_endpoint" "secrets_manager" {
  vpc_id              = aws_vpc.main.id
  service_name        = "com.amazonaws.ap-southeast-1.secretsmanager"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = aws_subnet.private_app[*].id
  security_group_ids  = [aws_security_group.vpc_endpoints.id]
  private_dns_enabled = true
}

# CloudWatch Logs Interface Endpoint
resource "aws_vpc_endpoint" "cloudwatch_logs" {
  vpc_id              = aws_vpc.main.id
  service_name        = "com.amazonaws.ap-southeast-1.logs"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = aws_subnet.private_app[*].id
  security_group_ids  = [aws_security_group.vpc_endpoints.id]
  private_dns_enabled = true
}
```

---

## 3. VPC

### 3.1 VPC Configuration

```hcl
# Main VPC
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name        = "revenue-mgmt-vpc"
    Environment = "production"
    Project     = "revenue-management"
    ManagedBy   = "terraform"
  }
}

# VPC Flow Logs
resource "aws_flow_log" "main" {
  iam_role_arn    = aws_iam_role.vpc_flow_log.arn
  log_destination = aws_cloudwatch_log_group.vpc_flow_log.arn
  traffic_type    = "ALL"
  vpc_id          = aws_vpc.main.id

  tags = {
    Name = "revenue-mgmt-vpc-flow-log"
  }
}

# Internet Gateway
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "revenue-mgmt-igw"
  }
}

# NAT Gateways (One per AZ for high availability)
resource "aws_eip" "nat" {
  count  = 3
  domain = "vpc"

  tags = {
    Name = "revenue-mgmt-nat-eip-${count.index + 1}"
  }
}

resource "aws_nat_gateway" "main" {
  count         = 3
  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = aws_subnet.public[count.index].id

  tags = {
    Name = "revenue-mgmt-nat-${count.index + 1}"
  }

  depends_on = [aws_internet_gateway.main]
}
```

### 3.2 VPC CIDR Allocation

```
VPC CIDR: 10.0.0.0/16 (65,536 IPs)

─────────────────────────────────────────────────────────────────┐
│  CIDR Block          │  Purpose              │  IPs Available  │
├─────────────────────────────────────────────────────────────────┤
│  10.0.0.0/24         │  Reserved             │  256            │
│  10.0.1.0/24         │  Public Subnet AZ-1   │  256            │
│  10.0.2.0/24         │  Public Subnet AZ-2   │  256            │
│  10.0.3.0/24         │  Public Subnet AZ-3   │  256            │
│  10.0.10.0/24        │  Private App AZ-1     │  256            │
│  10.0.20.0/24        │  Private App AZ-2     │  256            │
│  10.0.30.0/24        │  Private App AZ-3     │  256            │
│  10.0.40.0/24        │  Reserved (Future)    │  256            │
│  10.0.50.0/24        │  Reserved (Future)    │  256            │
│  10.0.100.0/24       │  Private Data AZ-1    │  256            │
│  10.0.200.0/24       │  Private Data AZ-2    │  256            │
│  10.0.300.0/24       │  Private Data AZ-3    │  256            │
│  10.0.110.0/24       │  Isolated DB AZ-1     │  256            │
│  10.0.210.0/24       │  Isolated DB AZ-2     │  256            │
│  10.0.310.0/24       │  Isolated DB AZ-3     │  256            │
│  10.0.120.0/24       │  Reserved (Future)    │  256            │
│  10.0.220.0/24       │  Reserved (Future)    │  256            │
│  10.0.320.0/24       │  Reserved (Future)    │  256            │
│  10.0.200.0/20       │  EKS Pod CIDR         │  4,096          │
│  10.0.220.0/22       │  EKS Service CIDR     │  1,024          │
─────────────────────────────────────────────────────────────────┘

Total Allocated: ~7,168 IPs (11% of VPC)
Reserved for Growth: ~58,368 IPs (89% of VPC)
```

### 3.3 Route Tables

```hcl
# Public Route Table
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }

  tags = {
    Name = "revenue-mgmt-public-rt"
  }
}

# Private App Route Table (via NAT)
resource "aws_route_table" "private_app" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.main[0].id
  }

  tags = {
    Name = "revenue-mgmt-private-app-rt"
  }
}

# Private Data Route Table (via NAT, restricted)
resource "aws_route_table" "private_data" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.main[0].id
  }

  # No route to IGW (isolated from internet)

  tags = {
    Name = "revenue-mgmt-private-data-rt"
  }
}

# Route Table Associations
resource "aws_route_table_association" "public" {
  count          = 3
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

resource "aws_route_table_association" "private_app" {
  count          = 3
  subnet_id      = aws_subnet.private_app[count.index].id
  route_table_id = aws_route_table.private_app.id
}

resource "aws_route_table_association" "private_data" {
  count          = 3
  subnet_id      = aws_subnet.private_data[count.index].id
  route_table_id = aws_route_table.private_data.id
}
```

---

## 4. Subnets

### 4.1 Subnet Configuration

```hcl
# Availability Zones
data "aws_availability_zones" "available" {
  state = "available"
}

# Public Subnets (ALB, NAT Gateway)
resource "aws_subnet" "public" {
  count                   = 3
  vpc_id                  = aws_vpc.main.id
  cidr_block              = cidrsubnet(aws_vpc.main.cidr_block, 8, count.index + 1)
  availability_zone       = data.aws_availability_zones.available.names[count.index]
  map_public_ip_on_launch = true

  tags = {
    Name                                            = "revenue-mgmt-public-${count.index + 1}"
    "kubernetes.io/role/elb"                        = "1"
    "kubernetes.io/cluster/revenue-mgmt-eks"        = "shared"
  }
}

# Private Application Subnets (EKS Nodes)
resource "aws_subnet" "private_app" {
  count             = 3
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(aws_vpc.main.cidr_block, 8, count.index + 10)
  availability_zone = data.aws_availability_zones.available.names[count.index]

  tags = {
    Name                                            = "revenue-mgmt-private-app-${count.index + 1}"
    "kubernetes.io/role/internal-elb"               = "1"
    "kubernetes.io/cluster/revenue-mgmt-eks"        = "shared"
  }
}

# Private Data Subnets (ElastiCache, MSK)
resource "aws_subnet" "private_data" {
  count             = 3
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(aws_vpc.main.cidr_block, 8, count.index + 100)
  availability_zone = data.aws_availability_zones.available.names[count.index]

  tags = {
    Name = "revenue-mgmt-private-data-${count.index + 1}"
  }
}

# Isolated Database Subnets (Aurora)
resource "aws_subnet" "isolated_db" {
  count             = 3
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(aws_vpc.main.cidr_block, 8, count.index + 110)
  availability_zone = data.aws_availability_zones.available.names[count.index]

  tags = {
    Name = "revenue-mgmt-isolated-db-${count.index + 1}"
  }
}
```

### 4.2 Subnet Usage Matrix

| Subnet Type | CIDR | AZ-1 | AZ-2 | AZ-3 | Purpose | Services |
|-------------|------|------|------|------|---------|----------|
| **Public** | 10.0.1-3.0/24 | ✅ | ✅ | ✅ | Internet-facing | ALB, NAT Gateway |
| **Private App** | 10.0.10-30.0/24 | ✅ | ✅ | ✅ | Application | EKS Nodes, Bastion |
| **Private Data** | 10.0.100-300.0/24 | ✅ | ✅ | ✅ | Data Services | ElastiCache, MSK |
| **Isolated DB** | 10.0.110-310.0/24 | ✅ | ✅ | ✅ | Database | Aurora PostgreSQL |

### 4.3 Network ACLs

```hcl
# Network ACL: Public Subnet
resource "aws_network_acl" "public" {
  vpc_id     = aws_vpc.main.id
  subnet_ids = aws_subnet.public[*].id

  # Inbound Rules
  ingress {
    rule_no    = 100
    protocol   = "tcp"
    rule_action = "allow"
    cidr_block = "0.0.0.0/0"
    from_port  = 443
    to_port    = 443
  }

  ingress {
    rule_no    = 110
    protocol   = "tcp"
    rule_action = "allow"
    cidr_block = "0.0.0.0/0"
    from_port  = 80
    to_port    = 80
  }

  ingress {
    rule_no    = 120
    protocol   = "tcp"
    rule_action = "allow"
    cidr_block = "0.0.0.0/0"
    from_port  = 1024
    to_port    = 65535
  }

  # Outbound Rules
  egress {
    rule_no    = 100
    protocol   = "-1"
    rule_action = "allow"
    cidr_block = "0.0.0.0/0"
    from_port  = 0
    to_port    = 0
  }

  tags = {
    Name = "revenue-mgmt-public-nacl"
  }
}

# Network ACL: Private Subnet (Restrictive)
resource "aws_network_acl" "private" {
  vpc_id     = aws_vpc.main.id
  subnet_ids = concat(
    aws_subnet.private_app[*].id,
    aws_subnet.private_data[*].id,
    aws_subnet.isolated_db[*].id
  )

  # Inbound Rules
  ingress {
    rule_no    = 100
    protocol   = "tcp"
    rule_action = "allow"
    cidr_block = "10.0.0.0/16"  # VPC internal only
    from_port  = 0
    to_port    = 65535
  }

  ingress {
    rule_no    = 110
    protocol   = "tcp"
    rule_action = "allow"
    cidr_block = "10.0.0.0/16"
    from_port  = 1024
    to_port    = 65535
  }

  # Outbound Rules
  egress {
    rule_no    = 100
    protocol   = "tcp"
    rule_action = "allow"
    cidr_block = "0.0.0.0/0"
    from_port  = 443
    to_port    = 443
  }

  egress {
    rule_no    = 110
    protocol   = "tcp"
    rule_action = "allow"
    cidr_block = "0.0.0.0/0"
    from_port  = 80
    to_port    = 80
  }

  egress {
    rule_no    = 120
    protocol   = "tcp"
    rule_action = "allow"
    cidr_block = "10.0.0.0/16"
    from_port  = 0
    to_port    = 65535
  }

  tags = {
    Name = "revenue-mgmt-private-nacl"
  }
}
```

---

## 5. Load Balancer

### 5.1 Load Balancer Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    LOAD BALANCER LAYERS                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  LAYER 1: CloudFront (Global CDN)                                │
│  ├─ Purpose: Static assets, API caching, DDoS protection        │
│  ├─ Type: CDN                                                    │
│  └─ SSL: AWS Certificate Manager (ACM)                          │
│                                                                  │
│  LAYER 2: Application Load Balancer (ALB)                        │
│  ├─ Purpose: HTTP/HTTPS traffic routing                         │
│  ├─ Type: Layer 7 (Application)                                  │
│  ├─ Target Groups:                                               │
│  │  ├─ React Frontend (S3/CloudFront origin)                    │
│  │  ├─ NestJS API Gateway (EKS)                                 │
│  │  └─ Health Check Endpoints                                   │
│  └─ Features:                                                    │
│     ├─ SSL termination                                           │
│     ├─ WAF integration                                           │
│     ├─ Sticky sessions (if needed)                               │
│     └─ Path-based routing                                        │
│                                                                  │
│  LAYER 3: Network Load Balancer (NLB)                            │
│  ├─ Purpose: High-performance, low-latency traffic              │
│  ├─ Type: Layer 4 (Transport)                                    │
│  ├─ Target Groups:                                               │
│  │  ├─ Go Services (gRPC)                                       │
│  │  ├─ High-throughput APIs                                     │
│  │  └─ WebSocket connections                                    │
│  └─ Features:                                                    │
│     ├─ Static IP addresses                                       │
│     ├─ TLS termination                                           │
│     ├─ Connection draining                                       │
│     └─ Cross-zone load balancing                                 │
│                                                                  │
│  LAYER 4: Internal Load Balancer                                 │
│  ├─ Purpose: Inter-service communication                        │
│  ├─ Type: Internal ALB/NLB                                       │
│  ├─ Target Groups:                                               │
│  │  ├─ Service Mesh (Istio/Linkerd)                             │
│  │  └─ Direct service-to-service                                │
│  └─ Features:                                                    │
│     ├─ VPC-only access                                           │
│     ├─ mTLS (via service mesh)                                   │
│     └─ No internet exposure                                      │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 5.2 ALB Configuration

```hcl
# Application Load Balancer
resource "aws_lb" "main" {
  name               = "revenue-mgmt-alb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb.id]
  subnets            = aws_subnet.public[*].id

  enable_deletion_protection = true
  enable_http2               = true
  enable_cross_zone_load_balancing = true

  access_logs {
    bucket  = aws_s3_bucket.alb_logs.id
    prefix  = "alb-logs"
    enabled = true
  }

  tags = {
    Name        = "revenue-mgmt-alb"
    Environment = "production"
  }
}

# HTTPS Listener
resource "aws_lb_listener" "https" {
  load_balancer_arn = aws_lb.main.arn
  port              = 443
  protocol          = "HTTPS"
  ssl_policy        = "ELBSecurityPolicy-TLS13-1-2-2021-06"
  certificate_arn   = aws_acm_certificate.main.arn

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.api.arn
  }
}

# HTTP to HTTPS Redirect
resource "aws_lb_listener" "http" {
  load_balancer_arn = aws_lb.main.arn
  port              = 80
  protocol          = "HTTP"

  default_action {
    type = "redirect"

    redirect {
      port        = "443"
      protocol    = "HTTPS"
      status_code = "HTTP_301"
    }
  }
}

# Target Group: API (NestJS)
resource "aws_lb_target_group" "api" {
  name        = "revenue-mgmt-api"
  port        = 3000
  protocol    = "HTTP"
  vpc_id      = aws_vpc.main.id
  target_type = "ip"

  health_check {
    enabled             = true
    healthy_threshold   = 2
    interval            = 30
    matcher             = "200"
    path                = "/health"
    port                = "traffic-port"
    protocol            = "HTTP"
    timeout             = 5
    unhealthy_threshold = 3
  }

  stickiness {
    type    = "lb_cookie"
    enabled = false
  }

  deregistration_delay = 30

  tags = {
    Name = "revenue-mgmt-api-tg"
  }
}

# Target Group: React Frontend (if served from EKS)
resource "aws_lb_target_group" "frontend" {
  name        = "revenue-mgmt-frontend"
  port        = 80
  protocol    = "HTTP"
  vpc_id      = aws_vpc.main.id
  target_type = "ip"

  health_check {
    enabled             = true
    healthy_threshold   = 2
    interval            = 30
    matcher             = "200"
    path                = "/"
    port                = "traffic-port"
    protocol            = "HTTP"
    timeout             = 5
    unhealthy_threshold = 3
  }

  tags = {
    Name = "revenue-mgmt-frontend-tg"
  }
}

# Listener Rules: Path-based routing
resource "aws_lb_listener_rule" "api" {
  listener_arn = aws_lb_listener.https.arn
  priority     = 100

  action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.api.arn
  }

  condition {
    path_pattern {
      values = ["/api/*", "/graphql", "/health"]
    }
  }
}

resource "aws_lb_listener_rule" "frontend" {
  listener_arn = aws_lb_listener.https.arn
  priority     = 200

  action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.frontend.arn
  }

  condition {
    path_pattern {
      values = ["/*"]
    }
  }
}
```

### 5.3 NLB Configuration (for Go Services/gRPC)

```hcl
# Network Load Balancer (for gRPC services)
resource "aws_lb" "grpc" {
  name               = "revenue-mgmt-grpc-nlb"
  internal           = true
  load_balancer_type = "network"
  subnets            = aws_subnet.private_app[*].id

  enable_deletion_protection = true
  enable_cross_zone_load_balancing = true

  tags = {
    Name = "revenue-mgmt-grpc-nlb"
  }
}

# TLS Listener for gRPC
resource "aws_lb_listener" "grpc" {
  load_balancer_arn = aws_lb.grpc.arn
  port              = 9090
  protocol          = "TLS"
  ssl_policy        = "ELBSecurityPolicy-TLS13-1-2-2021-06"
  certificate_arn   = aws_acm_certificate.grpc.arn

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.grpc.arn
  }
}

# Target Group: gRPC Services
resource "aws_lb_target_group" "grpc" {
  name        = "revenue-mgmt-grpc"
  port        = 9090
  protocol    = "TLS"
  vpc_id      = aws_vpc.main.id
  target_type = "ip"

  health_check {
    enabled             = true
    healthy_threshold   = 2
    interval            = 30
    matcher             = "0"
    path                = "/grpc.health.v1.Health/Check"
    port                = "traffic-port"
    protocol            = "HTTPS"
    timeout             = 5
    unhealthy_threshold = 3
  }

  tags = {
    Name = "revenue-mgmt-grpc-tg"
  }
}
```

### 5.4 WAF Configuration

```hcl
# Web Application Firewall
resource "aws_wafv2_web_acl" "main" {
  name        = "revenue-mgmt-waf"
  scope       = "REGIONAL"
  description = "WAF for Revenue Management ALB"

  default_action {
    allow {}
  }

  # AWS Managed Rules
  rule {
    name     = "AWSManagedRulesCommonRuleSet"
    priority = 1

    override_action {
      none {}
    }

    statement {
      managed_rule_group_statement {
        name        = "AWSManagedRulesCommonRuleSet"
        vendor_name = "AWS"
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "AWSManagedRulesCommonRuleSet"
      sampled_requests_enabled   = true
    }
  }

  rule {
    name     = "AWSManagedRulesKnownBadInputsRuleSet"
    priority = 2

    override_action {
      none {}
    }

    statement {
      managed_rule_group_statement {
        name        = "AWSManagedRulesKnownBadInputsRuleSet"
        vendor_name = "AWS"
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "AWSManagedRulesKnownBadInputsRuleSet"
      sampled_requests_enabled   = true
    }
  }

  # Rate Limiting Rule
  rule {
    name     = "RateLimitRule"
    priority = 3

    action {
      block {}
    }

    statement {
      rate_based_statement {
        limit              = 2000
        aggregate_key_type = "IP"
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "RateLimitRule"
      sampled_requests_enabled   = true
    }
  }

  # Geo-blocking Rule (optional)
  rule {
    name     = "GeoBlockRule"
    priority = 4

    action {
      block {}
    }

    statement {
      geo_match_statement {
        country_codes = ["CN", "RU", "KP"]  # Block high-risk countries
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "GeoBlockRule"
      sampled_requests_enabled   = true
    }
  }

  visibility_config {
    cloudwatch_metrics_enabled = true
    metric_name                = "revenue-mgmt-waf"
    sampled_requests_enabled   = true
  }

  tags = {
    Name = "revenue-mgmt-waf"
  }
}

# Associate WAF with ALB
resource "aws_wafv2_web_acl_association" "alb" {
  resource_arn = aws_lb.main.arn
  web_acl_arn  = aws_wafv2_web_acl.main.arn
}
```

---

## 6. CDN

### 6.1 CloudFront Configuration

```hcl
# CloudFront Distribution
resource "aws_cloudfront_distribution" "main" {
  origin {
    domain_name = aws_s3_bucket.frontend.bucket_regional_domain_name
    origin_id   = "S3-frontend"

    s3_origin_config {
      origin_access_identity = aws_cloudfront_origin_access_identity.frontend.iam_arn
    }
  }

  origin {
    domain_name = aws_lb.main.dns_name
    origin_id   = "ALB-api"

    custom_origin_config {
      http_port              = 80
      https_port             = 443
      origin_protocol_policy = "https-only"
      origin_ssl_protocols   = ["TLSv1.2"]
    }
  }

  # Cache Behavior: Static Assets (S3)
  ordered_cache_behavior {
    path_pattern     = "/static/*"
    allowed_methods  = ["GET", "HEAD", "OPTIONS"]
    cached_methods   = ["GET", "HEAD"]
    target_origin_id = "S3-frontend"

    forwarded_values {
      query_string = false
      cookies {
        forward = "none"
      }
    }

    min_ttl     = 0
    default_ttl = 86400    # 24 hours
    max_ttl     = 31536000 # 1 year

    compress               = true
    viewer_protocol_policy = "redirect-to-https"
  }

  # Cache Behavior: API (ALB)
  ordered_cache_behavior {
    path_pattern     = "/api/*"
    allowed_methods  = ["DELETE", "GET", "HEAD", "OPTIONS", "PATCH", "POST", "PUT"]
    cached_methods   = ["GET", "HEAD"]
    target_origin_id = "ALB-api"

    forwarded_values {
      query_string = true
      headers      = ["Authorization", "Content-Type", "X-Correlation-Id"]
      cookies {
        forward = "none"
      }
    }

    min_ttl     = 0
    default_ttl = 0      # No caching for API
    max_ttl     = 0

    compress               = true
    viewer_protocol_policy = "redirect-to-https"
  }

  # Default Cache Behavior: React App
  default_cache_behavior {
    allowed_methods  = ["DELETE", "GET", "HEAD", "OPTIONS", "PATCH", "POST", "PUT"]
    cached_methods   = ["GET", "HEAD"]
    target_origin_id = "S3-frontend"

    forwarded_values {
      query_string = true
      headers      = ["Authorization", "Content-Type"]
      cookies {
        forward = "none"
      }
    }

    min_ttl     = 0
    default_ttl = 3600     # 1 hour
    max_ttl     = 86400    # 24 hours

    compress               = true
    viewer_protocol_policy = "redirect-to-https"
  }

  # Restrictions
  restrictions {
    geo_restriction {
      restriction_type = "none"
    }
  }

  # Viewer Certificate
  viewer_certificate {
    acm_certificate_arn      = aws_acm_certificate.cloudfront.arn
    ssl_support_method       = "sni-only"
    minimum_protocol_version = "TLSv1.2_2021"
  }

  # WAF Association
  web_acl_id = aws_wafv2_web_acl.cloudfront.arn

  # Logging
  logging_config {
    include_cookies = false
    bucket          = aws_s3_bucket.cloudfront_logs.bucket_domain_name
    prefix          = "cloudfront-logs/"
  }

  enabled             = true
  is_ipv6_enabled     = true
  comment             = "Revenue Management CDN"
  default_root_object = "index.html"

  tags = {
    Name = "revenue-mgmt-cloudfront"
  }
}

# Origin Access Identity for S3
resource "aws_cloudfront_origin_access_identity" "frontend" {
  comment = "OAI for Revenue Management Frontend"
}
```

### 6.2 CDN Caching Strategy

```yaml
# Caching Rules by Content Type

Static Assets (JS, CSS, Images):
  Path: /static/*
  TTL: 1 year (31536000 seconds)
  Cache Key: URL only
  Compression: Brotli + Gzip
  Invalidation: Version-based filenames (app.[hash].js)

API Responses:
  Path: /api/*
  TTL: 0 (No caching)
  Cache Key: URL + Query String + Authorization Header
  Compression: Gzip
  Note: Use Cache-Control headers from backend

React App (HTML):
  Path: /*
  TTL: 1 hour (3600 seconds)
  Cache Key: URL only
  Compression: Gzip
  Invalidation: On deployment

GraphQL:
  Path: /graphql
  TTL: 0 (No caching)
  Cache Key: URL + Query + Variables
  Compression: Gzip
  Note: Implement persisted queries for caching

Health Check:
  Path: /health
  TTL: 0 (No caching)
  Cache Key: URL only
  Compression: None
```

### 6.3 CloudFront Functions

```javascript
// CloudFront Function: Add Security Headers
function handler(event) {
  var response = event.response;
  var headers = response.headers;

  // Security Headers
  headers['strict-transport-security'] = {
    value: 'max-age=63072000; includeSubDomains; preload'
  };
  
  headers['x-content-type-options'] = {
    value: 'nosniff'
  };
  
  headers['x-frame-options'] = {
    value: 'DENY'
  };
  
  headers['x-xss-protection'] = {
    value: '1; mode=block'
  };
  
  headers['referrer-policy'] = {
    value: 'strict-origin-when-cross-origin'
  };
  
  headers['content-security-policy'] = {
    value: "default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline'; img-src 'self' data: https:; font-src 'self' data:; connect-src 'self' https://*.amazonaws.com;"
  };

  return response;
}

// CloudFront Function: Redirect www to apex
function redirectHandler(event) {
  var request = event.request;
  var host = request.headers.host.value;
  
  if (host.startsWith('www.')) {
    var newHost = host.slice(4);
    return {
      statusCode: 301,
      statusDescription: 'Moved Permanently',
      headers: {
        'location': { value: 'https://' + newHost + request.uri }
      }
    };
  }
  
  return request;
}
```

---

## 7. Object Storage

### 7.1 S3 Bucket Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    S3 BUCKETS ARCHITECTURE                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. FRONTEND BUCKET (Static Website)                             │
│     ├─ Bucket: revenue-mgmt-frontend-prod                        │
│     ├─ Purpose: React application static files                   │
│     ├─ Versioning: Enabled                                       │
│     ├─ Encryption: SSE-S3                                        │
│     ├─ Access: CloudFront OAI only                               │
│     └─ Lifecycle: None (permanent)                               │
│                                                                  │
│  2. DOCUMENTS BUCKET (Business Documents)                        │
│     ├─ Bucket: revenue-mgmt-documents-prod                       │
│     ├─ Purpose: Contracts, invoices, proposals                   │
│     ├─ Versioning: Enabled                                       │
│     ├─ Encryption: SSE-KMS                                       │
│     ├─ Access: EKS services via IAM roles                        │
│     └─ Lifecycle: Transition to Glacier after 7 years            │
│                                                                  │
│  3. BACKUP BUCKET (Database Backups)                             │
│     ├─ Bucket: revenue-mgmt-backups-prod                         │
│     ├─ Purpose: RDS snapshots, application backups               │
│     ├─ Versioning: Enabled                                       │
│     ├─ Encryption: SSE-KMS                                       │
│     ├─ Access: Restricted (backup service only)                  │
│     └─ Lifecycle: Delete after 35 days                           │
│                                                                  │
│  4. LOGS BUCKET (Centralized Logs)                               │
│     ├─ Bucket: revenue-mgmt-logs-prod                            │
│     ├─ Purpose: ALB logs, CloudFront logs, VPC flow logs         │
│     ├─ Versioning: Suspended                                     │
│     ├─ Encryption: SSE-S3                                        │
│     ├─ Access: Logging services only                             │
│     └─ Lifecycle: Delete after 90 days                           │
│                                                                  │
│  5. ARTIFACTS BUCKET (CI/CD)                                     │
│     ├─ Bucket: revenue-mgmt-artifacts-prod                       │
│     ├─ Purpose: Build artifacts, deployment packages             │
│     ├─ Versioning: Enabled                                       │
│     ├─ Encryption: SSE-S3                                        │
│     ├─ Access: CI/CD pipeline only                               │
│     └─ Lifecycle: Delete after 30 days                           │
│                                                                  │
│  6. TEMP BUCKET (Temporary Processing)                           │
│     ├─ Bucket: revenue-mgmt-temp-prod                            │
│     ├─ Purpose: Temporary files during processing                │
│     ├─ Versioning: Suspended                                     │
│     ├─ Encryption: SSE-S3                                        │
│     ├─ Access: EKS services via IAM roles                        │
│     └─ Lifecycle: Delete after 7 days                            │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 7.2 S3 Bucket Configuration

```hcl
# Documents Bucket (Main business bucket)
resource "aws_s3_bucket" "documents" {
  bucket = "revenue-mgmt-documents-${var.environment}"

  tags = {
    Name        = "revenue-mgmt-documents"
    Environment = var.environment
  }
}

# Versioning
resource "aws_s3_bucket_versioning" "documents" {
  bucket = aws_s3_bucket.documents.id

  versioning_configuration {
    status = "Enabled"
  }
}

# Server-side encryption
resource "aws_s3_bucket_server_side_encryption_configuration" "documents" {
  bucket = aws_s3_bucket.documents.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = aws_kms_key.documents.arn
    }
    bucket_key_enabled = true
  }
}

# Block public access
resource "aws_s3_bucket_public_access_block" "documents" {
  bucket = aws_s3_bucket.documents.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# Lifecycle configuration
resource "aws_s3_bucket_lifecycle_configuration" "documents" {
  bucket = aws_s3_bucket.documents.id

  rule {
    id     = "transition-to-glacier"
    status = "Enabled"

    transition {
      days          = 2555 # 7 years
      storage_class = "GLACIER"
    }

    noncurrent_version_transition {
      noncurrent_days = 90
      storage_class   = "GLACIER"
    }

    noncurrent_version_expiration {
      noncurrent_days = 365
    }
  }
}

# CORS configuration (if needed for direct browser uploads)
resource "aws_s3_bucket_cors_configuration" "documents" {
  bucket = aws_s3_bucket.documents.id

  cors_rule {
    allowed_headers = ["*"]
    allowed_methods = ["GET", "PUT", "POST"]
    allowed_origins = ["https://app.revenuemanagement.com"]
    expose_headers  = ["ETag"]
    max_age_seconds = 3000
  }
}

# Bucket policy (restrict access)
resource "aws_s3_bucket_policy" "documents" {
  bucket = aws_s3_bucket.documents.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid       = "DenyInsecureTransport"
        Effect    = "Deny"
        Principal = "*"
        Action    = "s3:*"
        Resource  = [
          aws_s3_bucket.documents.arn,
          "${aws_s3_bucket.documents.arn}/*"
        ]
        Condition = {
          Bool = {
            "aws:SecureTransport" = "false"
          }
        }
      },
      {
        Sid       = "AllowEKSServices"
        Effect    = "Allow"
        Principal = {
          AWS = aws_iam_role.eks_services.arn
        }
        Action = [
          "s3:GetObject",
          "s3:PutObject",
          "s3:DeleteObject"
        ]
        Resource = "${aws_s3_bucket.documents.arn}/*"
      }
    ]
  })
}

# KMS Key for encryption
resource "aws_kms_key" "documents" {
  description             = "KMS key for Revenue Management documents"
  deletion_window_in_days = 30
  enable_key_rotation     = true

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "Enable IAM User Permissions"
        Effect = "Allow"
        Principal = {
          AWS = "arn:aws:iam::${var.aws_account_id}:root"
        }
        Action   = "kms:*"
        Resource = "*"
      },
      {
        Sid    = "Allow EKS Services"
        Effect = "Allow"
        Principal = {
          AWS = aws_iam_role.eks_services.arn
        }
        Action = [
          "kms:Encrypt",
          "kms:Decrypt",
          "kms:GenerateDataKey"
        ]
        Resource = "*"
      }
    ]
  })

  tags = {
    Name = "revenue-mgmt-documents-key"
  }
}
```

### 7.3 S3 Access Patterns

```typescript
// NestJS - S3 Service for document management
@Injectable()
export class DocumentService {
  private readonly s3: S3Client;
  private readonly bucketName: string;

  constructor(
    private readonly configService: ConfigService,
  ) {
    this.s3 = new S3Client({
      region: this.configService.get('AWS_REGION'),
    });
    this.bucketName = this.configService.get('S3_DOCUMENTS_BUCKET');
  }

  async uploadDocument(
    file: Buffer,
    key: string,
    contentType: string,
    metadata: Record<string, string>,
  ): Promise<string> {
    const command = new PutObjectCommand({
      Bucket: this.bucketName,
      Key: key,
      Body: file,
      ContentType: contentType,
      Metadata: metadata,
      ServerSideEncryption: 'aws:kms',
    });

    await this.s3.send(command);
    return `s3://${this.bucketName}/${key}`;
  }

  async getDocument(key: string): Promise<Buffer> {
    const command = new GetObjectCommand({
      Bucket: this.bucketName,
      Key: key,
    });

    const response = await this.s3.send(command);
    return response.Body.transformToByteArray();
  }

  async generatePresignedUrl(
    key: string,
    expiresIn: number = 3600,
  ): Promise<string> {
    const command = new GetObjectCommand({
      Bucket: this.bucketName,
      Key: key,
    });

    return getSignedUrl(this.s3, command, { expiresIn });
  }

  async deleteDocument(key: string): Promise<void> {
    const command = new DeleteObjectCommand({
      Bucket: this.bucketName,
      Key: key,
    });

    await this.s3.send(command);
  }
}
```

---

## 8. Container Registry

### 8.1 ECR Configuration

```hcl
# ECR Repository: API (NestJS)
resource "aws_ecr_repository" "api" {
  name                 = "revenue-mgmt/api"
  image_tag_mutability = "MUTABLE"
  force_delete         = true

  image_scanning_configuration {
    scan_on_push = true
  }

  encryption_configuration {
    encryption_type = "KMS"
    kms_key         = aws_kms_key.ecr.arn
  }

  tags = {
    Name = "revenue-mgmt-api"
  }
}

# ECR Repository: Go Services
resource "aws_ecr_repository" "go_services" {
  name                 = "revenue-mgmt/go-services"
  image_tag_mutability = "MUTABLE"
  force_delete         = true

  image_scanning_configuration {
    scan_on_push = true
  }

  encryption_configuration {
    encryption_type = "KMS"
    kms_key         = aws_kms_key.ecr.arn
  }

  tags = {
    Name = "revenue-mgmt-go-services"
  }
}

# ECR Repository: Python Services
resource "aws_ecr_repository" "python_services" {
  name                 = "revenue-mgmt/python-services"
  image_tag_mutability = "MUTABLE"
  force_delete         = true

  image_scanning_configuration {
    scan_on_push = true
  }

  encryption_configuration {
    encryption_type = "KMS"
    kms_key         = aws_kms_key.ecr.arn
  }

  tags = {
    Name = "revenue-mgmt-python-services"
  }
}

# ECR Repository: Frontend (React)
resource "aws_ecr_repository" "frontend" {
  name                 = "revenue-mgmt/frontend"
  image_tag_mutability = "MUTABLE"
  force_delete         = true

  image_scanning_configuration {
    scan_on_push = true
  }

  encryption_configuration {
    encryption_type = "KMS"
    kms_key         = aws_kms_key.ecr.arn
  }

  tags = {
    Name = "revenue-mgmt-frontend"
  }
}

# Lifecycle Policy: Keep last 30 images
resource "aws_ecr_lifecycle_policy" "api" {
  repository = aws_ecr_repository.api.name

  policy = jsonencode({
    rules = [
      {
        rulePriority = 1
        description  = "Keep last 30 images"
        selection = {
          tagStatus   = "any"
          countType   = "imageCountMoreThan"
          countNumber = 30
        }
        action = {
          type = "expire"
        }
      }
    ]
  })
}

# KMS Key for ECR encryption
resource "aws_kms_key" "ecr" {
  description             = "KMS key for ECR encryption"
  deletion_window_in_days = 30
  enable_key_rotation     = true

  tags = {
    Name = "revenue-mgmt-ecr-key"
  }
}
```

### 8.2 Docker Image Strategy

```yaml
# Multi-stage Dockerfile Example (NestJS API)

# Stage 1: Build
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Stage 2: Production
FROM node:18-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/package.json ./

# Security: Run as non-root user
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nestjs -u 1001
USER nestjs

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD node -e "require('http').get('http://localhost:3000/health', (r) => {process.exit(r.statusCode === 200 ? 0 : 1)})"

EXPOSE 3000
CMD ["node", "dist/main"]

# Image Tagging Strategy
Tags:
  - latest: Latest stable release
  - v1.2.3: Semantic version
  - sha-abc123: Git commit SHA
  - build-456: Build number
  
# Image Scanning
  - Scan on push: Enabled
  - Vulnerability threshold: HIGH
  - Block deployment if: CRITICAL vulnerabilities found
```

### 8.3 ECR Access Policy

```hcl
# IAM Policy for EKS to pull images
resource "aws_iam_policy" "ecr_pull" {
  name        = "revenue-mgmt-ecr-pull"
  description = "Allow EKS to pull images from ECR"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "ecr:GetDownloadUrlForLayer",
          "ecr:BatchGetImage",
          "ecr:DescribeImages"
        ]
        Resource = [
          aws_ecr_repository.api.arn,
          aws_ecr_repository.go_services.arn,
          aws_ecr_repository.python_services.arn,
          aws_ecr_repository.frontend.arn
        ]
      },
      {
        Effect = "Allow"
        Action = [
          "ecr:GetAuthorizationToken"
        ]
        Resource = "*"
      }
    ]
  })
}
```

---

## 9. Kubernetes Cluster

### 9.1 EKS Cluster Configuration

```hcl
# EKS Cluster
resource "aws_eks_cluster" "main" {
  name     = "revenue-mgmt-eks"
  role_arn = aws_iam_role.eks_cluster.arn
  version  = "1.29"

  vpc_config {
    subnet_ids              = concat(
      aws_subnet.private_app[*].id,
      aws_subnet.private_data[*].id
    )
    endpoint_private_access = true
    endpoint_public_access  = true
    public_access_cidrs     = ["10.0.0.0/16"]  # Restrict to VPC
  }

  enabled_cluster_log_types = [
    "api",
    "audit",
    "authenticator",
    "controllerManager",
    "scheduler"
  ]

  encryption_config {
    provider {
      key_arn = aws_kms_key.eks.arn
    }
    resources = ["secrets"]
  }

  tags = {
    Name        = "revenue-mgmt-eks"
    Environment = "production"
  }

  depends_on = [
    aws_iam_role_policy_attachment.eks_cluster_policy,
    aws_iam_role_policy_attachment.eks_vpc_controller_policy,
  ]
}

# EKS Node Groups

# General Purpose Node Group (NestJS, Python services)
resource "aws_eks_node_group" "general" {
  cluster_name    = aws_eks_cluster.main.name
  node_group_name = "general"
  node_role_arn   = aws_iam_role.eks_nodes.arn
  subnet_ids      = aws_subnet.private_app[*].id

  scaling_config {
    desired_size = 6
    max_size     = 12
    min_size     = 3
  }

  instance_types = ["m5.xlarge"]  # 4 vCPU, 16 GB RAM

  capacity_type = "ON_DEMAND"

  labels = {
    role = "general"
  }

  taint {
    key    = "dedicated"
    value  = "general"
    effect = "NO_SCHEDULE"
  }

  update_config {
    max_unavailable = 1
  }

  tags = {
    Name = "revenue-mgmt-eks-general"
  }
}

# Compute Optimized Node Group (Go services, high throughput)
resource "aws_eks_node_group" "compute" {
  cluster_name    = aws_eks_cluster.main.name
  node_group_name = "compute-optimized"
  node_role_arn   = aws_iam_role.eks_nodes.arn
  subnet_ids      = aws_subnet.private_app[*].id

  scaling_config {
    desired_size = 3
    max_size     = 6
    min_size     = 2
  }

  instance_types = ["c5.2xlarge"]  # 8 vCPU, 16 GB RAM

  capacity_type = "ON_DEMAND"

  labels = {
    role = "compute"
  }

  taint {
    key    = "dedicated"
    value  = "compute"
    effect = "NO_SCHEDULE"
  }

  tags = {
    Name = "revenue-mgmt-eks-compute"
  }
}

# Memory Optimized Node Group (Python AI/ML, caching)
resource "aws_eks_node_group" "memory" {
  cluster_name    = aws_eks_cluster.main.name
  node_group_name = "memory-optimized"
  node_role_arn   = aws_iam_role.eks_nodes.arn
  subnet_ids      = aws_subnet.private_app[*].id

  scaling_config {
    desired_size = 2
    max_size     = 4
    min_size     = 1
  }

  instance_types = ["r5.xlarge"]  # 4 vCPU, 32 GB RAM

  capacity_type = "ON_DEMAND"

  labels = {
    role = "memory"
  }

  taint {
    key    = "dedicated"
    value  = "memory"
    effect = "NO_SCHEDULE"
  }

  tags = {
    Name = "revenue-mgmt-eks-memory"
  }
}

# GPU Node Group (ML Training - optional, on-demand)
resource "aws_eks_node_group" "gpu" {
  cluster_name    = aws_eks_cluster.main.name
  node_group_name = "gpu"
  node_role_arn   = aws_iam_role.eks_nodes.arn
  subnet_ids      = aws_subnet.private_app[*].id

  scaling_config {
    desired_size = 0  # Start with 0, scale up when needed
    max_size     = 2
    min_size     = 0
  }

  instance_types = ["g4dn.xlarge"]  # GPU instance

  capacity_type = "ON_DEMAND"

  labels = {
    role = "gpu"
  }

  taint {
    key    = "nvidia.com/gpu"
    value  = "present"
    effect = "NO_SCHEDULE"
  }

  tags = {
    Name = "revenue-mgmt-eks-gpu"
  }
}
```

### 9.2 Kubernetes Namespace Structure

```yaml
# Namespace Organization

apiVersion: v1
kind: Namespace
metadata:
  name: revenue-mgmt
  labels:
    name: revenue-mgmt
    environment: production

---
# Additional Namespaces

apiVersion: v1
kind: Namespace
metadata:
  name: monitoring
  labels:
    name: monitoring

---
apiVersion: v1
kind: Namespace
metadata:
  name: ingress-nginx
  labels:
    name: ingress-nginx

---
apiVersion: v1
kind: Namespace
metadata:
  name: cert-manager
  labels:
    name: cert-manager

---
apiVersion: v1
kind: Namespace
metadata:
  name: argocd
  labels:
    name: argocd
```

### 9.3 Application Deployment Manifests

```yaml
# NestJS API Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: revenue-mgmt-api
  namespace: revenue-mgmt
  labels:
    app: revenue-mgmt-api
    tier: backend
spec:
  replicas: 6
  selector:
    matchLabels:
      app: revenue-mgmt-api
  template:
    metadata:
      labels:
        app: revenue-mgmt-api
        tier: backend
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "3000"
        prometheus.io/path: "/metrics"
    spec:
      nodeSelector:
        role: general
      containers:
        - name: api
          image: ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/revenue-mgmt/api:latest
          ports:
            - containerPort: 3000
              name: http
          env:
            - name: NODE_ENV
              value: "production"
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: revenue-mgmt-secrets
                  key: database-url
            - name: REDIS_URL
              valueFrom:
                secretKeyRef:
                  name: revenue-mgmt-secrets
                  key: redis-url
            - name: KAFKA_BROKERS
              valueFrom:
                secretKeyRef:
                  name: revenue-mgmt-secrets
                  key: kafka-brokers
          resources:
            requests:
              memory: "512Mi"
              cpu: "500m"
            limits:
              memory: "1Gi"
              cpu: "1000m"
          readinessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 10
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 30
            periodSeconds: 30
          volumeMounts:
            - name: tmp
              mountPath: /tmp
      volumes:
        - name: tmp
          emptyDir: {}
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchExpressions:
                    - key: app
                      operator: In
                      values:
                        - revenue-mgmt-api
                topologyKey: kubernetes.io/hostname

---
# Go Billing Service Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: billing-service
  namespace: revenue-mgmt
  labels:
    app: billing-service
    tier: backend
spec:
  replicas: 4
  selector:
    matchLabels:
      app: billing-service
  template:
    metadata:
      labels:
        app: billing-service
        tier: backend
    spec:
      nodeSelector:
        role: compute
      containers:
        - name: billing
          image: ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/revenue-mgmt/go-services:billing-latest
          ports:
            - containerPort: 9090
              name: grpc
            - containerPort: 8080
              name: http
          env:
            - name: ENV
              value: "production"
            - name: DB_HOST
              valueFrom:
                secretKeyRef:
                  name: revenue-mgmt-secrets
                  key: db-host
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: revenue-mgmt-secrets
                  key: db-password
          resources:
            requests:
              memory: "1Gi"
              cpu: "1000m"
            limits:
              memory: "2Gi"
              cpu: "2000m"
          readinessProbe:
            grpc:
              port: 9090
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            grpc:
              port: 9090
            initialDelaySeconds: 15
            periodSeconds: 20

---
# Python AI Service Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ai-service
  namespace: revenue-mgmt
  labels:
    app: ai-service
    tier: backend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: ai-service
  template:
    metadata:
      labels:
        app: ai-service
        tier: backend
    spec:
      nodeSelector:
        role: memory
      containers:
        - name: ai
          image: ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/revenue-mgmt/python-services:ai-latest
          ports:
            - containerPort: 8000
              name: http
          env:
            - name: ENVIRONMENT
              value: "production"
            - name: MODEL_PATH
              value: "/models"
          resources:
            requests:
              memory: "2Gi"
              cpu: "1000m"
            limits:
              memory: "4Gi"
              cpu: "2000m"
          volumeMounts:
            - name: model-storage
              mountPath: /models
      volumes:
        - name: model-storage
          persistentVolumeClaim:
            claimName: model-pvc

---
# Horizontal Pod Autoscaler
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: revenue-mgmt-api-hpa
  namespace: revenue-mgmt
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: revenue-mgmt-api
  minReplicas: 6
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second
        target:
          type: AverageValue
          averageValue: 100
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Percent
          value: 50
          periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
        - type: Percent
          value: 100
          periodSeconds: 60
        - type: Pods
          value: 4
          periodSeconds: 60
      selectPolicy: Max
```

### 9.4 Kubernetes Services & Ingress

```yaml
# Internal Service for API
apiVersion: v1
kind: Service
metadata:
  name: revenue-mgmt-api
  namespace: revenue-mgmt
  labels:
    app: revenue-mgmt-api
spec:
  type: ClusterIP
  ports:
    - port: 3000
      targetPort: 3000
      protocol: TCP
      name: http
  selector:
    app: revenue-mgmt-api

---
# Internal Service for Go Billing (gRPC)
apiVersion: v1
kind: Service
metadata:
  name: billing-service
  namespace: revenue-mgmt
  labels:
    app: billing-service
spec:
  type: ClusterIP
  ports:
    - port: 9090
      targetPort: 9090
      protocol: TCP
      name: grpc
    - port: 8080
      targetPort: 8080
      protocol: TCP
      name: http
  selector:
    app: billing-service

---
# Ingress (via NGINX Ingress Controller)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: revenue-mgmt-ingress
  namespace: revenue-mgmt
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "300"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "300"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - api.revenuemanagement.com
      secretName: revenue-mgmt-tls
  rules:
    - host: api.revenuemanagement.com
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: revenue-mgmt-api
                port:
                  number: 3000
          - path: /graphql
            pathType: Exact
            backend:
              service:
                name: revenue-mgmt-api
                port:
                  number: 3000
          - path: /health
            pathType: Exact
            backend:
              service:
                name: revenue-mgmt-api
                port:
                  number: 3000
```

### 9.5 Kubernetes Resource Quotas

```yaml
# Resource Quota for revenue-mgmt namespace
apiVersion: v1
kind: ResourceQuota
metadata:
  name: revenue-mgmt-quota
  namespace: revenue-mgmt
spec:
  hard:
    requests.cpu: "32"
    requests.memory: "64Gi"
    limits.cpu: "64"
    limits.memory: "128Gi"
    pods: "100"
    services: "20"
    persistentvolumeclaims: "10"
    secrets: "50"
    configmaps: "50"

---
# Limit Range for default container limits
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: revenue-mgmt
spec:
  limits:
    - default:
        cpu: "500m"
        memory: "512Mi"
      defaultRequest:
        cpu: "100m"
        memory: "128Mi"
      type: Container
```

---

## 10. Secrets Management

### 10.1 Secrets Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                  SECRETS MANAGEMENT ARCHITECTURE                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  AWS Secrets Manager                                             │
│  ├─ Database credentials (Aurora)                               │
│  ├─ API keys (Payment gateways, e-signature)                    │
│  ├─ OAuth client secrets                                        │
│  ├─ TLS certificates (if not using ACM)                         │
│  └─ Encryption keys (application-level)                         │
│                                                                  │
│  AWS Systems Manager Parameter Store                             │
│  ├─ Configuration values (non-sensitive)                        │
│  ├─ Feature flags                                               │
│  ├─ Service endpoints                                           │
│  └─ Application settings                                        │
│                                                                  │
│  Kubernetes Secrets (Encrypted at Rest)                          │
│  ├─ Runtime secrets injected via External Secrets Operator      │
│  ├─ TLS certificates for services                               │
│  └─ Service account tokens                                      │
│                                                                  │
│  HashiCorp Vault (Optional, for advanced use cases)              │
│  ├─ Dynamic database credentials                                │
│  ├─ PKI certificates                                            │
│  └─ Secret rotation automation                                  │
│                                                                  │
─────────────────────────────────────────────────────────────────┘
```

### 10.2 AWS Secrets Manager Configuration

```hcl
# Database Credentials Secret
resource "aws_secretsmanager_secret" "database" {
  name                    = "revenue-mgmt/database"
  description             = "Aurora PostgreSQL credentials"
  recovery_window_in_days = 30

  tags = {
    Name        = "revenue-mgmt-database"
    Environment = "production"
  }
}

resource "aws_secretsmanager_secret_version" "database" {
  secret_id = aws_secretsmanager_secret.database.id
  secret_string = jsonencode({
    username = "revenue_admin"
    password = random_password.database.result
    engine   = "postgres"
    host     = aws_rds_cluster.aurora.endpoint
    port     = 5432
    dbname   = "revenue_mgmt"
  })
}

# Redis Credentials
resource "aws_secretsmanager_secret" "redis" {
  name = "revenue-mgmt/redis"

  tags = {
    Name = "revenue-mgmt-redis"
  }
}

resource "aws_secretsmanager_secret_version" "redis" {
  secret_id = aws_secretsmanager_secret.redis.id
  secret_string = jsonencode({
    host     = aws_elasticache_cluster.redis.cache_nodes[0].address
    port     = 6379
    password = random_password.redis.result
  })
}

# Kafka Credentials
resource "aws_secretsmanager_secret" "kafka" {
  name = "revenue-mgmt/kafka"

  tags = {
    Name = "revenue-mgmt-kafka"
  }
}

resource "aws_secretsmanager_secret_version" "kafka" {
  secret_id = aws_secretsmanager_secret.kafka.id
  secret_string = jsonencode({
    brokers         = aws_msk_cluster.main.bootstrap_brokers
    brokers_tls     = aws_msk_cluster.main.bootstrap_brokers_tls
    sasl_username   = "kafka-user"
    sasl_password   = random_password.kafka.result
  })
}

# Payment Gateway API Key
resource "aws_secretsmanager_secret" "payment_gateway" {
  name = "revenue-mgmt/payment-gateway"

  tags = {
    Name = "revenue-mgmt-payment-gateway"
  }
}

resource "aws_secretsmanager_secret_version" "payment_gateway" {
  secret_id = aws_secretsmanager_secret.payment_gateway.id
  secret_string = jsonencode({
    stripe_secret_key = var.stripe_secret_key
    stripe_publishable_key = var.stripe_publishable_key
    webhook_secret = var.stripe_webhook_secret
  })
}

# Random Passwords
resource "random_password" "database" {
  length  = 32
  special = true
}

resource "random_password" "redis" {
  length  = 32
  special = false
}

resource "random_password" "kafka" {
  length  = 32
  special = true
}
```

### 10.3 External Secrets Operator (Kubernetes)

```yaml
# ExternalSecret: Database Credentials
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: revenue-mgmt-database
  namespace: revenue-mgmt
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secretsmanager
    kind: ClusterSecretStore
  target:
    name: revenue-mgmt-secrets
    creationPolicy: Owner
  data:
    - secretKey: database-url
      remoteRef:
        key: revenue-mgmt/database
        property: host
    - secretKey: database-username
      remoteRef:
        key: revenue-mgmt/database
        property: username
    - secretKey: database-password
      remoteRef:
        key: revenue-mgmt/database
        property: password

---
# ClusterSecretStore: AWS Secrets Manager
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: aws-secretsmanager
spec:
  provider:
    aws:
      service: SecretsManager
      region: ap-southeast-1
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets-sa
            namespace: revenue-mgmt

---
# ServiceAccount for External Secrets
apiVersion: v1
kind: ServiceAccount
metadata:
  name: external-secrets-sa
  namespace: revenue-mgmt
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::${AWS_ACCOUNT_ID}:role/revenue-mgmt-external-secrets-role

---
# IAM Role for Service Account (IRSA)
resource "aws_iam_role" "external_secrets" {
  name = "revenue-mgmt-external-secrets-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          Federated = "arn:aws:iam::${var.aws_account_id}:oidc-provider/${aws_eks_cluster.main.identity[0].oidc[0].issuer}"
        }
        Action = "sts:AssumeRoleWithWebIdentity"
        Condition = {
          StringEquals = {
            "${aws_eks_cluster.main.identity[0].oidc[0].issuer}:sub" = "system:serviceaccount:revenue-mgmt:external-secrets-sa"
          }
        }
      }
    ]
  })
}

resource "aws_iam_role_policy" "external_secrets" {
  name = "revenue-mgmt-external-secrets-policy"
  role = aws_iam_role.external_secrets.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "secretsmanager:GetSecretValue",
          "secretsmanager:DescribeSecret"
        ]
        Resource = "arn:aws:secretsmanager:${var.aws_region}:${var.aws_account_id}:secret:revenue-mgmt/*"
      }
    ]
  })
}
```

### 10.4 Secret Rotation

```hcl
# Automatic Rotation for Database Credentials
resource "aws_secretsmanager_secret_rotation" "database" {
  secret_id = aws_secretsmanager_secret.database.id

  rotation_lambda_arn = aws_lambda_function.secret_rotation.arn

  rotation_rules {
    automatically_after_days = 90
  }
}

# Lambda Function for Rotation
resource "aws_lambda_function" "secret_rotation" {
  filename         = "secret-rotation.zip"
  function_name    = "revenue-mgmt-secret-rotation"
  role             = aws_iam_role.lambda_rotation.arn
  handler          = "index.handler"
  runtime          = "nodejs18.x"
  timeout          = 300

  environment {
    variables = {
      SECRETS_MANAGER_ENDPOINT = "https://secretsmanager.ap-southeast-1.amazonaws.com"
    }
  }

  vpc_config {
    subnet_ids         = aws_subnet.private_app[*].id
    security_group_ids = [aws_security_group.lambda_rotation.id]
  }

  tags = {
    Name = "revenue-mgmt-secret-rotation"
  }
}
```

---

## 11. CI/CD

### 11.1 CI/CD Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    CI/CD PIPELINE ARCHITECTURE                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  SOURCE CONTROL (GitHub)                                         │
│  ├─ Branch: main (production)                                   │
│  ├─ Branch: develop (staging)                                   │
│  ├─ Branch: feature/* (development)                             │
│  └─ Pull Requests with required reviews                         │
│                                                                  │
│  CI PIPELINE (GitHub Actions)                                    │
│  ├─ Trigger: Push to any branch, PR creation                    │
│  ├─ Steps:                                                       │
│  │  1. Checkout code                                            │
│  │  2. Setup Node.js/Go/Python                                  │
│  │  3. Install dependencies                                     │
│  │  4. Lint (ESLint, golangci-lint, Ruff)                       │
│  │  5. Unit tests with coverage                                 │
│  │  6. Build Docker images                                      │
│  │  7. Security scan (Trivy, Snyk)                              │
│  │  8. Push to ECR                                              │
│  │  9. Update Kubernetes manifests                              │
│  └─ Artifacts: Docker images, test reports                      │
│                                                                  │
│  CD PIPELINE (ArgoCD - GitOps)                                   │
│  ├─ Repository: Kubernetes manifests (Git)                      │
│  ├─ Sync Policy: Automatic (with approval for production)       │
│  ├─ Health Checks: Custom health checks                         │
│  └─ Rollback: Automatic on failure                              │
│                                                                  │
│  DEPLOYMENT STRATEGY                                             │
│  ├─ Development: Direct deployment                              │
│  ├─ Staging: Automatic after CI pass                            │
│  └─ Production: Manual approval + Blue-Green                    │
│                                                                  │
─────────────────────────────────────────────────────────────────┘
```

### 11.2 GitHub Actions Workflow

```yaml
# .github/workflows/ci-cd.yml
name: CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

env:
  AWS_REGION: ap-southeast-1
  ECR_REGISTRY: ${{ secrets.AWS_ACCOUNT_ID }}.dkr.ecr.ap-southeast-1.amazonaws.com
  EKS_CLUSTER_NAME: revenue-mgmt-eks

jobs:
  # Job 1: Lint and Test
  lint-and-test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        service: [api, billing, ai, frontend]
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js
        if: matrix.service == 'api' || matrix.service == 'frontend'
        uses: actions/setup-node@v4
        with:
          node-version: '18'
          cache: 'npm'
      
      - name: Setup Go
        if: matrix.service == 'billing'
        uses: actions/setup-go@v5
        with:
          go-version: '1.21'
      
      - name: Setup Python
        if: matrix.service == 'ai'
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'
      
      - name: Install dependencies
        run: |
          if [ "${{ matrix.service }}" = "api" ] || [ "${{ matrix.service }}" = "frontend" ]; then
            npm ci
          elif [ "${{ matrix.service }}" = "billing" ]; then
            go mod download
          elif [ "${{ matrix.service }}" = "ai" ]; then
            pip install -r requirements.txt
          fi
      
      - name: Lint
        run: |
          if [ "${{ matrix.service }}" = "api" ] || [ "${{ matrix.service }}" = "frontend" ]; then
            npm run lint
          elif [ "${{ matrix.service }}" = "billing" ]; then
            golangci-lint run
          elif [ "${{ matrix.service }}" = "ai" ]; then
            ruff check .
          fi
      
      - name: Run tests
        run: |
          if [ "${{ matrix.service }}" = "api" ] || [ "${{ matrix.service }}" = "frontend" ]; then
            npm run test:coverage
          elif [ "${{ matrix.service }}" = "billing" ]; then
            go test -coverprofile=coverage.out ./...
          elif [ "${{ matrix.service }}" = "ai" ]; then
            pytest --cov=src --cov-report=xml
          fi
      
      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          files: ./coverage.xml
          flags: ${{ matrix.service }}

  # Job 2: Build and Push Docker Images
  build-and-push:
    needs: lint-and-test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main' || github.ref == 'refs/heads/develop'
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}
      
      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2
      
      - name: Build, tag, and push image
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          IMAGE_TAG: ${{ github.sha }}
        run: |
          # Build API image
          docker build -t $ECR_REGISTRY/revenue-mgmt/api:$IMAGE_TAG -f api/Dockerfile ./api
          docker push $ECR_REGISTRY/revenue-mgmt/api:$IMAGE_TAG
          
          # Build Go services
          docker build -t $ECR_REGISTRY/revenue-mgmt/go-services:billing-$IMAGE_TAG -f services/billing/Dockerfile ./services/billing
          docker push $ECR_REGISTRY/revenue-mgmt/go-services:billing-$IMAGE_TAG
          
          # Build Python services
          docker build -t $ECR_REGISTRY/revenue-mgmt/python-services:ai-$IMAGE_TAG -f services/ai/Dockerfile ./services/ai
          docker push $ECR_REGISTRY/revenue-mgmt/python-services:ai-$IMAGE_TAG
          
          # Build Frontend
          docker build -t $ECR_REGISTRY/revenue-mgmt/frontend:$IMAGE_TAG -f frontend/Dockerfile ./frontend
          docker push $ECR_REGISTRY/revenue-mgmt/frontend:$IMAGE_TAG
      
      - name: Update Kubernetes manifests
        run: |
          # Update image tags in k8s manifests
          sed -i "s|image:.*api:.*|image: $ECR_REGISTRY/revenue-mgmt/api:$IMAGE_TAG|g" k8s/api-deployment.yaml
          sed -i "s|image:.*go-services:.*|image: $ECR_REGISTRY/revenue-mgmt/go-services:billing-$IMAGE_TAG|g" k8s/billing-deployment.yaml
          sed -i "s|image:.*python-services:.*|image: $ECR_REGISTRY/revenue-mgmt/python-services:ai-$IMAGE_TAG|g" k8s/ai-deployment.yaml
          sed -i "s|image:.*frontend:.*|image: $ECR_REGISTRY/revenue-mgmt/frontend:$IMAGE_TAG|g" k8s/frontend-deployment.yaml
      
      - name: Commit and push manifests
        run: |
          git config --global user.name 'GitHub Actions'
          git config --global user.email 'actions@github.com'
          git add k8s/
          git commit -m "Update image tags to $IMAGE_TAG"
          git push

  # Job 3: Deploy to Staging
  deploy-staging:
    needs: build-and-push
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/develop'
    environment: staging
    
    steps:
      - name: Deploy to staging
        run: |
          echo "ArgoCD will automatically sync from git repository"
          echo "Staging deployment triggered"

  # Job 4: Deploy to Production (with approval)
  deploy-production:
    needs: build-and-push
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    environment: 
      name: production
      url: https://app.revenuemanagement.com
    
    steps:
      - name: Deploy to production
        run: |
          echo "ArgoCD will automatically sync from git repository"
          echo "Production deployment triggered"
      
      - name: Run smoke tests
        run: |
          # Wait for deployment to complete
          kubectl rollout status deployment/revenue-mgmt-api -n revenue-mgmt --timeout=300s
          
          # Run health check
          curl -f https://api.revenuemanagement.com/health || exit 1
          
          echo "Production deployment successful"
```

### 11.3 ArgoCD Configuration

```yaml
# ArgoCD Application
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: revenue-mgmt
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/your-org/revenue-mgmt-k8s.git
    targetRevision: HEAD
    path: k8s/overlays/production
  destination:
    server: https://kubernetes.default.svc
    namespace: revenue-mgmt
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - PrunePropagationPolicy=foreground
      - PruneLast=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
  revisionHistoryLimit: 10

---
# ArgoCD Project
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: revenue-mgmt
  namespace: argocd
spec:
  description: Revenue Management Application
  sourceRepos:
    - https://github.com/your-org/revenue-mgmt-k8s.git
  destinations:
    - namespace: revenue-mgmt
      server: https://kubernetes.default.svc
    - namespace: monitoring
      server: https://kubernetes.default.svc
  clusterResourceWhitelist:
    - group: '*'
      kind: '*'
  namespaceResourceWhitelist:
    - group: '*'
      kind: '*'
```

### 11.4 Deployment Strategies

```yaml
# Blue-Green Deployment (Production)
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: revenue-mgmt-api
  namespace: revenue-mgmt
spec:
  replicas: 6
  strategy:
    blueGreen:
      activeService: revenue-mgmt-api
      previewService: revenue-mgmt-api-preview
      autoPromotionEnabled: false
      autoPromotionSeconds: 300
      prePromotionAnalysis:
        templates:
          - templateName: smoke-test
      postPromotionAnalysis:
        templates:
          - templateName: canary-analysis
  selector:
    matchLabels:
      app: revenue-mgmt-api
  template:
    metadata:
      labels:
        app: revenue-mgmt-api
    spec:
      containers:
        - name: api
          image: revenue-mgmt/api:latest
          ports:
            - containerPort: 3000

---
# Canary Deployment (Staging)
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: revenue-mgmt-api-canary
  namespace: revenue-mgmt
spec:
  replicas: 6
  strategy:
    canary:
      steps:
        - setWeight: 10
        - pause: {duration: 10m}
        - setWeight: 25
        - pause: {duration: 10m}
        - setWeight: 50
        - pause: {duration: 10m}
        - setWeight: 75
        - pause: {duration: 10m}
        - setWeight: 100
      analysis:
        templates:
          - templateName: success-rate
          - templateName: latency
  selector:
    matchLabels:
      app: revenue-mgmt-api
  template:
    metadata:
      labels:
        app: revenue-mgmt-api
    spec:
      containers:
        - name: api
          image: revenue-mgmt/api:latest
```

---

## 12. Monitoring

### 12.1 Monitoring Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    MONITORING STACK                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  METRICS COLLECTION                                              │
│  ├─ Prometheus (Kubernetes metrics)                             │
│  ├─ Node Exporter (Host metrics)                                │
│  ├─ AWS CloudWatch (AWS service metrics)                        │
│  └─ Custom exporters (Application metrics)                      │
│                                                                  │
│  LOGGING                                                         │
│  ├─ Fluentd/Fluent Bit (Log collection)                         │
│  ├─ Elasticsearch (Log storage)                                 │
│  ├─ Kibana (Log visualization)                                  │
│  └─ CloudWatch Logs (AWS service logs)                          │
│                                                                  │
│  TRACING                                                         │
│  ├─ Jaeger (Distributed tracing)                                │
│  ├─ AWS X-Ray (AWS service tracing)                             │
│  └─ OpenTelemetry (Instrumentation)                             │
│                                                                  │
│  VISUALIZATION & ALERTING                                        │
│  ├─ Grafana (Dashboards)                                        │
│  ├─ Prometheus Alertmanager (Alerting)                          │
│  ├─ PagerDuty (Incident management)                             │
│  └─ Slack (Notifications)                                       │
│                                                                  │
│  APPLICATION PERFORMANCE MONITORING (APM)                        │
│  ├─ New Relic / Datadog (Optional)                              │
│  ├─ Custom metrics via OpenTelemetry                            │
│  └─ Kubernetes metrics via kube-state-metrics                   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 12.2 Prometheus Configuration

```yaml
# Prometheus Helm Values
server:
  replicas: 2
  resources:
    requests:
      memory: "2Gi"
      cpu: "1000m"
    limits:
      memory: "4Gi"
      cpu: "2000m"
  persistentVolume:
    enabled: true
    size: 100Gi
    storageClass: gp3

alertmanager:
  enabled: true
  replicas: 2
  config:
    route:
      group_by: ['alertname', 'cluster', 'service']
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 12h
      receiver: 'pagerduty'
      routes:
        - match:
            severity: 'critical'
          receiver: 'pagerduty'
        - match:
            severity: 'warning'
          receiver: 'slack'

extraScrapeConfigs: |
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)
      - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
        action: replace
        regex: ([^:]+)(?::\d+)?;(\d+)
        replacement: $1:$2
        target_label: __address__
      - action: labelmap
        regex: __meta_kubernetes_pod_label_(.+)
      - source_labels: [__meta_kubernetes_namespace]
        action: replace
        target_label: kubernetes_namespace
      - source_labels: [__meta_kubernetes_pod_name]
        action: replace
        target_label: kubernetes_pod_name

  - job_name: 'kubernetes-services'
    kubernetes_sd_configs:
      - role: service
    metrics_path: /probe
    params:
      module: [http_2xx]
    relabel_configs:
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_probe]
        action: keep
        regex: true
      - source_labels: [__address__]
        target_label: __param_target
      - target_label: __address__
        replacement: blackbox-exporter:9115
      - source_labels: [__param_target]
        target_label: instance
      - action: labelmap
        regex: __meta_kubernetes_service_label_(.+)
      - source_labels: [__meta_kubernetes_namespace]
        target_label: kubernetes_namespace
      - source_labels: [__meta_kubernetes_service_name]
        target_label: kubernetes_name
```

### 12.3 Grafana Dashboards

```yaml
# Grafana Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana
  namespace: monitoring
spec:
  replicas: 2
  selector:
    matchLabels:
      app: grafana
  template:
    metadata:
      labels:
        app: grafana
    spec:
      containers:
        - name: grafana
          image: grafana/grafana:10.3.0
          ports:
            - containerPort: 3000
          env:
            - name: GF_SECURITY_ADMIN_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: grafana-secrets
                  key: admin-password
            - name: GF_INSTALL_PLUGINS
              value: "grafana-piechart-panel,grafana-worldmap-panel"
          resources:
            requests:
              memory: "512Mi"
              cpu: "250m"
            limits:
              memory: "1Gi"
              cpu: "500m"
          volumeMounts:
            - name: grafana-storage
              mountPath: /var/lib/grafana
            - name: grafana-dashboards
              mountPath: /var/lib/grafana/dashboards
      volumes:
        - name: grafana-storage
          persistentVolumeClaim:
            claimName: grafana-pvc
        - name: grafana-dashboards
          configMap:
            name: grafana-dashboards

---
# Grafana Dashboards ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-dashboards
  namespace: monitoring
data:
  revenue-mgmt-overview.json: |
    {
      "dashboard": {
        "title": "Revenue Management Overview",
        "panels": [
          {
            "title": "API Request Rate",
            "type": "graph",
            "targets": [
              {
                "expr": "rate(http_requests_total{job=\"revenue-mgmt-api\"}[5m])",
                "legendFormat": "{{method}} {{endpoint}}"
              }
            ]
          },
          {
            "title": "Invoice Generation Rate",
            "type": "stat",
            "targets": [
              {
                "expr": "rate(invoice_generated_total[1h])",
                "legendFormat": "Invoices/hour"
              }
            ]
          },
          {
            "title": "Database Connection Pool",
            "type": "gauge",
            "targets": [
              {
                "expr": "pg_stat_activity_count{datname=\"revenue_mgmt\"}",
                "legendFormat": "Active connections"
              }
            ]
          },
          {
            "title": "Kafka Consumer Lag",
            "type": "graph",
            "targets": [
              {
                "expr": "kafka_consumer_group_lag{group=\"billing-consumer\"}",
                "legendFormat": "{{topic}}-{{partition}}"
              }
            ]
          }
        ]
      }
    }
```

### 12.4 Alerting Rules

```yaml
# Prometheus Alerting Rules
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-alerts
  namespace: monitoring
data:
  revenue-mgmt-alerts.yaml: |
    groups:
      - name: revenue-mgmt-alerts
        rules:
          # Critical Alerts
          - alert: HighErrorRate
            expr: |
              sum(rate(http_requests_total{job="revenue-mgmt-api", status=~"5.."}[5m])) 
              / 
              sum(rate(http_requests_total{job="revenue-mgmt-api"}[5m])) > 0.05
            for: 5m
            labels:
              severity: critical
              team: revenue-mgmt
            annotations:
              summary: "High error rate detected"
              description: "Error rate is {{ $value | humanizePercentage }} for the last 5 minutes"
              runbook_url: "https://wiki.internal/runbooks/high-error-rate"

          - alert: DatabaseDown
            expr: pg_up{job="postgres"} == 0
            for: 1m
            labels:
              severity: critical
              team: revenue-mgmt
            annotations:
              summary: "Database is down"
              description: "PostgreSQL instance {{ $labels.instance }} is down"

          - alert: HighCPUUsage
            expr: |
              sum(rate(container_cpu_usage_seconds_total{namespace="revenue-mgmt"}[5m])) 
              / 
              sum(container_spec_cpu_quota{namespace="revenue-mgmt"} / container_spec_cpu_period{namespace="revenue-mgmt"}) > 0.9
            for: 10m
            labels:
              severity: critical
              team: revenue-mgmt
            annotations:
              summary: "High CPU usage"
              description: "CPU usage is {{ $value | humanizePercentage }} for the last 10 minutes"

          - alert: HighMemoryUsage
            expr: |
              sum(container_memory_working_set_bytes{namespace="revenue-mgmt"}) 
              / 
              sum(container_spec_memory_limit_bytes{namespace="revenue-mgmt"}) > 0.9
            for: 10m
            labels:
              severity: critical
              team: revenue-mgmt
            annotations:
              summary: "High memory usage"
              description: "Memory usage is {{ $value | humanizePercentage }}"

          - alert: KafkaConsumerLag
            expr: kafka_consumer_group_lag{group="billing-consumer"} > 10000
            for: 15m
            labels:
              severity: warning
              team: revenue-mgmt
            annotations:
              summary: "High Kafka consumer lag"
              description: "Consumer lag is {{ $value }} for group {{ $labels.group }}"

          - alert: InvoiceGenerationSlow
            expr: |
              histogram_quantile(0.95, 
                rate(invoice_generation_duration_seconds_bucket[5m])
              ) > 30
            for: 10m
            labels:
              severity: warning
              team: revenue-mgmt
            annotations:
              summary: "Invoice generation is slow"
              description: "95th percentile invoice generation time is {{ $value }}s"

          - alert: DiskSpaceLow
            expr: |
              (node_filesystem_avail_bytes{mountpoint="/var/lib/postgresql"} 
              / 
              node_filesystem_size_bytes{mountpoint="/var/lib/postgresql"}) < 0.1
            for: 5m
            labels:
              severity: critical
              team: revenue-mgmt
            annotations:
              summary: "Low disk space"
              description: "Disk space is {{ $value | humanizePercentage }} on {{ $labels.instance }}"

          # Warning Alerts
          - alert: HighLatency
            expr: |
              histogram_quantile(0.95, 
                rate(http_request_duration_seconds_bucket{job="revenue-mgmt-api"}[5m])
              ) > 2
            for: 10m
            labels:
              severity: warning
              team: revenue-mgmt
            annotations:
              summary: "High API latency"
              description: "95th percentile latency is {{ $value }}s"

          - alert: PodRestarting
            expr: |
              increase(kube_pod_container_status_restarts_total{namespace="revenue-mgmt"}[1h]) > 5
            for: 5m
            labels:
              severity: warning
              team: revenue-mgmt
            annotations:
              summary: "Pod restarting frequently"
              description: "Pod {{ $labels.pod }} has restarted {{ $value }} times in the last hour"
```

### 12.5 Distributed Tracing (Jaeger)

```yaml
# Jaeger Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jaeger
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: jaeger
  template:
    metadata:
      labels:
        app: jaeger
    spec:
      containers:
        - name: jaeger
          image: jaegertracing/all-in-one:1.53
          ports:
            - containerPort: 16686
              name: ui
            - containerPort: 14268
              name: collector
            - containerPort: 6831
              name: agent-compact
          env:
            - name: COLLECTOR_OTLP_ENABLED
              value: "true"
            - name: SPAN_STORAGE_TYPE
              value: "elasticsearch"
            - name: ES_SERVER_URLS
              value: "http://elasticsearch:9200"
          resources:
            requests:
              memory: "1Gi"
              cpu: "500m"
            limits:
              memory: "2Gi"
              cpu: "1000m"

---
# OpenTelemetry Collector
apiVersion: apps/v1
kind: Deployment
metadata:
  name: otel-collector
  namespace: monitoring
spec:
  replicas: 2
  selector:
    matchLabels:
      app: otel-collector
  template:
    metadata:
      labels:
        app: otel-collector
    spec:
      containers:
        - name: otel-collector
          image: otel/opentelemetry-collector:0.91.0
          ports:
            - containerPort: 4317
              name: otlp-grpc
            - containerPort: 4318
              name: otlp-http
            - containerPort: 8888
              name: metrics
          volumeMounts:
            - name: otel-config
              mountPath: /etc/otelcol/config.yaml
              subPath: config.yaml
      volumes:
        - name: otel-config
          configMap:
            name: otel-collector-config

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: otel-collector-config
  namespace: monitoring
data:
  config.yaml: |
    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317
          http:
            endpoint: 0.0.0.0:4318

    processors:
      batch:
        send_batch_size: 10000
        timeout: 10s
      memory_limiter:
        limit_mib: 1500
        spike_limit_mib: 500
        check_interval: 5s

    exporters:
      jaeger:
        endpoint: jaeger:14250
        tls:
          insecure: true
      prometheus:
        endpoint: 0.0.0.0:8889

    service:
      pipelines:
        traces:
          receivers: [otlp]
          processors: [memory_limiter, batch]
          exporters: [jaeger]
        metrics:
          receivers: [otlp]
          processors: [memory_limiter, batch]
          exporters: [prometheus]
```

---

## 13. Backup Strategy

### 13.1 Backup Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    BACKUP STRATEGY                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  DATABASE BACKUPS (Aurora PostgreSQL)                            │
│  ├─ Automated Backups: Enabled (35 days retention)              │
│  ├─ Point-in-Time Recovery: Enabled                             │
│  ├─ Cross-Region Snapshots: Daily to ap-northeast-1             │
│  ├─ Manual Snapshots: Before major changes                      │
│  └─ Backup Verification: Weekly restore test                    │
│                                                                  │
│  REDIS BACKUPS (ElastiCache)                                     │
│  ├─ Automated Backups: Enabled (7 days retention)               │
│  ├─ Manual Snapshots: Before maintenance                        │
│  └─ Note: Redis is cache, data can be regenerated               │
│                                                                  │
│  KAFKA BACKUPS (MSK)                                             │
│  ├─ Retention: 7 days (configurable per topic)                  │
│  ├─ Cross-Region Replication: MirrorMaker 2.0                   │
│  ─ Note: Events are ephemeral, focus on retention              │
│                                                                  │
│  S3 BACKUPS                                                      │
│  ├─ Versioning: Enabled on all buckets                          │
│  ├─ Cross-Region Replication: Enabled                           │
│  ├─ Lifecycle Policies: Transition to Glacier                   │
│  └─ MFA Delete: Enabled for critical buckets                    │
│                                                                  │
│  KUBERNETES BACKUPS                                              │
│  ├─ Velero: Cluster state backup                                │
│  ├─ Git Repository: Kubernetes manifests (source of truth)      │
│  ─ etcd Backups: Automated via EKS                             │
│                                                                  │
│  APPLICATION BACKUPS                                             │
│  ├─ Configuration: Stored in Git                                │
│  ├─ Secrets: AWS Secrets Manager (automated backup)             │
│  └─ State: Database (primary source)                            │
│                                                                  │
─────────────────────────────────────────────────────────────────┘
```

### 13.2 Aurora Backup Configuration

```hcl
# Aurora Cluster with Backup Configuration
resource "aws_rds_cluster" "aurora" {
  cluster_identifier     = "revenue-mgmt-aurora"
  engine                 = "aurora-postgresql"
  engine_version         = "15.4"
  database_name          = "revenue_mgmt"
  master_username        = "revenue_admin"
  master_password        = random_password.database.result
  backup_retention_period = 35
  preferred_backup_window = "03:00-04:00"
  preferred_maintenance_window = "sun:04:00-sun:05:00"
  deletion_protection    = true
  skip_final_snapshot    = false
  final_snapshot_identifier = "revenue-mgmt-aurora-final"

  vpc_security_group_ids = [aws_security_group.aurora.id]
  db_subnet_group_name   = aws_db_subnet_group.aurora.name

  enabled_cloudwatch_logs_exports = [
    "postgresql",
    "upgrade"
  ]

  tags = {
    Name        = "revenue-mgmt-aurora"
    Environment = "production"
  }
}

# Aurora Cluster Instance
resource "aws_rds_cluster_instance" "aurora" {
  count              = 3
  identifier         = "revenue-mgmt-aurora-${count.index}"
  cluster_identifier = aws_rds_cluster.aurora.id
  instance_class     = "db.r5.xlarge"
  engine             = aws_rds_cluster.aurora.engine
  engine_version     = aws_rds_cluster.aurora.engine_version

  tags = {
    Name = "revenue-mgmt-aurora-instance-${count.index}"
  }
}

# Automated Backup to S3 (Optional)
resource "aws_rds_cluster" "aurora_s3_export" {
  # Configure S3 export for long-term retention
  s3_import {
    source_engine         = "postgresql"
    source_engine_version = "15.4"
    bucket_name           = aws_s3_bucket.backups.id
    bucket_prefix         = "aurora-import/"
    ingestion_role        = aws_iam_role.aurora_s3_import.arn
  }
}

# Cross-Region Snapshot Copy
resource "aws_db_cluster_snapshot" "cross_region" {
  db_cluster_identifier          = aws_rds_cluster.aurora.id
  db_cluster_snapshot_identifier = "revenue-mgmt-aurora-snapshot-${formatdate("YYYY-MM-DD-HHMM", timestamp())}"

  # Copy to Tokyo region
  lifecycle {
    create_before_destroy = true
  }
}
```

### 13.3 Velero (Kubernetes Backup)

```yaml
# Velero Installation
apiVersion: apps/v1
kind: Deployment
metadata:
  name: velero
  namespace: velero
spec:
  replicas: 1
  selector:
    matchLabels:
      app: velero
  template:
    metadata:
      labels:
        app: velero
    spec:
      serviceAccountName: velero
      containers:
        - name: velero
          image: velero/velero:v1.12.0
          command:
            - /velero
          args:
            - server
            - --default-volumes-to-fs-backup
            - --metrics-address=0.0.0.0:8085
          ports:
            - containerPort: 8085
              name: metrics
          resources:
            requests:
              memory: "512Mi"
              cpu: "500m"
            limits:
              memory: "1Gi"
              cpu: "1000m"
          volumeMounts:
            - name: cloud-credentials
              mountPath: /credentials
          env:
            - name: AWS_SHARED_CREDENTIALS_FILE
              value: /credentials/cloud
            - name: VELERO_SCRATCH_DIR
              value: /scratch
      volumes:
        - name: cloud-credentials
          secret:
            secretName: cloud-credentials

---
# Velero Schedule: Daily backup
apiVersion: velero.io/v1
kind: Schedule
metadata:
  name: daily-backup
  namespace: velero
spec:
  schedule: "0 2 * * *"  # Daily at 2 AM
  template:
    ttl: 720h0m0s  # 30 days retention
    includedNamespaces:
      - revenue-mgmt
      - monitoring
    storageLocation: default
    volumeSnapshotLocations:
      - default
    defaultVolumesToFsBackup: true

---
# Velero Schedule: Weekly full backup
apiVersion: velero.io/v1
kind: Schedule
metadata:
  name: weekly-full-backup
  namespace: velero
spec:
  schedule: "0 3 * * 0"  # Weekly on Sunday at 3 AM
  template:
    ttl: 2160h0m0s  # 90 days retention
    includedNamespaces:
      - '*'
    storageLocation: default
    volumeSnapshotLocations:
      - default
    defaultVolumesToFsBackup: true

---
# Backup Storage Location
apiVersion: velero.io/v1
kind: BackupStorageLocation
metadata:
  name: default
  namespace: velero
spec:
  provider: aws
  default: true
  objectStorage:
    bucket: revenue-mgmt-backups-prod
    prefix: velero
  config:
    region: ap-southeast-1
    s3ForcePathStyle: "false"
```

### 13.4 Backup Verification & Testing

```yaml
# Backup Test Schedule
Backup Testing:
  Daily:
    - Verify backup completion status
    - Check backup size anomalies
    - Monitor backup duration
  
  Weekly:
    - Restore test to staging environment
    - Verify data integrity
    - Test application functionality
  
  Monthly:
    - Full disaster recovery drill
    - Cross-region restore test
    - RTO/RPO validation
  
  Quarterly:
    - Complete DR exercise
    - Stakeholder review
    - Update runbooks

# Backup Verification Script
#!/bin/bash
# verify-backup.sh

echo "Verifying backups..."

# Check Aurora backups
aws rds describe-db-cluster-snapshots \
  --db-cluster-identifier revenue-mgmt-aurora \
  --query 'DBClusterSnapshots[?SnapshotType==`automated`].{ID:DBClusterSnapshotIdentifier,Status:Status,Created:SnapshotCreateTime}' \
  --output table

# Check S3 bucket versioning
aws s3api get-bucket-versioning \
  --bucket revenue-mgmt-documents-prod

# Check Velero backups
kubectl get backups -n velero -o wide

# Check backup age
echo "Latest backup age:"
aws rds describe-db-cluster-snapshots \
  --db-cluster-identifier revenue-mgmt-aurora \
  --query 'max_by(DBClusterSnapshots, &SnapshotCreateTime).SnapshotCreateTime' \
  --output text
```

---

## 14. Cost Estimation

### 14.1 Monthly Cost Breakdown

```
┌─────────────────────────────────────────────────────────────────┐
│              MONTHLY COST ESTIMATION (Production)                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  COMPUTE (EKS)                                                   │
│  ├─ General Node Group (m5.xlarge x 6): $1,166.40              │
│  ├─ Compute Node Group (c5.2xlarge x 3): $1,051.20             │
│  ├─ Memory Node Group (r5.xlarge x 2): $511.20                 │
│  ├─ GPU Node Group (g4dn.xlarge x 0): $0 (on-demand)           │
│  └─ Subtotal: $2,728.80                                         │
│                                                                  │
│  DATABASE (Aurora PostgreSQL)                                    │
│  ├─ 3 x db.r5.xlarge instances: $1,752.00                      │
│  ├─ Storage (100 GB): $23.00                                   │
│  ├─ I/O requests: $50.00                                       │
│  └─ Subtotal: $1,825.00                                         │
│                                                                  │
│  CACHE (ElastiCache Redis)                                       │
│  ├─ 3 x cache.r5.large (Cluster mode): $1,095.00               │
│  └─ Subtotal: $1,095.00                                         │
│                                                                  │
│  MESSAGE QUEUE (MSK Kafka)                                       │
│  ├─ 3 x kafka.m5.large brokers: $876.00                        │
│  ├─ Storage (1 TB): $115.00                                    │
│  └─ Subtotal: $991.00                                           │
│                                                                  │
│  STORAGE (S3)                                                    │
│  ├─ Documents (500 GB): $11.50                                 │
│  ├─ Backups (2 TB): $46.00                                     │
│  ├─ Logs (100 GB): $2.30                                       │
│  ├─ Frontend (10 GB): $0.23                                    │
│  └─ Subtotal: $60.03                                            │
│                                                                  │
│  NETWORKING                                                      │
│  ├─ NAT Gateway (3 x $32.40): $97.20                           │
│  ├─ Data Transfer (10 TB): $900.00                             │
│  ├─ CloudFront (1 TB): $85.00                                  │
│  ├─ ALB (2 x $16.20): $32.40                                   │
│  └─ Subtotal: $1,114.60                                         │
│                                                                  │
│  SECURITY                                                        │
│  ├─ WAF: $5.00 + $1 per million requests: $50.00               │
│  ├─ KMS Keys (10 keys): $1.00                                  │
│  ├─ Secrets Manager (100 secrets): $4.00                       │
│  ├─ GuardDuty: $50.00                                          │
│  └─ Subtotal: $110.00                                           │
│                                                                  │
│  MONITORING                                                      │
│  ├─ CloudWatch Logs (100 GB): $25.00                           │
│  ├─ CloudWatch Metrics: $10.00                                 │
│  ├─ X-Ray Traces: $20.00                                       │
│  └─ Subtotal: $55.00                                            │
│                                                                  │
│  CI/CD & DEVELOPMENT                                             │
│  ├─ CodeBuild (5000 min): $7.50                                │
│  ├─ CodePipeline: $1.00                                        │
│  ├─ ECR Storage (100 GB): $10.00                               │
│  └─ Subtotal: $18.50                                            │
│                                                                  │
│  OTHER SERVICES                                                  │
│  ├─ Route 53 (Hosted Zone): $0.50                              │
│  ├─ ACM Certificates: $0.00 (free)                             │
│  ├─ CloudTrail: $2.00                                          │
│  ├─ Config: $5.00                                              │
│  └─ Subtotal: $7.50                                             │
│                                                                  │
│  ─────────────────────────────────────────────────────────────  │
│  TOTAL MONTHLY COST: $8,005.43                                  │
│  ─────────────────────────────────────────────────────────────  │
│                                                                  │
│  ANNUAL COST: $96,065.16                                        │
│  COST PER USER (1000 users): $8.01/month                        │
│  COST PER TRANSACTION (1M txns): $0.008                         │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 14.2 Cost Optimization Strategies

```yaml
# Cost Optimization Recommendations

Reserved Instances (40-60% savings):
  - EKS Nodes: 1-year reserved for baseline capacity
  - Aurora: Reserved instances for database
  - ElastiCache: Reserved for cache nodes
  
  Estimated Savings: $3,500/month (44%)

Spot Instances (60-70% savings):
  - Use for non-critical workloads
  - CI/CD runners
  - Batch processing jobs
  - ML training
  
  Estimated Savings: $500/month (6%)

Right-Sizing:
  - Monitor actual usage vs allocated
  - Downsize over-provisioned resources
  - Use AWS Compute Optimizer recommendations
  
  Estimated Savings: $800/month (10%)

Auto-Scaling:
  - Scale down during off-peak hours
  - Use Karpenter for efficient node provisioning
  - Implement HPA for all services
  
  Estimated Savings: $600/month (7.5%)

Storage Optimization:
  - Lifecycle policies for S3
  - Delete unused snapshots
  - Compress logs before storage
  
  Estimated Savings: $100/month (1.25%)

Total Potential Savings: $5,500/month (69%)
Optimized Monthly Cost: $2,505.43
```

### 14.3 Cost Allocation Tags

```hcl
# Tagging Strategy for Cost Allocation
resource "aws_cost_allocation_tag" "project" {
  tag_key      = "Project"
  tag_values   = ["revenue-management"]
  status       = "Active"
}

resource "aws_cost_allocation_tag" "environment" {
  tag_key      = "Environment"
  tag_values   = ["production", "staging", "development"]
  status       = "Active"
}

resource "aws_cost_allocation_tag" "team" {
  tag_key      = "Team"
  tag_values   = ["engineering", "data", "devops"]
  status       = "Active"
}

resource "aws_cost_allocation_tag" "cost_center" {
  tag_key      = "CostCenter"
  tag_values   = ["CC-1001"]
  status       = "Active"
}

# Apply tags to all resources
locals {
  common_tags = {
    Project     = "revenue-management"
    Environment = var.environment
    Team        = "engineering"
    CostCenter  = "CC-1001"
    ManagedBy   = "terraform"
    CreatedAt   = timestamp()
  }
}
```

### 14.4 Budget Alerts

```hcl
# AWS Budget
resource "aws_budgets_budget" "monthly" {
  name         = "revenue-mgmt-monthly-budget"
  budget_type  = "COST"
  limit_amount = "10000"
  limit_unit   = "USD"
  time_unit    = "MONTHLY"

  cost_filter {
    name   = "TagKeyValue"
    values = ["user:Project$revenue-management"]
  }

  notification {
    comparison_operator        = "GREATER_THAN"
    threshold                  = 80
    threshold_type             = "PERCENTAGE"
    notification_type          = "FORECASTED"
    subscriber_email_addresses = ["finance@company.com", "devops@company.com"]
  }

  notification {
    comparison_operator        = "GREATER_THAN"
    threshold                  = 100
    threshold_type             = "PERCENTAGE"
    notification_type          = "ACTUAL"
    subscriber_email_addresses = ["finance@company.com", "devops@company.com", "cto@company.com"]
  }
}
```

---

## ภาคผนวก

### ภาคผนวก A: Infrastructure Checklist

```yaml
Pre-Deployment Checklist:
  Network:
    - [ ] VPC created with proper CIDR
    - [ ] Subnets created across 3 AZs
    - [ ] Route tables configured
    - [ ] NAT Gateways deployed
    - [ ] Security groups configured
    - [ ] Network ACLs configured
    - [ ] VPC endpoints created
  
  Compute:
    - [ ] EKS cluster created
    - [ ] Node groups configured
    - [ ] IAM roles created
    - [ ] Service accounts configured
  
  Database:
    - [ ] Aurora cluster created
    - [ ] Subnet group configured
    - [ ] Parameter group configured
    - [ ] Backup enabled
    - [ ] Monitoring enabled
  
  Cache:
    - [ ] ElastiCache cluster created
    - [ ] Subnet group configured
    - [ ] Parameter group configured
  
  Message Queue:
    - [ ] MSK cluster created
    - [ ] Topics created
    - [ ] IAM roles configured
  
  Storage:
    - [ ] S3 buckets created
    - [ ] Versioning enabled
    - [ ] Encryption enabled
    - [ ] Lifecycle policies configured
  
  Security:
    - [ ] KMS keys created
    - [ ] Secrets Manager configured
    - [ ] WAF configured
    - [ ] SSL certificates created
  
  Monitoring:
    - [ ] Prometheus deployed
    - [ ] Grafana deployed
    - [ ] Alerting configured
    - [ ] Dashboards created
  
  CI/CD:
    - [ ] GitHub Actions configured
    - [ ] ArgoCD deployed
    - [ ] ECR repositories created
  
  Backup:
    - [ ] Aurora backups enabled
    - [ ] Velero deployed
    - [ ] Backup schedules configured
    - [ ] DR plan documented
```

### ภาคผนวก B: Disaster Recovery Runbook

```markdown
# Disaster Recovery Runbook

## Scenario 1: Database Failure

### Detection
- CloudWatch alarm: DatabaseDown
- Application health check failures
- Increased error rates

### Response
1. Assess the situation
2. If automated failover didn't trigger:
   - Manually promote read replica
   - Update DNS/endpoint
3. Verify application connectivity
4. Monitor for data consistency
5. Create new read replica
6. Document incident

### RTO: 15 minutes (automated), 30 minutes (manual)
### RPO: 0 (synchronous replication)

## Scenario 2: EKS Cluster Failure

### Detection
- kubectl commands failing
- Application pods not running
- Monitoring alerts

### Response
1. Check AWS EKS console
2. If cluster is unreachable:
   - Create new EKS cluster in different AZ
   - Deploy applications from Git (ArgoCD)
   - Restore database from backup if needed
3. Update DNS to new cluster
4. Verify all services
5. Document incident

### RTO: 2 hours
### RPO: 0 (GitOps)

## Scenario 3: Region Failure

### Detection
- AWS Service Health Dashboard
- Multiple service failures
- Network connectivity issues

### Response
1. Declare disaster
2. Activate DR region (Tokyo)
3. Restore database from cross-region snapshot
4. Deploy applications via ArgoCD
5. Update DNS (Route 53 failover)
6. Verify all services
7. Communicate with stakeholders
8. Document incident

### RTO: 4 hours
### RPO: 1 hour
```

### ภาคผนวก C: Terraform Module Structure

```
terraform/
├── modules/
│   ├── vpc/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   └── versions.tf
│   ├── eks/
│   ├── rds/
│   ├── elasticache/
│   ├── msk/
│   ├── s3/
│   ├── ecr/
│   ├── iam/
│   ├── kms/
│   ├── waf/
│   ├── cloudfront/
│   ├── route53/
│   └── monitoring/
├── environments/
│   ├── production/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── terraform.tfvars
│   │   └── backend.tf
│   ├── staging/
│   └── development/
├── scripts/
│   ├── init.sh
│   ├── plan.sh
│   ├── apply.sh
│   └── destroy.sh
├── .terraform.lock.hcl
├── terraform.tfvars.example
└── README.md
```

---

**เอกสารนี้จัดทำขึ้นเพื่อใช้เป็นแนวทางในการออกแบบ Cloud Infrastructure สำหรับระบบ Revenue Management**

**เวอร์ชัน:** 1.0  
**วันที่:** 22 มิถุนายน 2026  
**สถานะ:** Draft for Review  
**ผู้จัดทำ:** Infrastructure Team
</think>

# Infrastructure Design

**เวอร์ชัน:** 1.0  
**วันที่:** 22 มิถุนายน 2026  
**สถานะ:** Draft  
**เอกสารที่เกี่ยวข้อง:** `03-Software-Design-Specification.md`, `02-System-Architecture-Document.md`

---

## 📋 สารบัญ

1. [Cloud Provider](#1-cloud-provider)
2. [Network Topology](#2-network-topology)
3. [VPC](#3-vpc)
4. [Subnets](#4-subnets)
5. [Load Balancer](#5-load-balancer)
6. [CDN](#6-cdn)
7. [Object Storage](#7-object-storage)
8. [Container Registry](#8-container-registry)
9. [Kubernetes Cluster](#9-kubernetes-cluster)
10. [Secrets Management](#10-secrets-management)
11. [CI/CD](#11-cicd)
12. [Monitoring](#12-monitoring)
13. [Backup Strategy](#13-backup-strategy)
14. [Cost Estimation](#14-cost-estimation)

---

## 1. Cloud Provider

### 1.1 Provider Selection

**Selected Provider: Amazon Web Services (AWS)**

| Criteria | AWS | Azure | GCP | Decision Factor |
|----------|-----|-------|-----|-----------------|
| Market Share | 32% | 23% | 11% | ✅ Ecosystem maturity |
| Kubernetes Support | EKS (Excellent) | AKS (Good) | GKE (Excellent) | ✅ EKS with Fargate |
| Managed Kafka | MSK (Excellent) | Event Hubs (Good) | Pub/Sub (Good) | ✅ MSK for Kafka |
| Managed Redis | ElastiCache (Excellent) | Cache for Redis (Good) | Memorystore (Good) | ✅ ElastiCache |
| Managed PostgreSQL | Aurora/RDS (Excellent) | Flexible Server (Good) | Cloud SQL (Good) | ✅ Aurora PostgreSQL |
| AI/ML Services | SageMaker (Excellent) | Azure ML (Good) | Vertex AI (Good) | ✅ SageMaker |
| Global Presence | 33 Regions | 60+ Regions | 36 Regions | ✅ APAC coverage |
| Compliance | SOC2, ISO27001, GDPR | SOC2, ISO27001, GDPR | SOC2, ISO27001, GDPR | ✅ All compliant |
| Cost | Medium | Medium | Low-Medium | ️ Competitive |

### 1.2 Multi-Cloud Strategy

```
┌─────────────────────────────────────────────────────────────────┐
│                    MULTI-CLOUD STRATEGY                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  PRIMARY: AWS (Production)                                       │
│  ├─ Region: ap-southeast-1 (Singapore)                          │
│  ├─ Availability Zones: 3 AZs                                   │
│  └─ Services: EKS, RDS, ElastiCache, MSK, S3                   │
│                                                                  │
│  DISASTER RECOVERY: AWS (Secondary Region)                       │
│  ├─ Region: ap-northeast-1 (Tokyo)                              │
│  ├─ Strategy: Pilot Light                                       │
│  └─ RTO: 4 hours, RPO: 1 hour                                   │
│                                                                  │
│  DEVELOPMENT/STAGING: AWS                                        │
│  ├─ Region: ap-southeast-1                                      │
│  └─ Separate AWS Account (Dev Account)                          │
│                                                                  │
│  FUTURE CONSIDERATION: GCP (AI/ML Workloads)                     │
│  ├─ Use Case: Vertex AI for advanced ML                         │
│  └─ Integration: Via MuleSoft or Direct API                     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 1.3 AWS Account Structure

```
AWS Organization
├── Management Account (111111111111)
│   └── AWS Organizations, Billing, CloudTrail
│
├── Security Account (222222222222)
│   ├── GuardDuty
│   ├── Security Hub
│   ├── IAM Identity Center
│   └── KMS Keys
│
├── Shared Services Account (333333333333)
│   ├── ECR (Container Registry)
│   ├── CodePipeline
│   ├── CodeBuild
│   └── Shared VPC Endpoints
│
├── Production Account (444444444444)
│   ├── EKS Cluster
│   ├── RDS Aurora
│   ├── ElastiCache
│   ├── MSK
│   ├── S3 Buckets
│   └── CloudFront
│
├── Staging Account (555555555555)
│   ── Mirror of Production (smaller scale)
│
└── Development Account (666666666666)
    └── Developer environments
```

### 1.4 Infrastructure as Code (IaC)

```yaml
# Tool Selection
Primary IaC: Terraform (OpenTofu)
  Version: 1.7+
  State Backend: S3 + DynamoDB (locking)
  Modules: Custom modules in private registry
  
Secondary IaC: AWS CDK (TypeScript)
  Use Case: Complex EKS configurations
  Language: TypeScript
  
Configuration Management: Ansible
  Use Case: EC2 instance configuration
  Playbooks: Stored in Git

Policy as Code: OPA (Open Policy Agent)
  Use Case: Kubernetes policies, IAM policies
  
GitOps: ArgoCD
  Use Case: Kubernetes application deployment
  Repository: Git-based manifests
```

---

## 2. Network Topology

### 2.1 High-Level Network Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    INTERNET                                       │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
─────────────────────────────────────────────────────────────────┐
│              AWS GLOBAL ACCELERATOR (Optional)                   │
│         - Static IP addresses                                   │
│         - Edge locations worldwide                              │
│         - Health checks and failover                            │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
─────────────────────────────────────────────────────────────────┐
│                    CLOUDFRONT CDN                                │
│         - Static assets (React frontend)                        │
│         - API caching                                           │
│         - DDoS protection (AWS Shield)                          │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────
│              WEB APPLICATION FIREWALL (WAF)                      │
│         - OWASP Top 10 protection                               │
│         - Rate limiting                                         │
│         - IP reputation lists                                   │
│         - Geo-blocking                                          │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
─────────────────────────────────────────────────────────────────┐
│                  PUBLIC SUBNET (DMZ)                             │
│  ┌──────────────────┐  ┌──────────────────┐                    │
│  │   ALB (Internet) │  │   NAT Gateway    │                    │
│  │   - React App    │  │   - Outbound     │                    │
│  │   - API Gateway  │  │     Traffic      │                    │
│  └──────────────────┘  └──────────────────┘                    │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│               PRIVATE SUBNET (Application)                       │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │              EKS CLUSTER (Kubernetes)                     │  │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐  │  │
│  │  │ NestJS   │ │   Go     │ │  Python  │ │  Redis   │  │  │
│  │  │ Services │ │ Services │ │ Services │ │  Proxy   │  │  │
│  │  └──────────┘ ──────────┘ └──────────┘ ──────────┘  │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│              ISOLATED SUBNET (Data Layer)                        │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐      │
│  │  Aurora  │  │  MSK     │  │  Elastic │  │   S3     │      │
│  │PostgreSQL│  │ (Kafka)  │  │  Cache   │  │ Endpoints│      │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘      │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 Network Security Groups

```hcl
# Security Group: ALB (Public)
resource "aws_security_group" "alb" {
  name        = "revenue-mgmt-alb-sg"
  description = "Security group for Application Load Balancer"
  vpc_id      = aws_vpc.main.id

  ingress {
    description = "HTTPS from Internet"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    description = "HTTP from Internet (redirect to HTTPS)"
    from_port   = 80
    to_port     = 80
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

# Security Group: EKS Nodes (Private)
resource "aws_security_group" "eks_nodes" {
  name        = "revenue-mgmt-eks-nodes-sg"
  description = "Security group for EKS worker nodes"
  vpc_id      = aws_vpc.main.id

  # Allow traffic from ALB
  ingress {
    description = "From ALB"
    from_port   = 30000
    to_port     = 32768
    protocol    = "tcp"
    security_groups = [aws_security_group.alb.id]
  }

  # Allow inter-node communication
  ingress {
    description = "From EKS nodes"
    from_port   = 0
    to_port     = 65535
    protocol    = "tcp"
    self        = true
  }

  # Allow SSH from bastion (if needed)
  ingress {
    description = "SSH from Bastion"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    security_groups = [aws_security_group.bastion.id]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# Security Group: Aurora PostgreSQL (Isolated)
resource "aws_security_group" "aurora" {
  name        = "revenue-mgmt-aurora-sg"
  description = "Security group for Aurora PostgreSQL"
  vpc_id      = aws_vpc.main.id

  ingress {
    description = "PostgreSQL from EKS"
    from_port   = 5432
    to_port     = 5432
    protocol    = "tcp"
    security_groups = [aws_security_group.eks_nodes.id]
  }

  # No egress needed (database doesn't initiate connections)
}

# Security Group: ElastiCache Redis (Isolated)
resource "aws_security_group" "redis" {
  name        = "revenue-mgmt-redis-sg"
  description = "Security group for ElastiCache Redis"
  vpc_id      = aws_vpc.main.id

  ingress {
    description = "Redis from EKS"
    from_port   = 6379
    to_port     = 6379
    protocol    = "tcp"
    security_groups = [aws_security_group.eks_nodes.id]
  }
}

# Security Group: MSK Kafka (Isolated)
resource "aws_security_group" "msk" {
  name        = "revenue-mgmt-msk-sg"
  description = "Security group for MSK Kafka"
  vpc_id      = aws_vpc.main.id

  ingress {
    description = "Kafka from EKS"
    from_port   = 9098
    to_port     = 9098
    protocol    = "tcp"
    security_groups = [aws_security_group.eks_nodes.id]
  }

  ingress {
    description = "Kafka inter-broker"
    from_port   = 9094
    to_port     = 9094
    protocol    = "tcp"
    self        = true
  }
}
```

### 2.3 VPC Endpoints (PrivateLink)

```hcl
# S3 Gateway Endpoint (Free)
resource "aws_vpc_endpoint" "s3" {
  vpc_id       = aws_vpc.main.id
  service_name = "com.amazonaws.ap-southeast-1.s3"
  route_table_ids = [
    aws_route_table.private_app.id,
    aws_route_table.private_data.id,
  ]
}

# ECR Interface Endpoint
resource "aws_vpc_endpoint" "ecr" {
  vpc_id              = aws_vpc.main.id
  service_name        = "com.amazonaws.ap-southeast-1.ecr.api"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = aws_subnet.private_app[*].id
  security_group_ids  = [aws_security_group.vpc_endpoints.id]
  private_dns_enabled = true
}

# ECR DKR Interface Endpoint
resource "aws_vpc_endpoint" "ecr_dkr" {
  vpc_id              = aws_vpc.main.id
  service_name        = "com.amazonaws.ap-southeast-1.ecr.dkr"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = aws_subnet.private_app[*].id
  security_group_ids  = [aws_security_group.vpc_endpoints.id]
  private_dns_enabled = true
}

# Secrets Manager Interface Endpoint
resource "aws_vpc_endpoint" "secrets_manager" {
  vpc_id              = aws_vpc.main.id
  service_name        = "com.amazonaws.ap-southeast-1.secretsmanager"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = aws_subnet.private_app[*].id
  security_group_ids  = [aws_security_group.vpc_endpoints.id]
  private_dns_enabled = true
}

# CloudWatch Logs Interface Endpoint
resource "aws_vpc_endpoint" "cloudwatch_logs" {
  vpc_id              = aws_vpc.main.id
  service_name        = "com.amazonaws.ap-southeast-1.logs"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = aws_subnet.private_app[*].id
  security_group_ids  = [aws_security_group.vpc_endpoints.id]
  private_dns_enabled = true
}
```

---

## 3. VPC

### 3.1 VPC Configuration

```hcl
# Main VPC
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name        = "revenue-mgmt-vpc"
    Environment = "production"
    Project     = "revenue-management"
    ManagedBy   = "terraform"
  }
}

# VPC Flow Logs
resource "aws_flow_log" "main" {
  iam_role_arn    = aws_iam_role.vpc_flow_log.arn
  log_destination = aws_cloudwatch_log_group.vpc_flow_log.arn
  traffic_type    = "ALL"
  vpc_id          = aws_vpc.main.id

  tags = {
    Name = "revenue-mgmt-vpc-flow-log"
  }
}

# Internet Gateway
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "revenue-mgmt-igw"
  }
}

# NAT Gateways (One per AZ for high availability)
resource "aws_eip" "nat" {
  count  = 3
  domain = "vpc"

  tags = {
    Name = "revenue-mgmt-nat-eip-${count.index + 1}"
  }
}

resource "aws_nat_gateway" "main" {
  count         = 3
  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = aws_subnet.public[count.index].id

  tags = {
    Name = "revenue-mgmt-nat-${count.index + 1}"
  }

  depends_on = [aws_internet_gateway.main]
}
```

### 3.2 VPC CIDR Allocation

```
VPC CIDR: 10.0.0.0/16 (65,536 IPs)

┌─────────────────────────────────────────────────────────────────┐
│  CIDR Block          │  Purpose              │  IPs Available  │
├─────────────────────────────────────────────────────────────────┤
│  10.0.0.0/24         │  Reserved             │  256            │
│  10.0.1.0/24         │  Public Subnet AZ-1   │  256            │
│  10.0.2.0/24         │  Public Subnet AZ-2   │  256            │
│  10.0.3.0/24         │  Public Subnet AZ-3   │  256            │
│  10.0.10.0/24        │  Private App AZ-1     │  256            │
│  10.0.20.0/24        │  Private App AZ-2     │  256            │
│  10.0.30.0/24        │  Private App AZ-3     │  256            │
│  10.0.40.0/24        │  Reserved (Future)    │  256            │
│  10.0.50.0/24        │  Reserved (Future)    │  256            │
│  10.0.100.0/24       │  Private Data AZ-1    │  256            │
│  10.0.200.0/24       │  Private Data AZ-2    │  256            │
│  10.0.300.0/24       │  Private Data AZ-3    │  256            │
│  10.0.110.0/24       │  Isolated DB AZ-1     │  256            │
│  10.0.210.0/24       │  Isolated DB AZ-2     │  256            │
│  10.0.310.0/24       │  Isolated DB AZ-3     │  256            │
│  10.0.120.0/24       │  Reserved (Future)    │  256            │
│  10.0.220.0/24       │  Reserved (Future)    │  256            │
│  10.0.320.0/24       │  Reserved (Future)    │  256            │
│  10.0.200.0/20       │  EKS Pod CIDR         │  4,096          │
│  10.0.220.0/22       │  EKS Service CIDR     │  1,024          │
└─────────────────────────────────────────────────────────────────┘

Total Allocated: ~7,168 IPs (11% of VPC)
Reserved for Growth: ~58,368 IPs (89% of VPC)
```

### 3.3 Route Tables

```hcl
# Public Route Table
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }

  tags = {
    Name = "revenue-mgmt-public-rt"
  }
}

# Private App Route Table (via NAT)
resource "aws_route_table" "private_app" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.main[0].id
  }

  tags = {
    Name = "revenue-mgmt-private-app-rt"
  }
}

# Private Data Route Table (via NAT, restricted)
resource "aws_route_table" "private_data" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.main[0].id
  }

  # No route to IGW (isolated from internet)

  tags = {
    Name = "revenue-mgmt-private-data-rt"
  }
}

# Route Table Associations
resource "aws_route_table_association" "public" {
  count          = 3
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

resource "aws_route_table_association" "private_app" {
  count          = 3
  subnet_id      = aws_subnet.private_app[count.index].id
  route_table_id = aws_route_table.private_app.id
}

resource "aws_route_table_association" "private_data" {
  count          = 3
  subnet_id      = aws_subnet.private_data[count.index].id
  route_table_id = aws_route_table.private_data.id
}
```

---

## 4. Subnets

### 4.1 Subnet Configuration

```hcl
# Availability Zones
data "aws_availability_zones" "available" {
  state = "available"
}

# Public Subnets (ALB, NAT Gateway)
resource "aws_subnet" "public" {
  count                   = 3
  vpc_id                  = aws_vpc.main.id
  cidr_block              = cidrsubnet(aws_vpc.main.cidr_block, 8, count.index + 1)
  availability_zone       = data.aws_availability_zones.available.names[count.index]
  map_public_ip_on_launch = true

  tags = {
    Name                                            = "revenue-mgmt-public-${count.index + 1}"
    "kubernetes.io/role/elb"                        = "1"
    "kubernetes.io/cluster/revenue-mgmt-eks"        = "shared"
  }
}

# Private Application Subnets (EKS Nodes)
resource "aws_subnet" "private_app" {
  count             = 3
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(aws_vpc.main.cidr_block, 8, count.index + 10)
  availability_zone = data.aws_availability_zones.available.names[count.index]

  tags = {
    Name                                            = "revenue-mgmt-private-app-${count.index + 1}"
    "kubernetes.io/role/internal-elb"               = "1"
    "kubernetes.io/cluster/revenue-mgmt-eks"        = "shared"
  }
}

# Private Data Subnets (ElastiCache, MSK)
resource "aws_subnet" "private_data" {
  count             = 3
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(aws_vpc.main.cidr_block, 8, count.index + 100)
  availability_zone = data.aws_availability_zones.available.names[count.index]

  tags = {
    Name = "revenue-mgmt-private-data-${count.index + 1}"
  }
}

# Isolated Database Subnets (Aurora)
resource "aws_subnet" "isolated_db" {
  count             = 3
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(aws_vpc.main.cidr_block, 8, count.index + 110)
  availability_zone = data.aws_availability_zones.available.names[count.index]

  tags = {
    Name = "revenue-mgmt-isolated-db-${count.index + 1}"
  }
}
```

### 4.2 Subnet Usage Matrix

| Subnet Type | CIDR | AZ-1 | AZ-2 | AZ-3 | Purpose | Services |
|-------------|------|------|------|------|---------|----------|
| **Public** | 10.0.1-3.0/24 | ✅ | ✅ | ✅ | Internet-facing | ALB, NAT Gateway |
| **Private App** | 10.0.10-30.0/24 | ✅ | ✅ | ✅ | Application | EKS Nodes, Bastion |
| **Private Data** | 10.0.100-300.0/24 | ✅ | ✅ | ✅ | Data Services | ElastiCache, MSK |
| **Isolated DB** | 10.0.110-310.0/24 | ✅ | ✅ | ✅ | Database | Aurora PostgreSQL |

### 4.3 Network ACLs

```hcl
# Network ACL: Public Subnet
resource "aws_network_acl" "public" {
  vpc_id     = aws_vpc.main.id
  subnet_ids = aws_subnet.public[*].id

  # Inbound Rules
  ingress {
    rule_no    = 100
    protocol   = "tcp"
    rule_action = "allow"
    cidr_block = "0.0.0.0/0"
    from_port  = 443
    to_port    = 443
  }

  ingress {
    rule_no    = 110
    protocol   = "tcp"
    rule_action = "allow"
    cidr_block = "0.0.0.0/0"
    from_port  = 80
    to_port    = 80
  }

  ingress {
    rule_no    = 120
    protocol   = "tcp"
    rule_action = "allow"
    cidr_block = "0.0.0.0/0"
    from_port  = 1024
    to_port    = 65535
  }

  # Outbound Rules
  egress {
    rule_no    = 100
    protocol   = "-1"
    rule_action = "allow"
    cidr_block = "0.0.0.0/0"
    from_port  = 0
    to_port    = 0
  }

  tags = {
    Name = "revenue-mgmt-public-nacl"
  }
}

# Network ACL: Private Subnet (Restrictive)
resource "aws_network_acl" "private" {
  vpc_id     = aws_vpc.main.id
  subnet_ids = concat(
    aws_subnet.private_app[*].id,
    aws_subnet.private_data[*].id,
    aws_subnet.isolated_db[*].id
  )

  # Inbound Rules
  ingress {
    rule_no    = 100
    protocol   = "tcp"
    rule_action = "allow"
    cidr_block = "10.0.0.0/16"  # VPC internal only
    from_port  = 0
    to_port    = 65535
  }

  ingress {
    rule_no    = 110
    protocol   = "tcp"
    rule_action = "allow"
    cidr_block = "10.0.0.0/16"
    from_port  = 1024
    to_port    = 65535
  }

  # Outbound Rules
  egress {
    rule_no    = 100
    protocol   = "tcp"
    rule_action = "allow"
    cidr_block = "0.0.0.0/0"
    from_port  = 443
    to_port    = 443
  }

  egress {
    rule_no    = 110
    protocol   = "tcp"
    rule_action = "allow"
    cidr_block = "0.0.0.0/0"
    from_port  = 80
    to_port    = 80
  }

  egress {
    rule_no    = 120
    protocol   = "tcp"
    rule_action = "allow"
    cidr_block = "10.0.0.0/16"
    from_port  = 0
    to_port    = 65535
  }

  tags = {
    Name = "revenue-mgmt-private-nacl"
  }
}
```

---

## 5. Load Balancer

### 5.1 Load Balancer Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    LOAD BALANCER LAYERS                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  LAYER 1: CloudFront (Global CDN)                                │
│  ├─ Purpose: Static assets, API caching, DDoS protection        │
│  ├─ Type: CDN                                                    │
│  └─ SSL: AWS Certificate Manager (ACM)                          │
│                                                                  │
│  LAYER 2: Application Load Balancer (ALB)                        │
│  ├─ Purpose: HTTP/HTTPS traffic routing                         │
│  ├─ Type: Layer 7 (Application)                                  │
│  ├─ Target Groups:                                               │
│  │  ├─ React Frontend (S3/CloudFront origin)                    │
│  │  ├─ NestJS API Gateway (EKS)                                 │
│  │  └─ Health Check Endpoints                                   │
│  └─ Features:                                                    │
│     ├─ SSL termination                                           │
│     ├─ WAF integration                                           │
│     ├─ Sticky sessions (if needed)                               │
│     └─ Path-based routing                                        │
│                                                                  │
│  LAYER 3: Network Load Balancer (NLB)                            │
│  ├─ Purpose: High-performance, low-latency traffic              │
│  ├─ Type: Layer 4 (Transport)                                    │
│  ├─ Target Groups:                                               │
│  │  ├─ Go Services (gRPC)                                       │
│  │  ├─ High-throughput APIs                                     │
│  │  └─ WebSocket connections                                    │
│  └─ Features:                                                    │
│     ├─ Static IP addresses                                       │
│     ├─ TLS termination                                           │
│     ├─ Connection draining                                       │
│     └─ Cross-zone load balancing                                 │
│                                                                  │
│  LAYER 4: Internal Load Balancer                                 │
│  ├─ Purpose: Inter-service communication                        │
│  ├─ Type: Internal ALB/NLB                                       │
│  ├─ Target Groups:                                               │
│  │  ├─ Service Mesh (Istio/Linkerd)                             │
│  │  └─ Direct service-to-service                                │
│  └─ Features:                                                    │
│     ├─ VPC-only access                                           │
│     ├─ mTLS (via service mesh)                                   │
│     └─ No internet exposure                                      │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 5.2 ALB Configuration

```hcl
# Application Load Balancer
resource "aws_lb" "main" {
  name               = "revenue-mgmt-alb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb.id]
  subnets            = aws_subnet.public[*].id

  enable_deletion_protection = true
  enable_http2               = true
  enable_cross_zone_load_balancing = true

  access_logs {
    bucket  = aws_s3_bucket.alb_logs.id
    prefix  = "alb-logs"
    enabled = true
  }

  tags = {
    Name        = "revenue-mgmt-alb"
    Environment = "production"
  }
}

# HTTPS Listener
resource "aws_lb_listener" "https" {
  load_balancer_arn = aws_lb.main.arn
  port              = 443
  protocol          = "HTTPS"
  ssl_policy        = "ELBSecurityPolicy-TLS13-1-2-2021-06"
  certificate_arn   = aws_acm_certificate.main.arn

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.api.arn
  }
}

# HTTP to HTTPS Redirect
resource "aws_lb_listener" "http" {
  load_balancer_arn = aws_lb.main.arn
  port              = 80
  protocol          = "HTTP"

  default_action {
    type = "redirect"

    redirect {
      port        = "443"
      protocol    = "HTTPS"
      status_code = "HTTP_301"
    }
  }
}

# Target Group: API (NestJS)
resource "aws_lb_target_group" "api" {
  name        = "revenue-mgmt-api"
  port        = 3000
  protocol    = "HTTP"
  vpc_id      = aws_vpc.main.id
  target_type = "ip"

  health_check {
    enabled             = true
    healthy_threshold   = 2
    interval            = 30
    matcher             = "200"
    path                = "/health"
    port                = "traffic-port"
    protocol            = "HTTP"
    timeout             = 5
    unhealthy_threshold = 3
  }

  stickiness {
    type    = "lb_cookie"
    enabled = false
  }

  deregistration_delay = 30

  tags = {
    Name = "revenue-mgmt-api-tg"
  }
}

# Target Group: React Frontend (if served from EKS)
resource "aws_lb_target_group" "frontend" {
  name        = "revenue-mgmt-frontend"
  port        = 80
  protocol    = "HTTP"
  vpc_id      = aws_vpc.main.id
  target_type = "ip"

  health_check {
    enabled             = true
    healthy_threshold   = 2
    interval            = 30
    matcher             = "200"
    path                = "/"
    port                = "traffic-port"
    protocol            = "HTTP"
    timeout             = 5
    unhealthy_threshold = 3
  }

  tags = {
    Name = "revenue-mgmt-frontend-tg"
  }
}

# Listener Rules: Path-based routing
resource "aws_lb_listener_rule" "api" {
  listener_arn = aws_lb_listener.https.arn
  priority     = 100

  action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.api.arn
  }

  condition {
    path_pattern {
      values = ["/api/*", "/graphql", "/health"]
    }
  }
}

resource "aws_lb_listener_rule" "frontend" {
  listener_arn = aws_lb_listener.https.arn
  priority     = 200

  action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.frontend.arn
  }

  condition {
    path_pattern {
      values = ["/*"]
    }
  }
}
```

### 5.3 NLB Configuration (for Go Services/gRPC)

```hcl
# Network Load Balancer (for gRPC services)
resource "aws_lb" "grpc" {
  name               = "revenue-mgmt-grpc-nlb"
  internal           = true
  load_balancer_type = "network"
  subnets            = aws_subnet.private_app[*].id

  enable_deletion_protection = true
  enable_cross_zone_load_balancing = true

  tags = {
    Name = "revenue-mgmt-grpc-nlb"
  }
}

# TLS Listener for gRPC
resource "aws_lb_listener" "grpc" {
  load_balancer_arn = aws_lb.grpc.arn
  port              = 9090
  protocol          = "TLS"
  ssl_policy        = "ELBSecurityPolicy-TLS13-1-2-2021-06"
  certificate_arn   = aws_acm_certificate.grpc.arn

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.grpc.arn
  }
}

# Target Group: gRPC Services
resource "aws_lb_target_group" "grpc" {
  name        = "revenue-mgmt-grpc"
  port        = 9090
  protocol    = "TLS"
  vpc_id      = aws_vpc.main.id
  target_type = "ip"

  health_check {
    enabled             = true
    healthy_threshold   = 2
    interval            = 30
    matcher             = "0"
    path                = "/grpc.health.v1.Health/Check"
    port                = "traffic-port"
    protocol            = "HTTPS"
    timeout             = 5
    unhealthy_threshold = 3
  }

  tags = {
    Name = "revenue-mgmt-grpc-tg"
  }
}
```

### 5.4 WAF Configuration

```hcl
# Web Application Firewall
resource "aws_wafv2_web_acl" "main" {
  name        = "revenue-mgmt-waf"
  scope       = "REGIONAL"
  description = "WAF for Revenue Management ALB"

  default_action {
    allow {}
  }

  # AWS Managed Rules
  rule {
    name     = "AWSManagedRulesCommonRuleSet"
    priority = 1

    override_action {
      none {}
    }

    statement {
      managed_rule_group_statement {
        name        = "AWSManagedRulesCommonRuleSet"
        vendor_name = "AWS"
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "AWSManagedRulesCommonRuleSet"
      sampled_requests_enabled   = true
    }
  }

  rule {
    name     = "AWSManagedRulesKnownBadInputsRuleSet"
    priority = 2

    override_action {
      none {}
    }

    statement {
      managed_rule_group_statement {
        name        = "AWSManagedRulesKnownBadInputsRuleSet"
        vendor_name = "AWS"
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "AWSManagedRulesKnownBadInputsRuleSet"
      sampled_requests_enabled   = true
    }
  }

  # Rate Limiting Rule
  rule {
    name     = "RateLimitRule"
    priority = 3

    action {
      block {}
    }

    statement {
      rate_based_statement {
        limit              = 2000
        aggregate_key_type = "IP"
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "RateLimitRule"
      sampled_requests_enabled   = true
    }
  }

  # Geo-blocking Rule (optional)
  rule {
    name     = "GeoBlockRule"
    priority = 4

    action {
      block {}
    }

    statement {
      geo_match_statement {
        country_codes = ["CN", "RU", "KP"]  # Block high-risk countries
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "GeoBlockRule"
      sampled_requests_enabled   = true
    }
  }

  visibility_config {
    cloudwatch_metrics_enabled = true
    metric_name                = "revenue-mgmt-waf"
    sampled_requests_enabled   = true
  }

  tags = {
    Name = "revenue-mgmt-waf"
  }
}

# Associate WAF with ALB
resource "aws_wafv2_web_acl_association" "alb" {
  resource_arn = aws_lb.main.arn
  web_acl_arn  = aws_wafv2_web_acl.main.arn
}
```

---

## 6. CDN

### 6.1 CloudFront Configuration

```hcl
# CloudFront Distribution
resource "aws_cloudfront_distribution" "main" {
  origin {
    domain_name = aws_s3_bucket.frontend.bucket_regional_domain_name
    origin_id   = "S3-frontend"

    s3_origin_config {
      origin_access_identity = aws_cloudfront_origin_access_identity.frontend.iam_arn
    }
  }

  origin {
    domain_name = aws_lb.main.dns_name
    origin_id   = "ALB-api"

    custom_origin_config {
      http_port              = 80
      https_port             = 443
      origin_protocol_policy = "https-only"
      origin_ssl_protocols   = ["TLSv1.2"]
    }
  }

  # Cache Behavior: Static Assets (S3)
  ordered_cache_behavior {
    path_pattern     = "/static/*"
    allowed_methods  = ["GET", "HEAD", "OPTIONS"]
    cached_methods   = ["GET", "HEAD"]
    target_origin_id = "S3-frontend"

    forwarded_values {
      query_string = false
      cookies {
        forward = "none"
      }
    }

    min_ttl     = 0
    default_ttl = 86400    # 24 hours
    max_ttl     = 31536000 # 1 year

    compress               = true
    viewer_protocol_policy = "redirect-to-https"
  }

  # Cache Behavior: API (ALB)
  ordered_cache_behavior {
    path_pattern     = "/api/*"
    allowed_methods  = ["DELETE", "GET", "HEAD", "OPTIONS", "PATCH", "POST", "PUT"]
    cached_methods   = ["GET", "HEAD"]
    target_origin_id = "ALB-api"

    forwarded_values {
      query_string = true
      headers      = ["Authorization", "Content-Type", "X-Correlation-Id"]
      cookies {
        forward = "none"
      }
    }

    min_ttl     = 0
    default_ttl = 0      # No caching for API
    max_ttl     = 0

    compress               = true
    viewer_protocol_policy = "redirect-to-https"
  }

  # Default Cache Behavior: React App
  default_cache_behavior {
    allowed_methods  = ["DELETE", "GET", "HEAD", "OPTIONS", "PATCH", "POST", "PUT"]
    cached_methods   = ["GET", "HEAD"]
    target_origin_id = "S3-frontend"

    forwarded_values {
      query_string = true
      headers      = ["Authorization", "Content-Type"]
      cookies {
        forward = "none"
      }
    }

    min_ttl     = 0
    default_ttl = 3600     # 1 hour
    max_ttl     = 86400    # 24 hours

    compress               = true
    viewer_protocol_policy = "redirect-to-https"
  }

  # Restrictions
  restrictions {
    geo_restriction {
      restriction_type = "none"
    }
  }

  # Viewer Certificate
  viewer_certificate {
    acm_certificate_arn      = aws_acm_certificate.cloudfront.arn
    ssl_support_method       = "sni-only"
    minimum_protocol_version = "TLSv1.2_2021"
  }

  # WAF Association
  web_acl_id = aws_wafv2_web_acl.cloudfront.arn

  # Logging
  logging_config {
    include_cookies = false
    bucket          = aws_s3_bucket.cloudfront_logs.bucket_domain_name
    prefix          = "cloudfront-logs/"
  }

  enabled             = true
  is_ipv6_enabled     = true
  comment             = "Revenue Management CDN"
  default_root_object = "index.html"

  tags = {
    Name = "revenue-mgmt-cloudfront"
  }
}

# Origin Access Identity for S3
resource "aws_cloudfront_origin_access_identity" "frontend" {
  comment = "OAI for Revenue Management Frontend"
}
```

### 6.2 CDN Caching Strategy

```yaml
# Caching Rules by Content Type

Static Assets (JS, CSS, Images):
  Path: /static/*
  TTL: 1 year (31536000 seconds)
  Cache Key: URL only
  Compression: Brotli + Gzip
  Invalidation: Version-based filenames (app.[hash].js)

API Responses:
  Path: /api/*
  TTL: 0 (No caching)
  Cache Key: URL + Query String + Authorization Header
  Compression: Gzip
  Note: Use Cache-Control headers from backend

React App (HTML):
  Path: /*
  TTL: 1 hour (3600 seconds)
  Cache Key: URL only
  Compression: Gzip
  Invalidation: On deployment

GraphQL:
  Path: /graphql
  TTL: 0 (No caching)
  Cache Key: URL + Query + Variables
  Compression: Gzip
  Note: Implement persisted queries for caching

Health Check:
  Path: /health
  TTL: 0 (No caching)
  Cache Key: URL only
  Compression: None
```

### 6.3 CloudFront Functions

```javascript
// CloudFront Function: Add Security Headers
function handler(event) {
  var response = event.response;
  var headers = response.headers;

  // Security Headers
  headers['strict-transport-security'] = {
    value: 'max-age=63072000; includeSubDomains; preload'
  };
  
  headers['x-content-type-options'] = {
    value: 'nosniff'
  };
  
  headers['x-frame-options'] = {
    value: 'DENY'
  };
  
  headers['x-xss-protection'] = {
    value: '1; mode=block'
  };
  
  headers['referrer-policy'] = {
    value: 'strict-origin-when-cross-origin'
  };
  
  headers['content-security-policy'] = {
    value: "default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline'; img-src 'self' data: https:; font-src 'self' data:; connect-src 'self' https://*.amazonaws.com;"
  };

  return response;
}

// CloudFront Function: Redirect www to apex
function redirectHandler(event) {
  var request = event.request;
  var host = request.headers.host.value;
  
  if (host.startsWith('www.')) {
    var newHost = host.slice(4);
    return {
      statusCode: 301,
      statusDescription: 'Moved Permanently',
      headers: {
        'location': { value: 'https://' + newHost + request.uri }
      }
    };
  }
  
  return request;
}
```

---

## 7. Object Storage

### 7.1 S3 Bucket Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    S3 BUCKETS ARCHITECTURE                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. FRONTEND BUCKET (Static Website)                             │
│     ├─ Bucket: revenue-mgmt-frontend-prod                        │
│     ├─ Purpose: React application static files                   │
│     ├─ Versioning: Enabled                                       │
│     ├─ Encryption: SSE-S3                                        │
│     ├─ Access: CloudFront OAI only                               │
│     └─ Lifecycle: None (permanent)                               │
│                                                                  │
│  2. DOCUMENTS BUCKET (Business Documents)                        │
│     ├─ Bucket: revenue-mgmt-documents-prod                       │
│     ├─ Purpose: Contracts, invoices, proposals                   │
│     ├─ Versioning: Enabled                                       │
│     ├─ Encryption: SSE-KMS                                       │
│     ├─ Access: EKS services via IAM roles                        │
│     └─ Lifecycle: Transition to Glacier after 7 years            │
│                                                                  │
│  3. BACKUP BUCKET (Database Backups)                             │
│     ├─ Bucket: revenue-mgmt-backups-prod                         │
│     ├─ Purpose: RDS snapshots, application backups               │
│     ├─ Versioning: Enabled                                       │
│     ├─ Encryption: SSE-KMS                                       │
│     ├─ Access: Restricted (backup service only)                  │
│     └─ Lifecycle: Delete after 35 days                           │
│                                                                  │
│  4. LOGS BUCKET (Centralized Logs)                               │
│     ├─ Bucket: revenue-mgmt-logs-prod                            │
│     ├─ Purpose: ALB logs, CloudFront logs, VPC flow logs         │
│     ├─ Versioning: Suspended                                     │
│     ├─ Encryption: SSE-S3                                        │
│     ├─ Access: Logging services only                             │
│     └─ Lifecycle: Delete after 90 days                           │
│                                                                  │
│  5. ARTIFACTS BUCKET (CI/CD)                                     │
│     ├─ Bucket: revenue-mgmt-artifacts-prod                       │
│     ├─ Purpose: Build artifacts, deployment packages             │
│     ├─ Versioning: Enabled                                       │
│     ├─ Encryption: SSE-S3                                        │
│     ├─ Access: CI/CD pipeline only                               │
│     └─ Lifecycle: Delete after 30 days                           │
│                                                                  │
│  6. TEMP BUCKET (Temporary Processing)                           │
│     ├─ Bucket: revenue-mgmt-temp-prod                            │
│     ├─ Purpose: Temporary files during processing                │
│     ├─ Versioning: Suspended                                     │
│     ├─ Encryption: SSE-S3                                        │
│     ├─ Access: EKS services via IAM roles                        │
│     └─ Lifecycle: Delete after 7 days                            │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 7.2 S3 Bucket Configuration

```hcl
# Documents Bucket (Main business bucket)
resource "aws_s3_bucket" "documents" {
  bucket = "revenue-mgmt-documents-${var.environment}"

  tags = {
    Name        = "revenue-mgmt-documents"
    Environment = var.environment
  }
}

# Versioning
resource "aws_s3_bucket_versioning" "documents" {
  bucket = aws_s3_bucket.documents.id

  versioning_configuration {
    status = "Enabled"
  }
}

# Server-side encryption
resource "aws_s3_bucket_server_side_encryption_configuration" "documents" {
  bucket = aws_s3_bucket.documents.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = aws_kms_key.documents.arn
    }
    bucket_key_enabled = true
  }
}

# Block public access
resource "aws_s3_bucket_public_access_block" "documents" {
  bucket = aws_s3_bucket.documents.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# Lifecycle configuration
resource "aws_s3_bucket_lifecycle_configuration" "documents" {
  bucket = aws_s3_bucket.documents.id

  rule {
    id     = "transition-to-glacier"
    status = "Enabled"

    transition {
      days          = 2555 # 7 years
      storage_class = "GLACIER"
    }

    noncurrent_version_transition {
      noncurrent_days = 90
      storage_class   = "GLACIER"
    }

    noncurrent_version_expiration {
      noncurrent_days = 365
    }
  }
}

# CORS configuration (if needed for direct browser uploads)
resource "aws_s3_bucket_cors_configuration" "documents" {
  bucket = aws_s3_bucket.documents.id

  cors_rule {
    allowed_headers = ["*"]
    allowed_methods = ["GET", "PUT", "POST"]
    allowed_origins = ["https://app.revenuemanagement.com"]
    expose_headers  = ["ETag"]
    max_age_seconds = 3000
  }
}

# Bucket policy (restrict access)
resource "aws_s3_bucket_policy" "documents" {
  bucket = aws_s3_bucket.documents.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid       = "DenyInsecureTransport"
        Effect    = "Deny"
        Principal = "*"
        Action    = "s3:*"
        Resource  = [
          aws_s3_bucket.documents.arn,
          "${aws_s3_bucket.documents.arn}/*"
        ]
        Condition = {
          Bool = {
            "aws:SecureTransport" = "false"
          }
        }
      },
      {
        Sid       = "AllowEKSServices"
        Effect    = "Allow"
        Principal = {
          AWS = aws_iam_role.eks_services.arn
        }
        Action = [
          "s3:GetObject",
          "s3:PutObject",
          "s3:DeleteObject"
        ]
        Resource = "${aws_s3_bucket.documents.arn}/*"
      }
    ]
  })
}

# KMS Key for encryption
resource "aws_kms_key" "documents" {
  description             = "KMS key for Revenue Management documents"
  deletion_window_in_days = 30
  enable_key_rotation     = true

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "Enable IAM User Permissions"
        Effect = "Allow"
        Principal = {
          AWS = "arn:aws:iam::${var.aws_account_id}:root"
        }
        Action   = "kms:*"
        Resource = "*"
      },
      {
        Sid    = "Allow EKS Services"
        Effect = "Allow"
        Principal = {
          AWS = aws_iam_role.eks_services.arn
        }
        Action = [
          "kms:Encrypt",
          "kms:Decrypt",
          "kms:GenerateDataKey"
        ]
        Resource = "*"
      }
    ]
  })

  tags = {
    Name = "revenue-mgmt-documents-key"
  }
}
```

### 7.3 S3 Access Patterns

```typescript
// NestJS - S3 Service for document management
@Injectable()
export class DocumentService {
  private readonly s3: S3Client;
  private readonly bucketName: string;

  constructor(
    private readonly configService: ConfigService,
  ) {
    this.s3 = new S3Client({
      region: this.configService.get('AWS_REGION'),
    });
    this.bucketName = this.configService.get('S3_DOCUMENTS_BUCKET');
  }

  async uploadDocument(
    file: Buffer,
    key: string,
    contentType: string,
    metadata: Record<string, string>,
  ): Promise<string> {
    const command = new PutObjectCommand({
      Bucket: this.bucketName,
      Key: key,
      Body: file,
      ContentType: contentType,
      Metadata: metadata,
      ServerSideEncryption: 'aws:kms',
    });

    await this.s3.send(command);
    return `s3://${this.bucketName}/${key}`;
  }

  async getDocument(key: string): Promise<Buffer> {
    const command = new GetObjectCommand({
      Bucket: this.bucketName,
      Key: key,
    });

    const response = await this.s3.send(command);
    return response.Body.transformToByteArray();
  }

  async generatePresignedUrl(
    key: string,
    expiresIn: number = 3600,
  ): Promise<string> {
    const command = new GetObjectCommand({
      Bucket: this.bucketName,
      Key: key,
    });

    return getSignedUrl(this.s3, command, { expiresIn });
  }

  async deleteDocument(key: string): Promise<void> {
    const command = new DeleteObjectCommand({
      Bucket: this.bucketName,
      Key: key,
    });

    await this.s3.send(command);
  }
}
```

---

## 8. Container Registry

### 8.1 ECR Configuration

```hcl
# ECR Repository: API (NestJS)
resource "aws_ecr_repository" "api" {
  name                 = "revenue-mgmt/api"
  image_tag_mutability = "MUTABLE"
  force_delete         = true

  image_scanning_configuration {
    scan_on_push = true
  }

  encryption_configuration {
    encryption_type = "KMS"
    kms_key         = aws_kms_key.ecr.arn
  }

  tags = {
    Name = "revenue-mgmt-api"
  }
}

# ECR Repository: Go Services
resource "aws_ecr_repository" "go_services" {
  name                 = "revenue-mgmt/go-services"
  image_tag_mutability = "MUTABLE"
  force_delete         = true

  image_scanning_configuration {
    scan_on_push = true
  }

  encryption_configuration {
    encryption_type = "KMS"
    kms_key         = aws_kms_key.ecr.arn
  }

  tags = {
    Name = "revenue-mgmt-go-services"
  }
}

# ECR Repository: Python Services
resource "aws_ecr_repository" "python_services" {
  name                 = "revenue-mgmt/python-services"
  image_tag_mutability = "MUTABLE"
  force_delete         = true

  image_scanning_configuration {
    scan_on_push = true
  }

  encryption_configuration {
    encryption_type = "KMS"
    kms_key         = aws_kms_key.ecr.arn
  }

  tags = {
    Name = "revenue-mgmt-python-services"
  }
}

# ECR Repository: Frontend (React)
resource "aws_ecr_repository" "frontend" {
  name                 = "revenue-mgmt/frontend"
  image_tag_mutability = "MUTABLE"
  force_delete         = true

  image_scanning_configuration {
    scan_on_push = true
  }

  encryption_configuration {
    encryption_type = "KMS"
    kms_key         = aws_kms_key.ecr.arn
  }

  tags = {
    Name = "revenue-mgmt-frontend"
  }
}

# Lifecycle Policy: Keep last 30 images
resource "aws_ecr_lifecycle_policy" "api" {
  repository = aws_ecr_repository.api.name

  policy = jsonencode({
    rules = [
      {
        rulePriority = 1
        description  = "Keep last 30 images"
        selection = {
          tagStatus   = "any"
          countType   = "imageCountMoreThan"
          countNumber = 30
        }
        action = {
          type = "expire"
        }
      }
    ]
  })
}

# KMS Key for ECR encryption
resource "aws_kms_key" "ecr" {
  description             = "KMS key for ECR encryption"
  deletion_window_in_days = 30
  enable_key_rotation     = true

  tags = {
    Name = "revenue-mgmt-ecr-key"
  }
}
```

### 8.2 Docker Image Strategy

```yaml
# Multi-stage Dockerfile Example (NestJS API)

# Stage 1: Build
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Stage 2: Production
FROM node:18-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/package.json ./

# Security: Run as non-root user
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nestjs -u 1001
USER nestjs

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD node -e "require('http').get('http://localhost:3000/health', (r) => {process.exit(r.statusCode === 200 ? 0 : 1)})"

EXPOSE 3000
CMD ["node", "dist/main"]

# Image Tagging Strategy
Tags:
  - latest: Latest stable release
  - v1.2.3: Semantic version
  - sha-abc123: Git commit SHA
  - build-456: Build number
  
# Image Scanning
  - Scan on push: Enabled
  - Vulnerability threshold: HIGH
  - Block deployment if: CRITICAL vulnerabilities found
```

### 8.3 ECR Access Policy

```hcl
# IAM Policy for EKS to pull images
resource "aws_iam_policy" "ecr_pull" {
  name        = "revenue-mgmt-ecr-pull"
  description = "Allow EKS to pull images from ECR"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "ecr:GetDownloadUrlForLayer",
          "ecr:BatchGetImage",
          "ecr:DescribeImages"
        ]
        Resource = [
          aws_ecr_repository.api.arn,
          aws_ecr_repository.go_services.arn,
          aws_ecr_repository.python_services.arn,
          aws_ecr_repository.frontend.arn
        ]
      },
      {
        Effect = "Allow"
        Action = [
          "ecr:GetAuthorizationToken"
        ]
        Resource = "*"
      }
    ]
  })
}
```

---

## 9. Kubernetes Cluster

### 9.1 EKS Cluster Configuration

```hcl
# EKS Cluster
resource "aws_eks_cluster" "main" {
  name     = "revenue-mgmt-eks"
  role_arn = aws_iam_role.eks_cluster.arn
  version  = "1.29"

  vpc_config {
    subnet_ids              = concat(
      aws_subnet.private_app[*].id,
      aws_subnet.private_data[*].id
    )
    endpoint_private_access = true
    endpoint_public_access  = true
    public_access_cidrs     = ["10.0.0.0/16"]  # Restrict to VPC
  }

  enabled_cluster_log_types = [
    "api",
    "audit",
    "authenticator",
    "controllerManager",
    "scheduler"
  ]

  encryption_config {
    provider {
      key_arn = aws_kms_key.eks.arn
    }
    resources = ["secrets"]
  }

  tags = {
    Name        = "revenue-mgmt-eks"
    Environment = "production"
  }

  depends_on = [
    aws_iam_role_policy_attachment.eks_cluster_policy,
    aws_iam_role_policy_attachment.eks_vpc_controller_policy,
  ]
}

# EKS Node Groups

# General Purpose Node Group (NestJS, Python services)
resource "aws_eks_node_group" "general" {
  cluster_name    = aws_eks_cluster.main.name
  node_group_name = "general"
  node_role_arn   = aws_iam_role.eks_nodes.arn
  subnet_ids      = aws_subnet.private_app[*].id

  scaling_config {
    desired_size = 6
    max_size     = 12
    min_size     = 3
  }

  instance_types = ["m5.xlarge"]  # 4 vCPU, 16 GB RAM

  capacity_type = "ON_DEMAND"

  labels = {
    role = "general"
  }

  taint {
    key    = "dedicated"
    value  = "general"
    effect = "NO_SCHEDULE"
  }

  update_config {
    max_unavailable = 1
  }

  tags = {
    Name = "revenue-mgmt-eks-general"
  }
}

# Compute Optimized Node Group (Go services, high throughput)
resource "aws_eks_node_group" "compute" {
  cluster_name    = aws_eks_cluster.main.name
  node_group_name = "compute-optimized"
  node_role_arn   = aws_iam_role.eks_nodes.arn
  subnet_ids      = aws_subnet.private_app[*].id

  scaling_config {
    desired_size = 3
    max_size     = 6
    min_size     = 2
  }

  instance_types = ["c5.2xlarge"]  # 8 vCPU, 16 GB RAM

  capacity_type = "ON_DEMAND"

  labels = {
    role = "compute"
  }

  taint {
    key    = "dedicated"
    value  = "compute"
    effect = "NO_SCHEDULE"
  }

  tags = {
    Name = "revenue-mgmt-eks-compute"
  }
}

# Memory Optimized Node Group (Python AI/ML, caching)
resource "aws_eks_node_group" "memory" {
  cluster_name    = aws_eks_cluster.main.name
  node_group_name = "memory-optimized"
  node_role_arn   = aws_iam_role.eks_nodes.arn
  subnet_ids      = aws_subnet.private_app[*].id

  scaling_config {
    desired_size = 2
    max_size     = 4
    min_size     = 1
  }

  instance_types = ["r5.xlarge"]  # 4 vCPU, 32 GB RAM

  capacity_type = "ON_DEMAND"

  labels = {
    role = "memory"
  }

  taint {
    key    = "dedicated"
    value  = "memory"
    effect = "NO_SCHEDULE"
  }

  tags = {
    Name = "revenue-mgmt-eks-memory"
  }
}

# GPU Node Group (ML Training - optional, on-demand)
resource "aws_eks_node_group" "gpu" {
  cluster_name    = aws_eks_cluster.main.name
  node_group_name = "gpu"
  node_role_arn   = aws_iam_role.eks_nodes.arn
  subnet_ids      = aws_subnet.private_app[*].id

  scaling_config {
    desired_size = 0  # Start with 0, scale up when needed
    max_size     = 2
    min_size     = 0
  }

  instance_types = ["g4dn.xlarge"]  # GPU instance

  capacity_type = "ON_DEMAND"

  labels = {
    role = "gpu"
  }

  taint {
    key    = "nvidia.com/gpu"
    value  = "present"
    effect = "NO_SCHEDULE"
  }

  tags = {
    Name = "revenue-mgmt-eks-gpu"
  }
}
```

### 9.2 Kubernetes Namespace Structure

```yaml
# Namespace Organization

apiVersion: v1
kind: Namespace
metadata:
  name: revenue-mgmt
  labels:
    name: revenue-mgmt
    environment: production

---
# Additional Namespaces

apiVersion: v1
kind: Namespace
metadata:
  name: monitoring
  labels:
    name: monitoring

---
apiVersion: v1
kind: Namespace
metadata:
  name: ingress-nginx
  labels:
    name: ingress-nginx

---
apiVersion: v1
kind: Namespace
metadata:
  name: cert-manager
  labels:
    name: cert-manager

---
apiVersion: v1
kind: Namespace
metadata:
  name: argocd
  labels:
    name: argocd
```

### 9.3 Application Deployment Manifests

```yaml
# NestJS API Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: revenue-mgmt-api
  namespace: revenue-mgmt
  labels:
    app: revenue-mgmt-api
    tier: backend
spec:
  replicas: 6
  selector:
    matchLabels:
      app: revenue-mgmt-api
  template:
    metadata:
      labels:
        app: revenue-mgmt-api
        tier: backend
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "3000"
        prometheus.io/path: "/metrics"
    spec:
      nodeSelector:
        role: general
      containers:
        - name: api
          image: ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/revenue-mgmt/api:latest
          ports:
            - containerPort: 3000
              name: http
          env:
            - name: NODE_ENV
              value: "production"
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: revenue-mgmt-secrets
                  key: database-url
            - name: REDIS_URL
              valueFrom:
                secretKeyRef:
                  name: revenue-mgmt-secrets
                  key: redis-url
            - name: KAFKA_BROKERS
              valueFrom:
                secretKeyRef:
                  name: revenue-mgmt-secrets
                  key: kafka-brokers
          resources:
            requests:
              memory: "512Mi"
              cpu: "500m"
            limits:
              memory: "1Gi"
              cpu: "1000m"
          readinessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 10
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 30
            periodSeconds: 30
          volumeMounts:
            - name: tmp
              mountPath: /tmp
      volumes:
        - name: tmp
          emptyDir: {}
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchExpressions:
                    - key: app
                      operator: In
                      values:
                        - revenue-mgmt-api
                topologyKey: kubernetes.io/hostname

---
# Go Billing Service Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: billing-service
  namespace: revenue-mgmt
  labels:
    app: billing-service
    tier: backend
spec:
  replicas: 4
  selector:
    matchLabels:
      app: billing-service
  template:
    metadata:
      labels:
        app: billing-service
        tier: backend
    spec:
      nodeSelector:
        role: compute
      containers:
        - name: billing
          image: ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/revenue-mgmt/go-services:billing-latest
          ports:
            - containerPort: 9090
              name: grpc
            - containerPort: 8080
              name: http
          env:
            - name: ENV
              value: "production"
            - name: DB_HOST
              valueFrom:
                secretKeyRef:
                  name: revenue-mgmt-secrets
                  key: db-host
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: revenue-mgmt-secrets
                  key: db-password
          resources:
            requests:
              memory: "1Gi"
              cpu: "1000m"
            limits:
              memory: "2Gi"
              cpu: "2000m"
          readinessProbe:
            grpc:
              port: 9090
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            grpc:
              port: 9090
            initialDelaySeconds: 15
            periodSeconds: 20

---
# Python AI Service Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ai-service
  namespace: revenue-mgmt
  labels:
    app: ai-service
    tier: backend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: ai-service
  template:
    metadata:
      labels:
        app: ai-service
        tier: backend
    spec:
      nodeSelector:
        role: memory
      containers:
        - name: ai
          image: ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/revenue-mgmt/python-services:ai-latest
          ports:
            - containerPort: 8000
              name: http
          env:
            - name: ENVIRONMENT
              value: "production"
            - name: MODEL_PATH
              value: "/models"
          resources:
            requests:
              memory: "2Gi"
              cpu: "1000m"
            limits:
              memory: "4Gi"
              cpu: "2000m"
          volumeMounts:
            - name: model-storage
              mountPath: /models
      volumes:
        - name: model-storage
          persistentVolumeClaim:
            claimName: model-pvc

---
# Horizontal Pod Autoscaler
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: revenue-mgmt-api-hpa
  namespace: revenue-mgmt
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: revenue-mgmt-api
  minReplicas: 6
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second
        target:
          type: AverageValue
          averageValue: 100
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Percent
          value: 50
          periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
        - type: Percent
          value: 100
          periodSeconds: 60
        - type: Pods
          value: 4
          periodSeconds: 60
      selectPolicy: Max
```

### 9.4 Kubernetes Services & Ingress

```yaml
# Internal Service for API
apiVersion: v1
kind: Service
metadata:
  name: revenue-mgmt-api
  namespace: revenue-mgmt
  labels:
    app: revenue-mgmt-api
spec:
  type: ClusterIP
  ports:
    - port: 3000
      targetPort: 3000
      protocol: TCP
      name: http
  selector:
    app: revenue-mgmt-api

---
# Internal Service for Go Billing (gRPC)
apiVersion: v1
kind: Service
metadata:
  name: billing-service
  namespace: revenue-mgmt
  labels:
    app: billing-service
spec:
  type: ClusterIP
  ports:
    - port: 9090
      targetPort: 9090
      protocol: TCP
      name: grpc
    - port: 8080
      targetPort: 8080
      protocol: TCP
      name: http
  selector:
    app: billing-service

---
# Ingress (via NGINX Ingress Controller)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: revenue-mgmt-ingress
  namespace: revenue-mgmt
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "300"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "300"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - api.revenuemanagement.com
      secretName: revenue-mgmt-tls
  rules:
    - host: api.revenuemanagement.com
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: revenue-mgmt-api
                port:
                  number: 3000
          - path: /graphql
            pathType: Exact
            backend:
              service:
                name: revenue-mgmt-api
                port:
                  number: 3000
          - path: /health
            pathType: Exact
            backend:
              service:
                name: revenue-mgmt-api
                port:
                  number: 3000
```

### 9.5 Kubernetes Resource Quotas

```yaml
# Resource Quota for revenue-mgmt namespace
apiVersion: v1
kind: ResourceQuota
metadata:
  name: revenue-mgmt-quota
  namespace: revenue-mgmt
spec:
  hard:
    requests.cpu: "32"
    requests.memory: "64Gi"
    limits.cpu: "64"
    limits.memory: "128Gi"
    pods: "100"
    services: "20"
    persistentvolumeclaims: "10"
    secrets: "50"
    configmaps: "50"

---
# Limit Range for default container limits
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: revenue-mgmt
spec:
  limits:
    - default:
        cpu: "500m"
        memory: "512Mi"
      defaultRequest:
        cpu: "100m"
        memory: "128Mi"
      type: Container
```

---

## 10. Secrets Management

### 10.1 Secrets Architecture

```
─────────────────────────────────────────────────────────────────┐
│                  SECRETS MANAGEMENT ARCHITECTURE                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  AWS Secrets Manager                                             │
│  ├─ Database credentials (Aurora)                               │
│  ├─ API keys (Payment gateways, e-signature)                    │
│  ├─ OAuth client secrets                                        │
│  ├─ TLS certificates (if not using ACM)                         │
│  └─ Encryption keys (application-level)                         │
│                                                                  │
│  AWS Systems Manager Parameter Store                             │
│  ├─ Configuration values (non-sensitive)                        │
│  ├─ Feature flags                                               │
│  ├─ Service endpoints                                           │
│  └─ Application settings                                        │
│                                                                  │
│  Kubernetes Secrets (Encrypted at Rest)                          │
│  ├─ Runtime secrets injected via External Secrets Operator      │
│  ├─ TLS certificates for services                               │
│  └─ Service account tokens                                      │
│                                                                  │
│  HashiCorp Vault (Optional, for advanced use cases)              │
│  ├─ Dynamic database credentials                                │
│  ├─ PKI certificates                                            │
│  └─ Secret rotation automation                                  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 10.2 AWS Secrets Manager Configuration

```hcl
# Database Credentials Secret
resource "aws_secretsmanager_secret" "database" {
  name                    = "revenue-mgmt/database"
  description             = "Aurora PostgreSQL credentials"
  recovery_window_in_days = 30

  tags = {
    Name        = "revenue-mgmt-database"
    Environment = "production"
  }
}

resource "aws_secretsmanager_secret_version" "database" {
  secret_id = aws_secretsmanager_secret.database.id
  secret_string = jsonencode({
    username = "revenue_admin"
    password = random_password.database.result
    engine   = "postgres"
    host     = aws_rds_cluster.aurora.endpoint
    port     = 5432
    dbname   = "revenue_mgmt"
  })
}

# Redis Credentials
resource "aws_secretsmanager_secret" "redis" {
  name = "revenue-mgmt/redis"

  tags = {
    Name = "revenue-mgmt-redis"
  }
}

resource "aws_secretsmanager_secret_version" "redis" {
  secret_id = aws_secretsmanager_secret.redis.id
  secret_string = jsonencode({
    host     = aws_elasticache_cluster.redis.cache_nodes[0].address
    port     = 6379
    password = random_password.redis.result
  })
}

# Kafka Credentials
resource "aws_secretsmanager_secret" "kafka" {
  name = "revenue-mgmt/kafka"

  tags = {
    Name = "revenue-mgmt-kafka"
  }
}

resource "aws_secretsmanager_secret_version" "kafka" {
  secret_id = aws_secretsmanager_secret.kafka.id
  secret_string = jsonencode({
    brokers         = aws_msk_cluster.main.bootstrap_brokers
    brokers_tls     = aws_msk_cluster.main.bootstrap_brokers_tls
    sasl_username   = "kafka-user"
    sasl_password   = random_password.kafka.result
  })
}

# Payment Gateway API Key
resource "aws_secretsmanager_secret" "payment_gateway" {
  name = "revenue-mgmt/payment-gateway"

  tags = {
    Name = "revenue-mgmt-payment-gateway"
  }
}

resource "aws_secretsmanager_secret_version" "payment_gateway" {
  secret_id = aws_secretsmanager_secret.payment_gateway.id
  secret_string = jsonencode({
    stripe_secret_key = var.stripe_secret_key
    stripe_publishable_key = var.stripe_publishable_key
    webhook_secret = var.stripe_webhook_secret
  })
}

# Random Passwords
resource "random_password" "database" {
  length  = 32
  special = true
}

resource "random_password" "redis" {
  length  = 32
  special = false
}

resource "random_password" "kafka" {
  length  = 32
  special = true
}
```

### 10.3 External Secrets Operator (Kubernetes)

```yaml
# ExternalSecret: Database Credentials
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: revenue-mgmt-database
  namespace: revenue-mgmt
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secretsmanager
    kind: ClusterSecretStore
  target:
    name: revenue-mgmt-secrets
    creationPolicy: Owner
  data:
    - secretKey: database-url
      remoteRef:
        key: revenue-mgmt/database
        property: host
    - secretKey: database-username
      remoteRef:
        key: revenue-mgmt/database
        property: username
    - secretKey: database-password
      remoteRef:
        key: revenue-mgmt/database
        property: password

---
# ClusterSecretStore: AWS Secrets Manager
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: aws-secretsmanager
spec:
  provider:
    aws:
      service: SecretsManager
      region: ap-southeast-1
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets-sa
            namespace: revenue-mgmt

---
# ServiceAccount for External Secrets
apiVersion: v1
kind: ServiceAccount
metadata:
  name: external-secrets-sa
  namespace: revenue-mgmt
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::${AWS_ACCOUNT_ID}:role/revenue-mgmt-external-secrets-role

---
# IAM Role for Service Account (IRSA)
resource "aws_iam_role" "external_secrets" {
  name = "revenue-mgmt-external-secrets-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          Federated = "arn:aws:iam::${var.aws_account_id}:oidc-provider/${aws_eks_cluster.main.identity[0].oidc[0].issuer}"
        }
        Action = "sts:AssumeRoleWithWebIdentity"
        Condition = {
          StringEquals = {
            "${aws_eks_cluster.main.identity[0].oidc[0].issuer}:sub" = "system:serviceaccount:revenue-mgmt:external-secrets-sa"
          }
        }
      }
    ]
  })
}

resource "aws_iam_role_policy" "external_secrets" {
  name = "revenue-mgmt-external-secrets-policy"
  role = aws_iam_role.external_secrets.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "secretsmanager:GetSecretValue",
          "secretsmanager:DescribeSecret"
        ]
        Resource = "arn:aws:secretsmanager:${var.aws_region}:${var.aws_account_id}:secret:revenue-mgmt/*"
      }
    ]
  })
}
```

### 10.4 Secret Rotation

```hcl
# Automatic Rotation for Database Credentials
resource "aws_secretsmanager_secret_rotation" "database" {
  secret_id = aws_secretsmanager_secret.database.id

  rotation_lambda_arn = aws_lambda_function.secret_rotation.arn

  rotation_rules {
    automatically_after_days = 90
  }
}

# Lambda Function for Rotation
resource "aws_lambda_function" "secret_rotation" {
  filename         = "secret-rotation.zip"
  function_name    = "revenue-mgmt-secret-rotation"
  role             = aws_iam_role.lambda_rotation.arn
  handler          = "index.handler"
  runtime          = "nodejs18.x"
  timeout          = 300

  environment {
    variables = {
      SECRETS_MANAGER_ENDPOINT = "https://secretsmanager.ap-southeast-1.amazonaws.com"
    }
  }

  vpc_config {
    subnet_ids         = aws_subnet.private_app[*].id
    security_group_ids = [aws_security_group.lambda_rotation.id]
  }

  tags = {
    Name = "revenue-mgmt-secret-rotation"
  }
}
```

---

## 11. CI/CD

### 11.1 CI/CD Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    CI/CD PIPELINE ARCHITECTURE                   │
├─────────────────────────────────────────────────────────────────
│                                                                  │
│  SOURCE CONTROL (GitHub)                                         │
│  ├─ Branch: main (production)                                   │
│  ├─ Branch: develop (staging)                                   │
│  ├─ Branch: feature/* (development)                             │
│  └─ Pull Requests with required reviews                         │
│                                                                  │
│  CI PIPELINE (GitHub Actions)                                    │
│  ├─ Trigger: Push to any branch, PR creation                    │
│  ├─ Steps:                                                       │
│  │  1. Checkout code                                            │
│  │  2. Setup Node.js/Go/Python                                  │
│  │  3. Install dependencies                                     │
│  │  4. Lint (ESLint, golangci-lint, Ruff)                       │
│  │  5. Unit tests with coverage                                 │
│  │  6. Build Docker images                                      │
│  │  7. Security scan (Trivy, Snyk)                              │
│  │  8. Push to ECR                                              │
│  │  9. Update Kubernetes manifests                              │
│  └─ Artifacts: Docker images, test reports                      │
│                                                                  │
│  CD PIPELINE (ArgoCD - GitOps)                                   │
│  ├─ Repository: Kubernetes manifests (Git)                      │
│  ├─ Sync Policy: Automatic (with approval for production)       │
│  ├─ Health Checks: Custom health checks                         │
│  └─ Rollback: Automatic on failure                              │
│                                                                  │
│  DEPLOYMENT STRATEGY                                             │
│  ├─ Development: Direct deployment                              │
│  ├─ Staging: Automatic after CI pass                            │
│  └─ Production: Manual approval + Blue-Green                    │
│                                                                  │
─────────────────────────────────────────────────────────────────┘
```

### 11.2 GitHub Actions Workflow

```yaml
# .github/workflows/ci-cd.yml
name: CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

env:
  AWS_REGION: ap-southeast-1
  ECR_REGISTRY: ${{ secrets.AWS_ACCOUNT_ID }}.dkr.ecr.ap-southeast-1.amazonaws.com
  EKS_CLUSTER_NAME: revenue-mgmt-eks

jobs:
  # Job 1: Lint and Test
  lint-and-test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        service: [api, billing, ai, frontend]
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js
        if: matrix.service == 'api' || matrix.service == 'frontend'
        uses: actions/setup-node@v4
        with:
          node-version: '18'
          cache: 'npm'
      
      - name: Setup Go
        if: matrix.service == 'billing'
        uses: actions/setup-go@v5
        with:
          go-version: '1.21'
      
      - name: Setup Python
        if: matrix.service == 'ai'
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'
      
      - name: Install dependencies
        run: |
          if [ "${{ matrix.service }}" = "api" ] || [ "${{ matrix.service }}" = "frontend" ]; then
            npm ci
          elif [ "${{ matrix.service }}" = "billing" ]; then
            go mod download
          elif [ "${{ matrix.service }}" = "ai" ]; then
            pip install -r requirements.txt
          fi
      
      - name: Lint
        run: |
          if [ "${{ matrix.service }}" = "api" ] || [ "${{ matrix.service }}" = "frontend" ]; then
            npm run lint
          elif [ "${{ matrix.service }}" = "billing" ]; then
            golangci-lint run
          elif [ "${{ matrix.service }}" = "ai" ]; then
            ruff check .
          fi
      
      - name: Run tests
        run: |
          if [ "${{ matrix.service }}" = "api" ] || [ "${{ matrix.service }}" = "frontend" ]; then
            npm run test:coverage
          elif [ "${{ matrix.service }}" = "billing" ]; then
            go test -coverprofile=coverage.out ./...
          elif [ "${{ matrix.service }}" = "ai" ]; then
            pytest --cov=src --cov-report=xml
          fi
      
      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          files: ./coverage.xml
          flags: ${{ matrix.service }}

  # Job 2: Build and Push Docker Images
  build-and-push:
    needs: lint-and-test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main' || github.ref == 'refs/heads/develop'
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}
      
      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2
      
      - name: Build, tag, and push image
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          IMAGE_TAG: ${{ github.sha }}
        run: |
          # Build API image
          docker build -t $ECR_REGISTRY/revenue-mgmt/api:$IMAGE_TAG -f api/Dockerfile ./api
          docker push $ECR_REGISTRY/revenue-mgmt/api:$IMAGE_TAG
          
          # Build Go services
          docker build -t $ECR_REGISTRY/revenue-mgmt/go-services:billing-$IMAGE_TAG -f services/billing/Dockerfile ./services/billing
          docker push $ECR_REGISTRY/revenue-mgmt/go-services:billing-$IMAGE_TAG
          
          # Build Python services
          docker build -t $ECR_REGISTRY/revenue-mgmt/python-services:ai-$IMAGE_TAG -f services/ai/Dockerfile ./services/ai
          docker push $ECR_REGISTRY/revenue-mgmt/python-services:ai-$IMAGE_TAG
          
          # Build Frontend
          docker build -t $ECR_REGISTRY/revenue-mgmt/frontend:$IMAGE_TAG -f frontend/Dockerfile ./frontend
          docker push $ECR_REGISTRY/revenue-mgmt/frontend:$IMAGE_TAG
      
      - name: Update Kubernetes manifests
        run: |
          # Update image tags in k8s manifests
          sed -i "s|image:.*api:.*|image: $ECR_REGISTRY/revenue-mgmt/api:$IMAGE_TAG|g" k8s/api-deployment.yaml
          sed -i "s|image:.*go-services:.*|image: $ECR_REGISTRY/revenue-mgmt/go-services:billing-$IMAGE_TAG|g" k8s/billing-deployment.yaml
          sed -i "s|image:.*python-services:.*|image: $ECR_REGISTRY/revenue-mgmt/python-services:ai-$IMAGE_TAG|g" k8s/ai-deployment.yaml
          sed -i "s|image:.*frontend:.*|image: $ECR_REGISTRY/revenue-mgmt/frontend:$IMAGE_TAG|g" k8s/frontend-deployment.yaml
      
      - name: Commit and push manifests
        run: |
          git config --global user.name 'GitHub Actions'
          git config --global user.email 'actions@github.com'
          git add k8s/
          git commit -m "Update image tags to $IMAGE_TAG"
          git push

  # Job 3: Deploy to Staging
  deploy-staging:
    needs: build-and-push
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/develop'
    environment: staging
    
    steps:
      - name: Deploy to staging
        run: |
          echo "ArgoCD will automatically sync from git repository"
          echo "Staging deployment triggered"

  # Job 4: Deploy to Production (with approval)
  deploy-production:
    needs: build-and-push
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    environment: 
      name: production
      url: https://app.revenuemanagement.com
    
    steps:
      - name: Deploy to production
        run: |
          echo "ArgoCD will automatically sync from git repository"
          echo "Production deployment triggered"
      
      - name: Run smoke tests
        run: |
          # Wait for deployment to complete
          kubectl rollout status deployment/revenue-mgmt-api -n revenue-mgmt --timeout=300s
          
          # Run health check
          curl -f https://api.revenuemanagement.com/health || exit 1
          
          echo "Production deployment successful"
```

### 11.3 ArgoCD Configuration

```yaml
# ArgoCD Application
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: revenue-mgmt
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/your-org/revenue-mgmt-k8s.git
    targetRevision: HEAD
    path: k8s/overlays/production
  destination:
    server: https://kubernetes.default.svc
    namespace: revenue-mgmt
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - PrunePropagationPolicy=foreground
      - PruneLast=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
  revisionHistoryLimit: 10

---
# ArgoCD Project
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: revenue-mgmt
  namespace: argocd
spec:
  description: Revenue Management Application
  sourceRepos:
    - https://github.com/your-org/revenue-mgmt-k8s.git
  destinations:
    - namespace: revenue-mgmt
      server: https://kubernetes.default.svc
    - namespace: monitoring
      server: https://kubernetes.default.svc
  clusterResourceWhitelist:
    - group: '*'
      kind: '*'
  namespaceResourceWhitelist:
    - group: '*'
      kind: '*'
```

### 11.4 Deployment Strategies

```yaml
# Blue-Green Deployment (Production)
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: revenue-mgmt-api
  namespace: revenue-mgmt
spec:
  replicas: 6
  strategy:
    blueGreen:
      activeService: revenue-mgmt-api
      previewService: revenue-mgmt-api-preview
      autoPromotionEnabled: false
      autoPromotionSeconds: 300
      prePromotionAnalysis:
        templates:
          - templateName: smoke-test
      postPromotionAnalysis:
        templates:
          - templateName: canary-analysis
  selector:
    matchLabels:
      app: revenue-mgmt-api
  template:
    metadata:
      labels:
        app: revenue-mgmt-api
    spec:
      containers:
        - name: api
          image: revenue-mgmt/api:latest
          ports:
            - containerPort: 3000

---
# Canary Deployment (Staging)
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: revenue-mgmt-api-canary
  namespace: revenue-mgmt
spec:
  replicas: 6
  strategy:
    canary:
      steps:
        - setWeight: 10
        - pause: {duration: 10m}
        - setWeight: 25
        - pause: {duration: 10m}
        - setWeight: 50
        - pause: {duration: 10m}
        - setWeight: 75
        - pause: {duration: 10m}
        - setWeight: 100
      analysis:
        templates:
          - templateName: success-rate
          - templateName: latency
  selector:
    matchLabels:
      app: revenue-mgmt-api
  template:
    metadata:
      labels:
        app: revenue-mgmt-api
    spec:
      containers:
        - name: api
          image: revenue-mgmt/api:latest
```

---

## 12. Monitoring

### 12.1 Monitoring Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    MONITORING STACK                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  METRICS COLLECTION                                              │
│  ├─ Prometheus (Kubernetes metrics)                             │
│  ├─ Node Exporter (Host metrics)                                │
│  ├─ AWS CloudWatch (AWS service metrics)                        │
│  └─ Custom exporters (Application metrics)                      │
│                                                                  │
│  LOGGING                                                         │
│  ├─ Fluentd/Fluent Bit (Log collection)                         │
│  ├─ Elasticsearch (Log storage)                                 │
│  ├─ Kibana (Log visualization)                                  │
│  └─ CloudWatch Logs (AWS service logs)                          │
│                                                                  │
│  TRACING                                                         │
│  ├─ Jaeger (Distributed tracing)                                │
│  ├─ AWS X-Ray (AWS service tracing)                             │
│  └─ OpenTelemetry (Instrumentation)                             │
│                                                                  │
│  VISUALIZATION & ALERTING                                        │
│  ├─ Grafana (Dashboards)                                        │
│  ├─ Prometheus Alertmanager (Alerting)                          │
│  ├─ PagerDuty (Incident management)                             │
│  └─ Slack (Notifications)                                       │
│                                                                  │
│  APPLICATION PERFORMANCE MONITORING (APM)                        │
│  ├─ New Relic / Datadog (Optional)                              │
│  ├─ Custom metrics via OpenTelemetry                            │
│  └─ Kubernetes metrics via kube-state-metrics                   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 12.2 Prometheus Configuration

```yaml
# Prometheus Helm Values
server:
  replicas: 2
  resources:
    requests:
      memory: "2Gi"
      cpu: "1000m"
    limits:
      memory: "4Gi"
      cpu: "2000m"
  persistentVolume:
    enabled: true
    size: 100Gi
    storageClass: gp3

alertmanager:
  enabled: true
  replicas: 2
  config:
    route:
      group_by: ['alertname', 'cluster', 'service']
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 12h
      receiver: 'pagerduty'
      routes:
        - match:
            severity: 'critical'
          receiver: 'pagerduty'
        - match:
            severity: 'warning'
          receiver: 'slack'

extraScrapeConfigs: |
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)
      - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
        action: replace
        regex: ([^:]+)(?::\d+)?;(\d+)
        replacement: $1:$2
        target_label: __address__
      - action: labelmap
        regex: __meta_kubernetes_pod_label_(.+)
      - source_labels: [__meta_kubernetes_namespace]
        action: replace
        target_label: kubernetes_namespace
      - source_labels: [__meta_kubernetes_pod_name]
        action: replace
        target_label: kubernetes_pod_name

  - job_name: 'kubernetes-services'
    kubernetes_sd_configs:
      - role: service
    metrics_path: /probe
    params:
      module: [http_2xx]
    relabel_configs:
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_probe]
        action: keep
        regex: true
      - source_labels: [__address__]
        target_label: __param_target
      - target_label: __address__
        replacement: blackbox-exporter:9115
      - source_labels: [__param_target]
        target_label: instance
      - action: labelmap
        regex: __meta_kubernetes_service_label_(.+)
      - source_labels: [__meta_kubernetes_namespace]
        target_label: kubernetes_namespace
      - source_labels: [__meta_kubernetes_service_name]
        target_label: kubernetes_name
```

### 12.3 Grafana Dashboards

```yaml
# Grafana Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana
  namespace: monitoring
spec:
  replicas: 2
  selector:
    matchLabels:
      app: grafana
  template:
    metadata:
      labels:
        app: grafana
    spec:
      containers:
        - name: grafana
          image: grafana/grafana:10.3.0
          ports:
            - containerPort: 3000
          env:
            - name: GF_SECURITY_ADMIN_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: grafana-secrets
                  key: admin-password
            - name: GF_INSTALL_PLUGINS
              value: "grafana-piechart-panel,grafana-worldmap-panel"
          resources:
            requests:
              memory: "512Mi"
              cpu: "250m"
            limits:
              memory: "1Gi"
              cpu: "500m"
          volumeMounts:
            - name: grafana-storage
              mountPath: /var/lib/grafana
            - name: grafana-dashboards
              mountPath: /var/lib/grafana/dashboards
      volumes:
        - name: grafana-storage
          persistentVolumeClaim:
            claimName: grafana-pvc
        - name: grafana-dashboards
          configMap:
            name: grafana-dashboards

---
# Grafana Dashboards ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-dashboards
  namespace: monitoring
data:
  revenue-mgmt-overview.json: |
    {
      "dashboard": {
        "title": "Revenue Management Overview",
        "panels": [
          {
            "title": "API Request Rate",
            "type": "graph",
            "targets": [
              {
                "expr": "rate(http_requests_total{job=\"revenue-mgmt-api\"}[5m])",
                "legendFormat": "{{method}} {{endpoint}}"
              }
            ]
          },
          {
            "title": "Invoice Generation Rate",
            "type": "stat",
            "targets": [
              {
                "expr": "rate(invoice_generated_total[1h])",
                "legendFormat": "Invoices/hour"
              }
            ]
          },
          {
            "title": "Database Connection Pool",
            "type": "gauge",
            "targets": [
              {
                "expr": "pg_stat_activity_count{datname=\"revenue_mgmt\"}",
                "legendFormat": "Active connections"
              }
            ]
          },
          {
            "title": "Kafka Consumer Lag",
            "type": "graph",
            "targets": [
              {
                "expr": "kafka_consumer_group_lag{group=\"billing-consumer\"}",
                "legendFormat": "{{topic}}-{{partition}}"
              }
            ]
          }
        ]
      }
    }
```

### 12.4 Alerting Rules

```yaml
# Prometheus Alerting Rules
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-alerts
  namespace: monitoring
data:
  revenue-mgmt-alerts.yaml: |
    groups:
      - name: revenue-mgmt-alerts
        rules:
          # Critical Alerts
          - alert: HighErrorRate
            expr: |
              sum(rate(http_requests_total{job="revenue-mgmt-api", status=~"5.."}[5m])) 
              / 
              sum(rate(http_requests_total{job="revenue-mgmt-api"}[5m])) > 0.05
            for: 5m
            labels:
              severity: critical
              team: revenue-mgmt
            annotations:
              summary: "High error rate detected"
              description: "Error rate is {{ $value | humanizePercentage }} for the last 5 minutes"
              runbook_url: "https://wiki.internal/runbooks/high-error-rate"

          - alert: DatabaseDown
            expr: pg_up{job="postgres"} == 0
            for: 1m
            labels:
              severity: critical
              team: revenue-mgmt
            annotations:
              summary: "Database is down"
              description: "PostgreSQL instance {{ $labels.instance }} is down"

          - alert: HighCPUUsage
            expr: |
              sum(rate(container_cpu_usage_seconds_total{namespace="revenue-mgmt"}[5m])) 
              / 
              sum(container_spec_cpu_quota{namespace="revenue-mgmt"} / container_spec_cpu_period{namespace="revenue-mgmt"}) > 0.9
            for: 10m
            labels:
              severity: critical
              team: revenue-mgmt
            annotations:
              summary: "High CPU usage"
              description: "CPU usage is {{ $value | humanizePercentage }} for the last 10 minutes"

          - alert: HighMemoryUsage
            expr: |
              sum(container_memory_working_set_bytes{namespace="revenue-mgmt"}) 
              / 
              sum(container_spec_memory_limit_bytes{namespace="revenue-mgmt"}) > 0.9
            for: 10m
            labels:
              severity: critical
              team: revenue-mgmt
            annotations:
              summary: "High memory usage"
              description: "Memory usage is {{ $value | humanizePercentage }}"

          - alert: KafkaConsumerLag
            expr: kafka_consumer_group_lag{group="billing-consumer"} > 10000
            for: 15m
            labels:
              severity: warning
              team: revenue-mgmt
            annotations:
              summary: "High Kafka consumer lag"
              description: "Consumer lag is {{ $value }} for group {{ $labels.group }}"

          - alert: InvoiceGenerationSlow
            expr: |
              histogram_quantile(0.95, 
                rate(invoice_generation_duration_seconds_bucket[5m])
              ) > 30
            for: 10m
            labels:
              severity: warning
              team: revenue-mgmt
            annotations:
              summary: "Invoice generation is slow"
              description: "95th percentile invoice generation time is {{ $value }}s"

          - alert: DiskSpaceLow
            expr: |
              (node_filesystem_avail_bytes{mountpoint="/var/lib/postgresql"} 
              / 
              node_filesystem_size_bytes{mountpoint="/var/lib/postgresql"}) < 0.1
            for: 5m
            labels:
              severity: critical
              team: revenue-mgmt
            annotations:
              summary: "Low disk space"
              description: "Disk space is {{ $value | humanizePercentage }} on {{ $labels.instance }}"

          # Warning Alerts
          - alert: HighLatency
            expr: |
              histogram_quantile(0.95, 
                rate(http_request_duration_seconds_bucket{job="revenue-mgmt-api"}[5m])
              ) > 2
            for: 10m
            labels:
              severity: warning
              team: revenue-mgmt
            annotations:
              summary: "High API latency"
              description: "95th percentile latency is {{ $value }}s"

          - alert: PodRestarting
            expr: |
              increase(kube_pod_container_status_restarts_total{namespace="revenue-mgmt"}[1h]) > 5
            for: 5m
            labels:
              severity: warning
              team: revenue-mgmt
            annotations:
              summary: "Pod restarting frequently"
              description: "Pod {{ $labels.pod }} has restarted {{ $value }} times in the last hour"
```

### 12.5 Distributed Tracing (Jaeger)

```yaml
# Jaeger Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jaeger
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: jaeger
  template:
    metadata:
      labels:
        app: jaeger
    spec:
      containers:
        - name: jaeger
          image: jaegertracing/all-in-one:1.53
          ports:
            - containerPort: 16686
              name: ui
            - containerPort: 14268
              name: collector
            - containerPort: 6831
              name: agent-compact
          env:
            - name: COLLECTOR_OTLP_ENABLED
              value: "true"
            - name: SPAN_STORAGE_TYPE
              value: "elasticsearch"
            - name: ES_SERVER_URLS
              value: "http://elasticsearch:9200"
          resources:
            requests:
              memory: "1Gi"
              cpu: "500m"
            limits:
              memory: "2Gi"
              cpu: "1000m"

---
# OpenTelemetry Collector
apiVersion: apps/v1
kind: Deployment
metadata:
  name: otel-collector
  namespace: monitoring
spec:
  replicas: 2
  selector:
    matchLabels:
      app: otel-collector
  template:
    metadata:
      labels:
        app: otel-collector
    spec:
      containers:
        - name: otel-collector
          image: otel/opentelemetry-collector:0.91.0
          ports:
            - containerPort: 4317
              name: otlp-grpc
            - containerPort: 4318
              name: otlp-http
            - containerPort: 8888
              name: metrics
          volumeMounts:
            - name: otel-config
              mountPath: /etc/otelcol/config.yaml
              subPath: config.yaml
      volumes:
        - name: otel-config
          configMap:
            name: otel-collector-config

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: otel-collector-config
  namespace: monitoring
data:
  config.yaml: |
    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317
          http:
            endpoint: 0.0.0.0:4318

    processors:
      batch:
        send_batch_size: 10000
        timeout: 10s
      memory_limiter:
        limit_mib: 1500
        spike_limit_mib: 500
        check_interval: 5s

    exporters:
      jaeger:
        endpoint: jaeger:14250
        tls:
          insecure: true
      prometheus:
        endpoint: 0.0.0.0:8889

    service:
      pipelines:
        traces:
          receivers: [otlp]
          processors: [memory_limiter, batch]
          exporters: [jaeger]
        metrics:
          receivers: [otlp]
          processors: [memory_limiter, batch]
          exporters: [prometheus]
```

---

## 13. Backup Strategy

### 13.1 Backup Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    BACKUP STRATEGY                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  DATABASE BACKUPS (Aurora PostgreSQL)                            │
│  ├─ Automated Backups: Enabled (35 days retention)              │
│  ├─ Point-in-Time Recovery: Enabled                             │
│  ├─ Cross-Region Snapshots: Daily to ap-northeast-1             │
│  ├─ Manual Snapshots: Before major changes                      │
│  └─ Backup Verification: Weekly restore test                    │
│                                                                  │
│  REDIS BACKUPS (ElastiCache)                                     │
│  ├─ Automated Backups: Enabled (7 days retention)               │
│  ├─ Manual Snapshots: Before maintenance                        │
│  └─ Note: Redis is cache, data can be regenerated               │
│                                                                  │
│  KAFKA BACKUPS (MSK)                                             │
│  ├─ Retention: 7 days (configurable per topic)                  │
│  ├─ Cross-Region Replication: MirrorMaker 2.0                   │
│  └─ Note: Events are ephemeral, focus on retention              │
│                                                                  │
│  S3 BACKUPS                                                      │
│  ├─ Versioning: Enabled on all buckets                          │
│  ├─ Cross-Region Replication: Enabled                           │
│  ├─ Lifecycle Policies: Transition to Glacier                   │
│  └─ MFA Delete: Enabled for critical buckets                    │
│                                                                  │
│  KUBERNETES BACKUPS                                              │
│  ├─ Velero: Cluster state backup                                │
│  ├─ Git Repository: Kubernetes manifests (source of truth)      │
│  └─ etcd Backups: Automated via EKS                             │
│                                                                  │
│  APPLICATION BACKUPS                                             │
│  ├─ Configuration: Stored in Git                                │
│  ├─ Secrets: AWS Secrets Manager (automated backup)             │
│  └─ State: Database (primary source)                            │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 13.2 Aurora Backup Configuration

```hcl
# Aurora Cluster with Backup Configuration
resource "aws_rds_cluster" "aurora" {
  cluster_identifier     = "revenue-mgmt-aurora"
  engine                 = "aurora-postgresql"
  engine_version         = "15.4"
  database_name          = "revenue_mgmt"
  master_username        = "revenue_admin"
  master_password        = random_password.database.result
  backup_retention_period = 35
  preferred_backup_window = "03:00-04:00"
  preferred_maintenance_window = "sun:04:00-sun:05:00"
  deletion_protection    = true
  skip_final_snapshot    = false
  final_snapshot_identifier = "revenue-mgmt-aurora-final"

  vpc_security_group_ids = [aws_security_group.aurora.id]
  db_subnet_group_name   = aws_db_subnet_group.aurora.name

  enabled_cloudwatch_logs_exports = [
    "postgresql",
    "upgrade"
  ]

  tags = {
    Name        = "revenue-mgmt-aurora"
    Environment = "production"
  }
}

# Aurora Cluster Instance
resource "aws_rds_cluster_instance" "aurora" {
  count              = 3
  identifier         = "revenue-mgmt-aurora-${count.index}"
  cluster_identifier = aws_rds_cluster.aurora.id
  instance_class     = "db.r5.xlarge"
  engine             = aws_rds_cluster.aurora.engine
  engine_version     = aws_rds_cluster.aurora.engine_version

  tags = {
    Name = "revenue-mgmt-aurora-instance-${count.index}"
  }
}

# Automated Backup to S3 (Optional)
resource "aws_rds_cluster" "aurora_s3_export" {
  # Configure S3 export for long-term retention
  s3_import {
    source_engine         = "postgresql"
    source_engine_version = "15.4"
    bucket_name           = aws_s3_bucket.backups.id
    bucket_prefix         = "aurora-import/"
    ingestion_role        = aws_iam_role.aurora_s3_import.arn
  }
}

# Cross-Region Snapshot Copy
resource "aws_db_cluster_snapshot" "cross_region" {
  db_cluster_identifier          = aws_rds_cluster.aurora.id
  db_cluster_snapshot_identifier = "revenue-mgmt-aurora-snapshot-${formatdate("YYYY-MM-DD-HHMM", timestamp())}"

  # Copy to Tokyo region
  lifecycle {
    create_before_destroy = true
  }
}
```

### 13.3 Velero (Kubernetes Backup)

```yaml
# Velero Installation
apiVersion: apps/v1
kind: Deployment
metadata:
  name: velero
  namespace: velero
spec:
  replicas: 1
  selector:
    matchLabels:
      app: velero
  template:
    metadata:
      labels:
        app: velero
    spec:
      serviceAccountName: velero
      containers:
        - name: velero
          image: velero/velero:v1.12.0
          command:
            - /velero
          args:
            - server
            - --default-volumes-to-fs-backup
            - --metrics-address=0.0.0.0:8085
          ports:
            - containerPort: 8085
              name: metrics
          resources:
            requests:
              memory: "512Mi"
              cpu: "500m"
            limits:
              memory: "1Gi"
              cpu: "1000m"
          volumeMounts:
            - name: cloud-credentials
              mountPath: /credentials
          env:
            - name: AWS_SHARED_CREDENTIALS_FILE
              value: /credentials/cloud
            - name: VELERO_SCRATCH_DIR
              value: /scratch
      volumes:
        - name: cloud-credentials
          secret:
            secretName: cloud-credentials

---
# Velero Schedule: Daily backup
apiVersion: velero.io/v1
kind: Schedule
metadata:
  name: daily-backup
  namespace: velero
spec:
  schedule: "0 2 * * *"  # Daily at 2 AM
  template:
    ttl: 720h0m0s  # 30 days retention
    includedNamespaces:
      - revenue-mgmt
      - monitoring
    storageLocation: default
    volumeSnapshotLocations:
      - default
    defaultVolumesToFsBackup: true

---
# Velero Schedule: Weekly full backup
apiVersion: velero.io/v1
kind: Schedule
metadata:
  name: weekly-full-backup
  namespace: velero
spec:
  schedule: "0 3 * * 0"  # Weekly on Sunday at 3 AM
  template:
    ttl: 2160h0m0s  # 90 days retention
    includedNamespaces:
      - '*'
    storageLocation: default
    volumeSnapshotLocations:
      - default
    defaultVolumesToFsBackup: true

---
# Backup Storage Location
apiVersion: velero.io/v1
kind: BackupStorageLocation
metadata:
  name: default
  namespace: velero
spec:
  provider: aws
  default: true
  objectStorage:
    bucket: revenue-mgmt-backups-prod
    prefix: velero
  config:
    region: ap-southeast-1
    s3ForcePathStyle: "false"
```

### 13.4 Backup Verification & Testing

```yaml
# Backup Test Schedule
Backup Testing:
  Daily:
    - Verify backup completion status
    - Check backup size anomalies
    - Monitor backup duration
  
  Weekly:
    - Restore test to staging environment
    - Verify data integrity
    - Test application functionality
  
  Monthly:
    - Full disaster recovery drill
    - Cross-region restore test
    - RTO/RPO validation
  
  Quarterly:
    - Complete DR exercise
    - Stakeholder review
    - Update runbooks

# Backup Verification Script
#!/bin/bash
# verify-backup.sh

echo "Verifying backups..."

# Check Aurora backups
aws rds describe-db-cluster-snapshots \
  --db-cluster-identifier revenue-mgmt-aurora \
  --query 'DBClusterSnapshots[?SnapshotType==`automated`].{ID:DBClusterSnapshotIdentifier,Status:Status,Created:SnapshotCreateTime}' \
  --output table

# Check S3 bucket versioning
aws s3api get-bucket-versioning \
  --bucket revenue-mgmt-documents-prod

# Check Velero backups
kubectl get backups -n velero -o wide

# Check backup age
echo "Latest backup age:"
aws rds describe-db-cluster-snapshots \
  --db-cluster-identifier revenue-mgmt-aurora \
  --query 'max_by(DBClusterSnapshots, &SnapshotCreateTime).SnapshotCreateTime' \
  --output text
```

---

## 14. Cost Estimation

### 14.1 Monthly Cost Breakdown

```
┌─────────────────────────────────────────────────────────────────┐
│              MONTHLY COST ESTIMATION (Production)                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  COMPUTE (EKS)                                                   │
│  ├─ General Node Group (m5.xlarge x 6): $1,166.40              │
│  ├─ Compute Node Group (c5.2xlarge x 3): $1,051.20             │
│  ├─ Memory Node Group (r5.xlarge x 2): $511.20                 │
│  ├─ GPU Node Group (g4dn.xlarge x 0): $0 (on-demand)           │
│  └─ Subtotal: $2,728.80                                         │
│                                                                  │
│  DATABASE (Aurora PostgreSQL)                                    │
│  ├─ 3 x db.r5.xlarge instances: $1,752.00                      │
│  ├─ Storage (100 GB): $23.00                                   │
│  ├─ I/O requests: $50.00                                       │
│  └─ Subtotal: $1,825.00                                         │
│                                                                  │
│  CACHE (ElastiCache Redis)                                       │
│  ├─ 3 x cache.r5.large (Cluster mode): $1,095.00               │
│  └─ Subtotal: $1,095.00                                         │
│                                                                  │
│  MESSAGE QUEUE (MSK Kafka)                                       │
│  ├─ 3 x kafka.m5.large brokers: $876.00                        │
│  ├─ Storage (1 TB): $115.00                                    │
│  └─ Subtotal: $991.00                                           │
│                                                                  │
│  STORAGE (S3)                                                    │
│  ├─ Documents (500 GB): $11.50                                 │
│  ├─ Backups (2 TB): $46.00                                     │
│  ├─ Logs (100 GB): $2.30                                       │
│  ├─ Frontend (10 GB): $0.23                                    │
│  └─ Subtotal: $60.03                                            │
│                                                                  │
│  NETWORKING                                                      │
│  ├─ NAT Gateway (3 x $32.40): $97.20                           │
│  ├─ Data Transfer (10 TB): $900.00                             │
│  ├─ CloudFront (1 TB): $85.00                                  │
│  ├─ ALB (2 x $16.20): $32.40                                   │
│  └─ Subtotal: $1,114.60                                         │
│                                                                  │
│  SECURITY                                                        │
│  ├─ WAF: $5.00 + $1 per million requests: $50.00               │
│  ├─ KMS Keys (10 keys): $1.00                                  │
│  ├─ Secrets Manager (100 secrets): $4.00                       │
│  ├─ GuardDuty: $50.00                                          │
│  └─ Subtotal: $110.00                                           │
│                                                                  │
│  MONITORING                                                      │
│  ├─ CloudWatch Logs (100 GB): $25.00                           │
│  ├─ CloudWatch Metrics: $10.00                                 │
│  ├─ X-Ray Traces: $20.00                                       │
│  └─ Subtotal: $55.00                                            │
│                                                                  │
│  CI/CD & DEVELOPMENT                                             │
│  ├─ CodeBuild (5000 min): $7.50                                │
│  ├─ CodePipeline: $1.00                                        │
│  ├─ ECR Storage (100 GB): $10.00                               │
│  └─ Subtotal: $18.50                                            │
│                                                                  │
│  OTHER SERVICES                                                  │
│  ├─ Route 53 (Hosted Zone): $0.50                              │
│  ├─ ACM Certificates: $0.00 (free)                             │
│  ├─ CloudTrail: $2.00                                          │
│  ├─ Config: $5.00                                              │
│  └─ Subtotal: $7.50                                             │
│                                                                  │
│  ─────────────────────────────────────────────────────────────  │
│  TOTAL MONTHLY COST: $8,005.43                                  │
│  ─────────────────────────────────────────────────────────────  │
│                                                                  │
│  ANNUAL COST: $96,065.16                                        │
│  COST PER USER (1000 users): $8.01/month                        │
│  COST PER TRANSACTION (1M txns): $0.008                         │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 14.2 Cost Optimization Strategies

```yaml
# Cost Optimization Recommendations

Reserved Instances (40-60% savings):
  - EKS Nodes: 1-year reserved for baseline capacity
  - Aurora: Reserved instances for database
  - ElastiCache: Reserved for cache nodes
  
  Estimated Savings: $3,500/month (44%)

Spot Instances (60-70% savings):
  - Use for non-critical workloads
  - CI/CD runners
  - Batch processing jobs
  - ML training
  
  Estimated Savings: $500/month (6%)

Right-Sizing:
  - Monitor actual usage vs allocated
  - Downsize over-provisioned resources
  - Use AWS Compute Optimizer recommendations
  
  Estimated Savings: $800/month (10%)

Auto-Scaling:
  - Scale down during off-peak hours
  - Use Karpenter for efficient node provisioning
  - Implement HPA for all services
  
  Estimated Savings: $600/month (7.5%)

Storage Optimization:
  - Lifecycle policies for S3
  - Delete unused snapshots
  - Compress logs before storage
  
  Estimated Savings: $100/month (1.25%)

Total Potential Savings: $5,500/month (69%)
Optimized Monthly Cost: $2,505.43
```

### 14.3 Cost Allocation Tags

```hcl
# Tagging Strategy for Cost Allocation
resource "aws_cost_allocation_tag" "project" {
  tag_key      = "Project"
  tag_values   = ["revenue-management"]
  status       = "Active"
}

resource "aws_cost_allocation_tag" "environment" {
  tag_key      = "Environment"
  tag_values   = ["production", "staging", "development"]
  status       = "Active"
}

resource "aws_cost_allocation_tag" "team" {
  tag_key      = "Team"
  tag_values   = ["engineering", "data", "devops"]
  status       = "Active"
}

resource "aws_cost_allocation_tag" "cost_center" {
  tag_key      = "CostCenter"
  tag_values   = ["CC-1001"]
  status       = "Active"
}

# Apply tags to all resources
locals {
  common_tags = {
    Project     = "revenue-management"
    Environment = var.environment
    Team        = "engineering"
    CostCenter  = "CC-1001"
    ManagedBy   = "terraform"
    CreatedAt   = timestamp()
  }
}
```

### 14.4 Budget Alerts

```hcl
# AWS Budget
resource "aws_budgets_budget" "monthly" {
  name         = "revenue-mgmt-monthly-budget"
  budget_type  = "COST"
  limit_amount = "10000"
  limit_unit   = "USD"
  time_unit    = "MONTHLY"

  cost_filter {
    name   = "TagKeyValue"
    values = ["user:Project$revenue-management"]
  }

  notification {
    comparison_operator        = "GREATER_THAN"
    threshold                  = 80
    threshold_type             = "PERCENTAGE"
    notification_type          = "FORECASTED"
    subscriber_email_addresses = ["finance@company.com", "devops@company.com"]
  }

  notification {
    comparison_operator        = "GREATER_THAN"
    threshold                  = 100
    threshold_type             = "PERCENTAGE"
    notification_type          = "ACTUAL"
    subscriber_email_addresses = ["finance@company.com", "devops@company.com", "cto@company.com"]
  }
}
```

---

## ภาคผนวก

### ภาคผนวก A: Infrastructure Checklist

```yaml
Pre-Deployment Checklist:
  Network:
    - [ ] VPC created with proper CIDR
    - [ ] Subnets created across 3 AZs
    - [ ] Route tables configured
    - [ ] NAT Gateways deployed
    - [ ] Security groups configured
    - [ ] Network ACLs configured
    - [ ] VPC endpoints created
  
  Compute:
    - [ ] EKS cluster created
    - [ ] Node groups configured
    - [ ] IAM roles created
    - [ ] Service accounts configured
  
  Database:
    - [ ] Aurora cluster created
    - [ ] Subnet group configured
    - [ ] Parameter group configured
    - [ ] Backup enabled
    - [ ] Monitoring enabled
  
  Cache:
    - [ ] ElastiCache cluster created
    - [ ] Subnet group configured
    - [ ] Parameter group configured
  
  Message Queue:
    - [ ] MSK cluster created
    - [ ] Topics created
    - [ ] IAM roles configured
  
  Storage:
    - [ ] S3 buckets created
    - [ ] Versioning enabled
    - [ ] Encryption enabled
    - [ ] Lifecycle policies configured
  
  Security:
    - [ ] KMS keys created
    - [ ] Secrets Manager configured
    - [ ] WAF configured
    - [ ] SSL certificates created
  
  Monitoring:
    - [ ] Prometheus deployed
    - [ ] Grafana deployed
    - [ ] Alerting configured
    - [ ] Dashboards created
  
  CI/CD:
    - [ ] GitHub Actions configured
    - [ ] ArgoCD deployed
    - [ ] ECR repositories created
  
  Backup:
    - [ ] Aurora backups enabled
    - [ ] Velero deployed
    - [ ] Backup schedules configured
    - [ ] DR plan documented
```

### ภาคผนวก B: Disaster Recovery Runbook

```markdown
# Disaster Recovery Runbook

## Scenario 1: Database Failure

### Detection
- CloudWatch alarm: DatabaseDown
- Application health check failures
- Increased error rates

### Response
1. Assess the situation
2. If automated failover didn't trigger:
   - Manually promote read replica
   - Update DNS/endpoint
3. Verify application connectivity
4. Monitor for data consistency
5. Create new read replica
6. Document incident

### RTO: 15 minutes (automated), 30 minutes (manual)
### RPO: 0 (synchronous replication)

## Scenario 2: EKS Cluster Failure

### Detection
- kubectl commands failing
- Application pods not running
- Monitoring alerts

### Response
1. Check AWS EKS console
2. If cluster is unreachable:
   - Create new EKS cluster in different AZ
   - Deploy applications from Git (ArgoCD)
   - Restore database from backup if needed
3. Update DNS to new cluster
4. Verify all services
5. Document incident

### RTO: 2 hours
### RPO: 0 (GitOps)

## Scenario 3: Region Failure

### Detection
- AWS Service Health Dashboard
- Multiple service failures
- Network connectivity issues

### Response
1. Declare disaster
2. Activate DR region (Tokyo)
3. Restore database from cross-region snapshot
4. Deploy applications via ArgoCD
5. Update DNS (Route 53 failover)
6. Verify all services
7. Communicate with stakeholders
8. Document incident

### RTO: 4 hours
### RPO: 1 hour
```

### ภาคผนวก C: Terraform Module Structure

```
terraform/
├── modules/
│   ├── vpc/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   └── versions.tf
│   ├── eks/
│   ├── rds/
│   ├── elasticache/
│   ├── msk/
│   ├── s3/
│   ├── ecr/
│   ├── iam/
│   ├── kms/
│   ├── waf/
│   ├── cloudfront/
│   ├── route53/
│   └── monitoring/
├── environments/
│   ├── production/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── terraform.tfvars
│   │   └── backend.tf
│   ├── staging/
│   └── development/
├── scripts/
│   ├── init.sh
│   ├── plan.sh
│   ├── apply.sh
│   └── destroy.sh
── .terraform.lock.hcl
├── terraform.tfvars.example
└── README.md
```

---

**เอกสารนี้จัดทำขึ้นเพื่อใช้เป็นแนวทางในการออกแบบ Cloud Infrastructure สำหรับระบบ Revenue Management**

**เวอร์ชัน:** 1.0  
**วันที่:** 22 มิถุนายน 2026  
**สถานะ:** Draft for Review  
**ผู้จัดทำ:** Infrastructure Team
