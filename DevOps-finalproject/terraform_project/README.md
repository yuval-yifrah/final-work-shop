# LY StatusPage - DevOps Project

## Project Description
Complete DevOps project for deploying Django Status Page application on AWS with secure infrastructure, Kubernetes, CI/CD Pipeline and advanced monitoring.

---

## Architecture

### AWS Infrastructure
- **VPC** with Public/Private Subnets
- **EKS Cluster** with Spot Instances for cost optimization
- **RDS PostgreSQL** (db.r6g.large)
- **ECR** for Docker image management
- **Route53** for DNS management
- **Jenkins Server** for CI/CD

### Application Stack
- **Django StatusPage** (3 replicas with HPA)
- **Redis** (temporary as Pod, future ElastiCache)
- **Prometheus & Grafana** for monitoring
- **NGINX Ingress** or ALB for routing

---

## Costs
- **Monthly cost**: ~$265
- **Cost until 21/9** (18 days): ~$159
- **Maximum budget**: $300

---

## Project Structure

```
terraform_project/
├── README.md
├── ARCHITECTURE.md
├── terraform/
│   ├── main.tf                 # Main infrastructure
│   ├── variables.tf            # Variables
│   ├── outputs.tf              # Outputs
│   └── terraform.tfvars        # Configuration
├── k8s/
│   ├── redis-deployment.yaml   # Temporary Redis
│   ├── django-deployment.yaml  # Django app
│   └── monitoring/
│       ├── prometheus.yaml     # (future)
│       └── grafana.yaml        # (future)
├── docker/
│   ├── Dockerfile              # Django image
│   └── requirements.txt        # Python dependencies
├── jenkins/
│   ├── Jenkinsfile             # Pipeline definition
│   └── scripts/
│       └── deploy.sh           # Deployment script
└── scripts/
    ├── setup.sh                # Initial setup
    ├── deploy.sh               # Deployment
    └── cleanup.sh              # Resource cleanup
```

---

## Installation and Deployment

### Prerequisites
- AWS CLI configured with permissions
- Terraform >= 1.5.0
- Docker
- kubectl
- Git

### Step 1: Infrastructure Setup

```bash
# Clone the repository
git clone <your-repo>
cd terraform_project

# Create Key Pair (if not exists)
aws ec2 create-key-pair --key-name yuvalyhome --query 'KeyMaterial' --output text > ~/.ssh/yuvalyhome.pem
chmod 400 ~/.ssh/yuvalyhome.pem

# Deploy infrastructure
terraform init
terraform validate
terraform plan
terraform apply
```

### Step 2: Application Build

```bash
# Get ECR details
ECR_URL=$(terraform output -raw ecr_repository_url)
RDS_ENDPOINT=$(terraform output -raw rds_endpoint)

# Login to ECR
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin $ECR_URL

# Build and Push Django image
cd ../statuspage-app  # Directory with Django code
docker build -t ly-statuspage .
docker tag ly-statuspage:latest $ECR_URL:latest
docker push $ECR_URL:latest
```

### Step 3: Kubernetes Deployment

```bash
# Connect to EKS
aws eks update-kubeconfig --region us-east-1 --name ly-statuspage-cluster

# Update ConfigMap with RDS endpoint
sed -i "s/REPLACE_WITH_RDS_ENDPOINT/$RDS_ENDPOINT/g" k8s/django-deployment.yaml
sed -i "s|REPLACE_WITH_ECR_URL|$ECR_URL|g" k8s/django-deployment.yaml

# Deploy temporary Redis
kubectl apply -f k8s/redis-deployment.yaml

# Deploy Django application
kubectl apply -f k8s/django-deployment.yaml

# Check status
kubectl get pods
kubectl get services
```

### Step 4: Jenkins Setup

```bash
# Get Jenkins IP
JENKINS_IP=$(terraform output -raw jenkins_public_ip)
echo "Jenkins URL: http://$JENKINS_IP:8080"

# Get initial password
ssh -i ~/.ssh/yuvalyhome.pem ec2-user@$JENKINS_IP "sudo cat /var/lib/jenkins/secrets/initialAdminPassword"

# Connect to Jenkins in Browser
# Setup Pipeline with Jenkinsfile
```

---

## Additional Configurations

### Jenkins Pipeline Configuration

Create New Pipeline Job in Jenkins with settings:
- **Pipeline script from SCM**
- **Git Repository**: your repository
- **Script Path**: `jenkins/Jenkinsfile`

### Environment Variables in Jenkins:
```
ECR_URL = <terraform output ecr_repository_url>
CLUSTER_NAME = ly-statuspage-cluster
AWS_REGION = us-east-1
```

### DNS Configuration (when ALB is available):

```bash
# Get ALB DNS name (future)
ALB_DNS=$(kubectl get service django-service -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

# Create Route53 A record
aws route53 change-resource-record-sets \
  --hosted-zone-id $(terraform output -raw route53_zone_id) \
  --change-batch '{
    "Changes": [{
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "status.ly-statuspage.com",
        "Type": "CNAME",
        "TTL": 300,
        "ResourceRecords": [{"Value": "'$ALB_DNS'"}]
      }
    }]
  }'
```

---

## Monitoring (Future)

### Install Prometheus & Grafana:

```bash
# Add Helm repository
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Install monitoring stack
helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --set grafana.service.type=NodePort \
  --set grafana.service.nodePort=30003

# Access Grafana
NODE_IP=$(kubectl get nodes -o wide --no-headers | awk '{print $7}' | head -1)
echo "Grafana URL: http://$NODE_IP:30003"
echo "Username: admin"
echo "Password: $(kubectl get secret monitoring-grafana -n monitoring -o jsonpath="{.data.admin-password}" | base64 --decode)"
```

---

## Testing

### Application Health Check:

```bash
# Check Pods
kubectl get pods -l app=django-app

# Check Logs
kubectl logs -l app=django-app

# Test application access
NODE_IP=$(kubectl get nodes -o wide --no-headers | awk '{print $7}' | head -1)
curl http://$NODE_IP:30080

# Test Database connection
kubectl exec -it deployment/django-app -- python manage.py dbshell
```

### Load Testing (Optional):

```bash
# Install Apache Bench
sudo yum install httpd-tools

# Performance testing
ab -n 1000 -c 10 http://$NODE_IP:30080/
```

---

## Security

### Security Best Practices:
- All services in Private Subnets
- Security Groups with Least Privilege
- Encrypted RDS
- ECR image scanning
- Secrets in Kubernetes secrets (not in code)

### Restrict Jenkins Access:
```bash
# Update Security Group to specific IP
MY_IP=$(curl -s ifconfig.me)
aws ec2 authorize-security-group-ingress \
  --group-id $(terraform output -raw jenkins_security_group_id) \
  --protocol tcp \
  --port 8080 \
  --cidr $MY_IP/32
```

---

## Updates and Maintenance

### Application Update:

```bash
# Build new version
docker build -t ly-statuspage:v2.0 .
docker tag ly-statuspage:v2.0 $ECR_URL:v2.0
docker push $ECR_URL:v2.0

# Rolling update
kubectl set image deployment/django-app django=$ECR_URL:v2.0
kubectl rollout status deployment/django-app

# Rollback if needed
kubectl rollout undo deployment/django-app
```

### Backup and Recovery:

```bash
# Backup Kubernetes configurations
kubectl get all -o yaml > backup-$(date +%Y%m%d).yaml

# Restore RDS from Snapshot (if needed)
aws rds restore-db-instance-from-db-snapshot \
  --db-instance-identifier ly-statuspage-restored \
  --db-snapshot-identifier <snapshot-id>
```

---

## Future Steps

### Short Term (1-2 weeks):
- [ ] ElastiCache Redis instead of temporary Pod
- [ ] Application Load Balancer + Target Groups
- [ ] SSL Certificates with ACM
- [ ] Custom Domain setup

### Medium Term (1 month):
- [ ] ELK Stack for log aggregation
- [ ] Advanced monitoring dashboards
- [ ] WAF for Web Application security
- [ ] Multi-AZ RDS for high availability

### Long Term (3 months):
- [ ] Multi-region deployment
- [ ] Blue-Green deployments
- [ ] Cost optimization with Reserved Instances
- [ ] Advanced security with GuardDuty

---

## Resource Cleanup

```bash
# Delete Kubernetes resources
kubectl delete -f k8s/

# Delete AWS infrastructure
terraform destroy

# Delete ECR images
aws ecr batch-delete-image \
  --repository-name ly-statuspage-repo \
  --image-ids imageTag=latest
```

---

## Support and Common Issues

### Common Issues:

**Pod not starting:**
```bash
kubectl describe pod <pod-name>
kubectl logs <pod-name>
```

**Database connection issues:**
```bash
# Check Security Groups
aws ec2 describe-security-groups --group-ids <rds-sg-id>

# Test connection from Pod
kubectl exec -it deployment/django-app -- nc -zv <rds-endpoint> 5432
```

**Jenkins not accessible:**
```bash
# Check Security Group
aws ec2 describe-security-groups --group-ids <jenkins-sg-id>

# Check EC2 instance
aws ec2 describe-instances --instance-ids <jenkins-instance-id>
```

---

## Tags

`#DevOps` `#AWS` `#Kubernetes` `#Django` `#Terraform` `#Jenkins` `#Docker` `#StatusPage`

---

**Created by:** Lir Chen + Yuval Ifrah  
**Date:** September 2025  
**Version:** 1.0
