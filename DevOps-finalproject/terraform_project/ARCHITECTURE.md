# LY StatusPage - Architecture Document

## Overview
Complete DevOps project for deploying a Django Status Page application on AWS with secure and modern infrastructure.

---

## Infrastructure Architecture

### Core Infrastructure Components

#### 1. Networking & Security
- **VPC**: `10.0.0.0/16` in us-east-1
- **Public Subnets**: 2 subnets (`10.0.1.0/24`, `10.0.2.0/24`)
  - Internet Gateway
  - NAT Gateway (single for cost optimization)
  - Jenkins Server
  - Future Load Balancers
- **Private Subnets**: 2 subnets (`10.0.3.0/24`, `10.0.4.0/24`)
  - EKS Worker Nodes
  - RDS PostgreSQL
  - ElastiCache Redis (future)

#### 2. Compute
- **EKS Cluster**: 
  - Version 1.28
  - AWS-managed Control Plane
  - Node Group: 1-2 instances of t3.small (Spot)
- **Jenkins Server**: 
  - EC2 t3.small in Public Subnet
  - Docker, kubectl, AWS CLI pre-installed
  - Access via port 8080

#### 3. Databases
- **RDS PostgreSQL**:
  - Instance: db.r6g.large
  - Storage: 20GB encrypted
  - Multi-AZ: No (cost optimization)
  - In Private Subnets with restricted Security Group
- **Redis** (phases):
  - Phase 1: Temporary Pod in Kubernetes
  - Phase 2: ElastiCache Cluster (when permissions available)

#### 4. Registry & Storage
- **ECR Repository**: `ly-statuspage-repo`
- **Route53**: ly-statuspage.com (DNS)

---

## Application Architecture

### Application Layer
```
Internet → Route53 → ALB (future) → EKS Ingress → Django Pods
                                                      ↓
                                               RDS + Redis
```

#### Django StatusPage:
- **Pods**: 3 replicas with HPA (3-10 pods)
- **Resources**: 256Mi-512Mi RAM, 0.25-0.5 CPU
- **Probes**: Health checks and Readiness checks
- **ConfigMaps**: Application configuration
- **Secrets**: DB passwords and API keys

### CI/CD Layer
```
GitHub → Jenkins → Docker Build → ECR Push → K8s Deploy
```

#### Pipeline Stages:
1. **Source**: GitHub webhook
2. **Build**: Docker image building
3. **Test**: Unit tests (future)
4. **Security**: Image scanning in ECR
5. **Deploy**: Rolling update in K8s

---

## Security

### Security Groups
- **EKS Cluster SG**: Control plane communication
- **EKS Nodes SG**: 
  - Port 30000-32767 (NodePort)
  - Port 80, 443 from future ALB
  - Internal ports from Control Plane
- **RDS SG**: Port 5432 only from EKS Nodes
- **Jenkins SG**: Port 8080, 22 (restricted to specific IP)

### Network Security
- **Private Subnets**: Application and DB isolated from internet
- **NAT Gateway**: Controlled outbound internet access
- **Security Groups**: Least Privilege Access

---

## Monitoring & Logging

### Current Phase:
- **CloudWatch**: EKS and EC2 logs
- **EKS Built-in**: Basic cluster metrics

### Future Phase:
- **Prometheus**: Metrics collection
- **Grafana**: Dashboards and visualization
- **AlertManager**: Alerting
- **ELK Stack**: Log aggregation and analytics

---

## Traffic Flow

### Current (NodePort):
```
User → Route53 → EKS Node IP:30080 → Django Pod
```

### Future (ALB):
```
User → Route53 → ALB → Target Group → EKS Service → Django Pod
```

---

## Deployment Plan

### Phase 1: Infrastructure ✓
- Terraform apply
- VPC, EKS, RDS, Jenkins
- ~15 minutes

### Phase 2: Application
- Docker build StatusPage
- ECR push
- Kubernetes manifests apply
- ~30 minutes

### Phase 3: CI/CD
- Jenkins configuration
- Pipeline setup
- GitHub integration
- ~45 minutes

### Phase 4: Monitoring
- Prometheus & Grafana
- Dashboards setup
- Alerts configuration
- ~30 minutes

### Phase 5: DNS & SSL
- Route53 A record
- SSL certificates
- Domain configuration
- ~20 minutes

---

## Cost Breakdown

### Estimated Monthly Costs:
| Component | Monthly Cost |
|-----------|-------------|
| EKS Control Plane | $72 |
| EKS Nodes (t3.small SPOT) | $6-8 |
| Jenkins (t3.small) | $15 |
| RDS (db.r6g.large) | $120 |
| NAT Gateway | $45 |
| Storage & Networking | $7 |
| **Total** | **~$265** |

### Cost until 21/9 (18 days):
**$265 × 0.6 = ~$159**

---

## Technical Aspects

### Software Versions:
- **Kubernetes**: 1.28
- **Python**: 3.10
- **Django**: Latest (from StatusPage repo)
- **PostgreSQL**: 15.4
- **Redis**: 7-alpine

### Performance Requirements:
- **Response Time**: < 200ms
- **Availability**: 99.9%
- **Concurrent Users**: 100-500
- **Database Connections**: 20-50

---

## Backup and Recovery Strategy

### Backups:
- **RDS**: Automated backups (7 days)
- **Application State**: Stateless (no backup required)
- **Configuration**: Git repository

### Disaster Recovery:
- **RTO**: 30 minutes (terraform apply)
- **RPO**: 24 hours (RDS backups)

---

## Future Enhancements

### Short Term:
1. **ElastiCache Redis** (replace temporary Pod)
2. **Application Load Balancer** + Target Groups
3. **SSL/TLS Certificates**
4. **Custom Domain**: status.ly-statuspage.com

### Medium Term:
1. **WAF** (Web Application Firewall)
2. **Multi-AZ RDS** for high availability
3. **EFS** for shared file storage
4. **CloudFront CDN**

### Long Term:
1. **EKS Fargate** for critical pods
2. **Multi-Region Setup**
3. **Advanced Security**: GuardDuty, Inspector
4. **Cost Optimization**: Reserved Instances

---

## Risks and Limitations

### Technical Risks:
- **Single NAT Gateway**: Single point of failure
- **Single AZ Database**: No full HA
- **Spot Instances**: May be interrupted

### Current Limitations:
- No ElastiCache permissions
- No Target Groups permissions
- Budget limited until 21/9

### Risk Mitigation Strategy:
- Regular stability testing
- Daily cost monitoring
- Detailed backup plans
