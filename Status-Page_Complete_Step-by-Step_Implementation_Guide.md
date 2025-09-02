# Status-Page Complete Step-by-Step Implementation Guide
## Repository: https://github.com/Status-Page/Status-Page/

## Prerequisites

### AWS Account Setup
- [ ] Create AWS account
- [ ] Set up billing alerts at $250 and $300
- [ ] Enable MFA on root account
- [ ] Create IAM user for daily operations
- [ ] Install AWS CLI v2
- [ ] Configure AWS CLI with access keys

### Domain and Email Setup
- [ ] Purchase domain name (recommend Namecheap, ~$12/year)
- [ ] Create Route 53 hosted zone
- [ ] Update nameservers at domain registrar
- [ ] Verify domain in SES for email notifications
- [ ] Choose primary AWS region: us-east-1 (recommended for cost)

---

## Phase 1: Enhanced Network Infrastructure (Day 1, 2-3 hours)

### Step 1.1: Create VPC and Subnets (Same as before)
```bash
# Set variables for Status-Page
export AWS_REGION=us-east-1
export VPC_NAME=StatusPage-VPC
export PROJECT_NAME=StatusPage

# Create VPC
VPC_ID=$(aws ec2 create-vpc \
    --cidr-block 10.0.0.0/16 \
    --tag-specifications "ResourceType=vpc,Tags=[{Key=Name,Value=$VPC_NAME},{Key=Project,Value=$PROJECT_NAME}]" \
    --query 'Vpc.VpcId' --output text)
echo "VPC ID: $VPC_ID"

# Enable DNS hostnames (required for Status-Page)
aws ec2 modify-vpc-attribute --vpc-id $VPC_ID --enable-dns-hostnames
aws ec2 modify-vpc-attribute --vpc-id $VPC_ID --enable-dns-support

# Create Internet Gateway
IGW_ID=$(aws ec2 create-internet-gateway \
    --tag-specifications "ResourceType=internet-gateway,Tags=[{Key=Name,Value=$VPC_NAME-IGW},{Key=Project,Value=$PROJECT_NAME}]" \
    --query 'InternetGateway.InternetGatewayId' --output text)
echo "Internet Gateway ID: $IGW_ID"

# Attach IGW to VPC
aws ec2 attach-internet-gateway --vpc-id $VPC_ID --internet-gateway-id $IGW_ID

# Create Subnets in different AZs (required for ALB and RDS Multi-AZ)
PUBLIC_SUBNET_1_ID=$(aws ec2 create-subnet \
    --vpc-id $VPC_ID \
    --cidr-block 10.0.1.0/24 \
    --availability-zone us-east-1a \
    --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=$VPC_NAME-Public-1a},{Key=Project,Value=$PROJECT_NAME}]" \
    --query 'Subnet.SubnetId' --output text)

PUBLIC_SUBNET_2_ID=$(aws ec2 create-subnet \
    --vpc-id $VPC_ID \
    --cidr-block 10.0.4.0/24 \
    --availability-zone us-east-1b \
    --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=$VPC_NAME-Public-1b},{Key=Project,Value=$PROJECT_NAME}]" \
    --query 'Subnet.SubnetId' --output text)

PRIVATE_SUBNET_1_ID=$(aws ec2 create-subnet \
    --vpc-id $VPC_ID \
    --cidr-block 10.0.2.0/24 \
    --availability-zone us-east-1a \
    --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=$VPC_NAME-Private-1a},{Key=Project,Value=$PROJECT_NAME}]" \
    --query 'Subnet.SubnetId' --output text)

PRIVATE_SUBNET_2_ID=$(aws ec2 create-subnet \
    --vpc-id $VPC_ID \
    --cidr-block 10.0.3.0/24 \
    --availability-zone us-east-1b \
    --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=$VPC_NAME-Private-1b},{Key=Project,Value=$PROJECT_NAME}]" \
    --query 'Subnet.SubnetId' --output text)

echo "Public Subnet 1: $PUBLIC_SUBNET_1_ID"
echo "Public Subnet 2: $PUBLIC_SUBNET_2_ID" 
echo "Private Subnet 1: $PRIVATE_SUBNET_1_ID"
echo "Private Subnet 2: $PRIVATE_SUBNET_2_ID"
```

### Step 1.2: Create NAT Gateway
```bash
# Allocate Elastic IP for NAT Gateway
EIP_ALLOC_ID=$(aws ec2 allocate-address \
    --domain vpc \
    --tag-specifications "ResourceType=elastic-ip,Tags=[{Key=Name,Value=$VPC_NAME-NAT-EIP},{Key=Project,Value=$PROJECT_NAME}]" \
    --query 'AllocationId' --output text)
echo "EIP Allocation ID: $EIP_ALLOC_ID"

# Create NAT Gateway in first public subnet
NAT_GW_ID=$(aws ec2 create-nat-gateway \
    --subnet-id $PUBLIC_SUBNET_1_ID \
    --allocation-id $EIP_ALLOC_ID \
    --tag-specifications "ResourceType=nat-gateway,Tags=[{Key=Name,Value=$VPC_NAME-NAT},{Key=Project,Value=$PROJECT_NAME}]" \
    --query 'NatGateway.NatGatewayId' --output text)
echo "NAT Gateway ID: $NAT_GW_ID"

# Wait for NAT Gateway to be available
aws ec2 wait nat-gateway-available --nat-gateway-ids $NAT_GW_ID
```

### Step 1.3: Create Route Tables
```bash
# Create Public Route Table
PUBLIC_RT_ID=$(aws ec2 create-route-table \
    --vpc-id $VPC_ID \
    --tag-specifications "ResourceType=route-table,Tags=[{Key=Name,Value=$VPC_NAME-Public-RT},{Key=Project,Value=$PROJECT_NAME}]" \
    --query 'RouteTable.RouteTableId' --output text)

# Create Private Route Table
PRIVATE_RT_ID=$(aws ec2 create-route-table \
    --vpc-id $VPC_ID \
    --tag-specifications "ResourceType=route-table,Tags=[{Key=Name,Value=$VPC_NAME-Private-RT},{Key=Project,Value=$PROJECT_NAME}]" \
    --query 'RouteTable.RouteTableId' --output text)

# Add routes
aws ec2 create-route --route-table-id $PUBLIC_RT_ID --destination-cidr-block 0.0.0.0/0 --gateway-id $IGW_ID
aws ec2 create-route --route-table-id $PRIVATE_RT_ID --destination-cidr-block 0.0.0.0/0 --nat-gateway-id $NAT_GW_ID

# Associate subnets with route tables
aws ec2 associate-route-table --subnet-id $PUBLIC_SUBNET_1_ID --route-table-id $PUBLIC_RT_ID
aws ec2 associate-route-table --subnet-id $PUBLIC_SUBNET_2_ID --route-table-id $PUBLIC_RT_ID
aws ec2 associate-route-table --subnet-id $PRIVATE_SUBNET_1_ID --route-table-id $PRIVATE_RT_ID
aws ec2 associate-route-table --subnet-id $PRIVATE_SUBNET_2_ID --route-table-id $PRIVATE_RT_ID
```

### Step 1.4: Create Enhanced Security Groups for Status-Page
```bash
# ALB Security Group (same as before)
ALB_SG_ID=$(aws ec2 create-security-group \
    --group-name StatusPage-ALB-SG \
    --description "Status-Page Load Balancer Security Group" \
    --vpc-id $VPC_ID \
    --tag-specifications "ResourceType=security-group,Tags=[{Key=Name,Value=StatusPage-ALB-SG},{Key=Project,Value=$PROJECT_NAME}]" \
    --query 'GroupId' --output text)

# EC2 Security Group - Enhanced for Status-Page
EC2_SG_ID=$(aws ec2 create-security-group \
    --group-name StatusPage-EC2-SG \
    --description "Status-Page Application Server Security Group" \
    --vpc-id $VPC_ID \
    --tag-specifications "ResourceType=security-group,Tags=[{Key=Name,Value=StatusPage-EC2-SG},{Key=Project,Value=$PROJECT_NAME}]" \
    --query 'GroupId' --output text)

# RDS Security Group
RDS_SG_ID=$(aws ec2 create-security-group \
    --group-name StatusPage-RDS-SG \
    --description "Status-Page Database Security Group" \
    --vpc-id $VPC_ID \
    --tag-specifications "ResourceType=security-group,Tags=[{Key=Name,Value=StatusPage-RDS-SG},{Key=Project,Value=$PROJECT_NAME}]" \
    --query 'GroupId' --output text)

# Redis Security Group
REDIS_SG_ID=$(aws ec2 create-security-group \
    --group-name StatusPage-Redis-SG \
    --description "Status-Page Redis Security Group" \
    --vpc-id $VPC_ID \
    --tag-specifications "ResourceType=security-group,Tags=[{Key=Name,Value=StatusPage-Redis-SG},{Key=Project,Value=$PROJECT_NAME}]" \
    --query 'GroupId' --output text)

# Configure Security Group Rules
aws ec2 authorize-security-group-ingress --group-id $ALB_SG_ID --protocol tcp --port 80 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id $ALB_SG_ID --protocol tcp --port 443 --cidr 0.0.0.0/0

# Status-Page runs on port 8000 by default
aws ec2 authorize-security-group-ingress --group-id $EC2_SG_ID --protocol tcp --port 8000 --source-group $ALB_SG_ID
# Allow HTTPS for external API calls and package downloads
aws ec2 authorize-security-group-ingress --group-id $EC2_SG_ID --protocol tcp --port 443 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id $EC2_SG_ID --protocol tcp --port 80 --cidr 0.0.0.0/0

# Database and cache access
aws ec2 authorize-security-group-ingress --group-id $RDS_SG_ID --protocol tcp --port 5432 --source-group $EC2_SG_ID
aws ec2 authorize-security-group-ingress --group-id $REDIS_SG_ID --protocol tcp --port 6379 --source-group $EC2_SG_ID

echo "Security Groups Created:"
echo "ALB SG: $ALB_SG_ID"
echo "EC2 SG: $EC2_SG_ID"
echo "RDS SG: $RDS_SG_ID"
echo "Redis SG: $REDIS_SG_ID"
```

---

## Phase 2: Enhanced Database and Cache Setup (Day 2, 1.5 hours)

### Step 2.1: Create RDS Subnet Group
```bash
aws rds create-db-subnet-group \
    --db-subnet-group-name statuspage-subnet-group \
    --db-subnet-group-description "Status-Page RDS Subnet Group" \
    --subnet-ids $PRIVATE_SUBNET_1_ID $PRIVATE_SUBNET_2_ID \
    --tags Key=Name,Value=StatusPage-DB-SubnetGroup Key=Project,Value=$PROJECT_NAME
```

### Step 2.2: Create Enhanced RDS PostgreSQL Instance for Status-Page
```bash
# Generate secure password
DB_PASSWORD=$(openssl rand -base64 32 | tr -d "=+/" | cut -c1-25)
echo "Database Password: $DB_PASSWORD" # Save this securely!

# Create enhanced RDS instance for Status-Page workload
aws rds create-db-instance \
    --db-instance-identifier statuspage-db \
    --db-instance-class db.t3.small \
    --engine postgres \
    --engine-version 14.9 \
    --master-username statuspage_admin \
    --master-user-password "$DB_PASSWORD" \
    --allocated-storage 30 \
    --max-allocated-storage 100 \
    --storage-type gp2 \
    --vpc-security-group-ids $RDS_SG_ID \
    --db-subnet-group-name statuspage-subnet-group \
    --db-name statuspage \
    --backup-retention-period 14 \
    --multi-az \
    --storage-encrypted \
    --auto-minor-version-upgrade \
    --tags Key=Name,Value=StatusPage-DB Key=Project,Value=$PROJECT_NAME \
    --enable-performance-insights \
    --performance-insights-retention-period 7
```

### Step 2.3: Create Enhanced ElastiCache for Status-Page
```bash
# Create ElastiCache subnet group
aws elasticache create-cache-subnet-group \
    --cache-subnet-group-name statuspage-redis-subnet \
    --cache-subnet-group-description "Status-Page Redis Subnet Group" \
    --subnet-ids $PRIVATE_SUBNET_1_ID $PRIVATE_SUBNET_2_ID

# Create enhanced Redis cluster for Status-Page
aws elasticache create-cache-cluster \
    --cache-cluster-id statuspage-redis \
    --cache-node-type cache.t3.small \
    --engine redis \
    --engine-version 7.0 \
    --num-cache-nodes 1 \
    --security-group-ids $REDIS_SG_ID \
    --cache-subnet-group-name statuspage-redis-subnet \
    --tags Key=Name,Value=StatusPage-Redis Key=Project,Value=$PROJECT_NAME \
    --snapshot-retention-limit 5 \
    --auto-minor-version-upgrade
```

### Step 2.4: Setup SES for Status-Page Notifications
```bash
# Verify domain for SES (required for Status-Page notifications)
aws ses verify-domain-identity --domain yourdomain.com

# Verify specific email addresses
aws ses verify-email-identity --email-address noreply@yourdomain.com
aws ses verify-email-identity --email-address admin@yourdomain.com

# Create SES configuration set for tracking
aws ses create-configuration-set --configuration-set Name=statuspage-notifications

# Set up SNS topic for bounce/complaint handling
SNS_TOPIC_ARN=$(aws sns create-topic --name statuspage-ses-bounces --query 'TopicArn' --output text)
aws ses put-identity-notification-attributes \
    --identity yourdomain.com \
    --notification-type Bounce \
    --sns-topic $SNS_TOPIC_ARN
```

### Step 2.5: Store Enhanced Configuration in Parameter Store
```bash
# Wait for RDS to be available
echo "Waiting for RDS instance to be available..."
aws rds wait db-instance-available --db-instance-identifier statuspage-db

# Get RDS endpoint
RDS_ENDPOINT=$(aws rds describe-db-instances \
    --db-instance-identifier statuspage-db \
    --query 'DBInstances[0].Endpoint.Address' --output text)

# Wait for Redis to be available
echo "Waiting for Redis cluster to be available..."
aws elasticache wait cache-cluster-available --cache-cluster-id statuspage-redis

# Get Redis endpoint
REDIS_ENDPOINT=$(aws elasticache describe-cache-clusters \
    --cache-cluster-id statuspage-redis \
    --show-cache-node-info \
    --query 'CacheClusters[0].CacheNodes[0].Endpoint.Address' --output text)

# Generate Django secret key
SECRET_KEY=$(python3 -c 'from django.core.management.utils import get_random_secret_key; print(get_random_secret_key())')

# Store all parameters
aws ssm put-parameter --name "/statuspage/prod/db_host" --value "$RDS_ENDPOINT" --type "String"
aws ssm put-parameter --name "/statuspage/prod/db_password" --value "$DB_PASSWORD" --type "SecureString"
aws ssm put-parameter --name "/statuspage/prod/redis_host" --value "$REDIS_ENDPOINT" --type "String"
aws ssm put-parameter --name "/statuspage/prod/secret_key" --value "$SECRET_KEY" --type "SecureString"
aws ssm put-parameter --name "/statuspage/prod/domain_name" --value "status.yourdomain.com" --type "String"
aws ssm put-parameter --name "/statuspage/prod/from_email" --value "noreply@yourdomain.com" --type "String"
aws ssm put-parameter --name "/statuspage/prod/admin_email" --value "admin@yourdomain.com" --type "String"

echo "Configuration stored in Parameter Store:"
echo "DB Host: $RDS_ENDPOINT"
echo "Redis Host: $REDIS_ENDPOINT"
echo "Domain: status.yourdomain.com"
```

---

## Phase 3: Status-Page Application Installation (Day 2-3, 4-6 hours)

### Step 3.1: Create Enhanced IAM Role for Status-Page
```bash
# Create trust policy
cat > statuspage-trust-policy.json << EOF
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": "ec2.amazonaws.com"
            },
            "Action": "sts:AssumeRole"
        }
    ]
}
EOF

# Create IAM role
aws iam create-role \
    --role-name StatusPageEC2Role \
    --assume-role-policy-document file://statuspage-trust-policy.json \
    --tags Key=Project,Value=StatusPage

# Create enhanced policy for Status-Page
cat > statuspage-ec2-policy.json << EOF
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ssm:GetParameter",
                "ssm:GetParameters",
                "ssm:GetParametersByPath"
            ],
            "Resource": "arn:aws:ssm:us-east-1:*:parameter/statuspage/*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:PutObject",
                "s3:DeleteObject",
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws:s3:::statuspage-static-*",
                "arn:aws:s3:::statuspage-static-*/*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "ses:SendEmail",
                "ses:SendRawEmail"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "sns:Publish"
            ],
            "Resource": "arn:aws:sns:us-east-1:*:statuspage-*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "cloudwatch:PutMetricData"
            ],
            "Resource": "*"
        }
    ]
}
EOF

aws iam put-role-policy \
    --role-name StatusPageEC2Role \
    --policy-name StatusPagePolicy \
    --policy-document file://statuspage-ec2-policy.json

# Attach SSM managed policy
aws iam attach-role-policy \
    --role-name StatusPageEC2Role \
    --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore

# Create instance profile
aws iam create-instance-profile --instance-profile-name StatusPageInstanceProfile
aws iam add-role-to-instance-profile \
    --instance-profile-name StatusPageInstanceProfile \
    --role-name StatusPageEC2Role

# Wait for instance profile to be ready
sleep 30
```

### Step 3.2: Create Enhanced User Data Script for Status-Page
```bash
cat > statuspage-user-data.sh << 'EOF'
#!/bin/bash
# Enhanced User Data Script for Status-Page

# Update system
dnf update -y

# Install Status-Page specific dependencies
dnf install -y \
    python3.10 python3.10-pip python3.10-devel \
    gcc gcc-c++ make cmake \
    libxml2-devel libxslt-devel \
    libffi-devel libjpeg-devel libpng-devel \
    postgresql15-devel openssl-devel \
    redis nginx git \
    graphviz graphviz-devel \
    openldap-devel \
    librdkafka-devel \
    supervisor

# Install Node.js for frontend assets (Status-Page may need it)
curl -fsSL https://rpm.nodesource.com/setup_18.x | bash -
dnf install -y nodejs

# Create status-page user with proper home directory
groupadd --system status-page
useradd --system -g status-page -d /opt/status-page -m status-page

# Set proper permissions
mkdir -p /opt/status-page/{media,static,plugins,logs}
chown -R status-page:status-page /opt/status-page
chmod 755 /opt/status-page

# Configure nginx basic settings
systemctl enable nginx
systemctl enable redis
systemctl start redis

# Create log rotation for Status-Page
cat > /etc/logrotate.d/statuspage << 'LOGROTATE_EOF'
/opt/status-page/logs/*.log {
    daily
    missingok
    rotate 52
    compress
    delaycompress
    notifempty
    create 644 status-page status-page
    postrotate
        systemctl reload statuspage > /dev/null 2>&1 || true
    endscript
}
LOGROTATE_EOF

# Install AWS CLI v2
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
./aws/install
rm -rf awscliv2.zip aws/

echo "Status-Page user data script completed" > /var/log/userdata.log
EOF
```

### Step 3.3: Launch Enhanced EC2 Instance for Status-Page
```bash
# Create key pair (optional, we'll use Session Manager)
aws ec2 create-key-pair \
    --key-name statuspage-key \
    --tag-specifications "ResourceType=key-pair,Tags=[{Key=Project,Value=StatusPage}]" \
    --query 'KeyMaterial' --output text > statuspage-key.pem
chmod 400 statuspage-key.pem

# Get the latest Amazon Linux 2023 AMI
AMI_ID=$(aws ec2 describe-images \
    --owners amazon \
    --filters "Name=name,Values=al2023-ami-*" "Name=architecture,Values=x86_64" "Name=state,Values=available" \
    --query 'Images | sort_by(@, &CreationDate) | [-1].ImageId' \
    --output text)

# Launch EC2 instance with enhanced specs for Status-Page
EC2_INSTANCE_ID=$(aws ec2 run-instances \
    --image-id $AMI_ID \
    --instance-type t3.small \
    --key-name statuspage-key \
    --security-group-ids $EC2_SG_ID \
    --subnet-id $PRIVATE_SUBNET_1_ID \
    --iam-instance-profile Name=StatusPageInstanceProfile \
    --user-data file://statuspage-user-data.sh \
    --block-device-mappings DeviceName=/dev/xvda,Ebs='{VolumeSize=30,VolumeType=gp3,Encrypted=true}' \
    --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=StatusPage-Instance},{Key=Project,Value=$PROJECT_NAME}]" \
    --metadata-options HttpTokens=required,HttpPutResponseHopLimit=2 \
    --query 'Instances[0].InstanceId' --output text)

echo "EC2 Instance ID: $EC2_INSTANCE_ID"

# Wait for instance to be running and ready
echo "Waiting for EC2 instance to be running..."
aws ec2 wait instance-running --instance-ids $EC2_INSTANCE_ID

echo "Waiting for status checks to pass..."
aws ec2 wait instance-status-ok --instance-ids $EC2_INSTANCE_ID
```

### Step 3.4: Install and Configure Status-Page Application
```bash
# Connect to instance using Session Manager
echo "Connecting to instance to install Status-Page..."
echo "Run this command to connect:"
echo "aws ssm start-session --target $EC2_INSTANCE_ID"

# The following commands should be run inside the EC2 instance
cat > install-statuspage.sh << 'EOF'
#!/bin/bash
# Run these commands inside the EC2 instance

# Switch to status-page user
sudo -i -u status-page

# Go to the application directory
cd /opt/status-page

# Clone the Status-Page repository
git clone https://github.com/Status-Page/Status-Page.git .

# Create Python virtual environment with Python 3.10
python3.10 -m venv venv
source venv/bin/activate

# Upgrade pip and install wheel
pip install --upgrade pip wheel setuptools

# Install Status-Page requirements
pip install -r requirements.txt

# Install additional production dependencies
pip install \
    gunicorn \
    boto3 \
    django-storages[boto3] \
    django-ses \
    psycopg2-binary \
    redis \
    channels \
    channels-redis \
    supervisor

# Create the configuration file
cp status_page/configuration_example.py status_page/configuration.py

# Create AWS-specific configuration
cat > status_page/aws_config.py << 'AWSEOF'
import boto3
import os

def get_parameter(name):
    """Get parameter from AWS Parameter Store"""
    try:
        ssm = boto3.client('ssm', region_name='us-east-1')
        response = ssm.get_parameter(Name=name, WithDecryption=True)
        return response['Parameter']['Value']
    except Exception as e:
        print(f"Error getting parameter {name}: {e}")
        return None

# Import base configuration
from .configuration_example import *

# Override with AWS-specific settings
ALLOWED_HOSTS = [
    get_parameter('/statuspage/prod/domain_name'),
    'localhost',
    '127.0.0.1'
]

# Database Configuration
DATABASE = {
    'NAME': 'statuspage',
    'USER': 'statuspage_admin', 
    'PASSWORD': get_parameter('/statuspage/prod/db_password'),
    'HOST': get_parameter('/statuspage/prod/db_host'),
    'PORT': '5432',
    'CONN_MAX_AGE': 300,
}

# Redis Configuration for Status-Page
REDIS = {
    'tasks': {
        'HOST': get_parameter('/statuspage/prod/redis_host'),
        'PORT': 6379,
        'PASSWORD': '',
        'DATABASE': 0,
        'SSL': False,
        'CONNECTION_POOL_KWARGS': {
            'max_connections': 50,
        },
    },
    'caching': {
        'HOST': get_parameter('/statuspage/prod/redis_host'),
        'PORT': 6379,
        'PASSWORD': '',
        'DATABASE': 1,
        'SSL': False,
        'CONNECTION_POOL_KWARGS': {
            'max_connections': 50,
        },
    }
}

# Secret Key
SECRET_KEY = get_parameter('/statuspage/prod/secret_key')

# Email Configuration for Status-Page notifications
EMAIL_BACKEND = 'django_ses.SESBackend'
AWS_SES_REGION_NAME = 'us-east-1'
AWS_SES_REGION_ENDPOINT = 'email.us-east-1.amazonaws.com'
DEFAULT_FROM_EMAIL = get_parameter('/statuspage/prod/from_email')
SERVER_EMAIL = get_parameter('/statuspage/prod/from_email')

# S3 Configuration for static files and media
AWS_STORAGE_BUCKET_NAME = get_parameter('/statuspage/prod/s3_bucket')
AWS_S3_REGION_NAME = 'us-east-1'
AWS_DEFAULT_ACL = None
AWS_S3_OBJECT_PARAMETERS = {
    'CacheControl': 'max-age=86400',
}

# Status-Page specific media handling
DEFAULT_FILE_STORAGE = 'storages.backends.s3boto3.S3Boto3Storage'
STATICFILES_STORAGE = 'storages.backends.s3boto3.S3StaticStorage'

# Plugin Configuration
PLUGINS_ROOT = '/opt/status-page/plugins'

# Logging Configuration
LOGGING = {
    'version': 1,
    'disable_existing_loggers': False,
    'formatters': {
        'verbose': {
            'format': '{levelname} {asctime} {module} {process:d} {thread:d} {message}',
            'style': '{',
        },
        'simple': {
            'format': '{levelname} {message}',
            'style': '{',
        },
    },
    'handlers': {
        'file': {
            'level': 'INFO',
            'class': 'logging.handlers.RotatingFileHandler',
            'filename': '/opt/status-page/logs/status-page.log',
            'formatter': 'verbose',
            'maxBytes': 1024*1024*15,  # 15MB
            'backupCount': 10,
        },
        'console': {
            'level': 'INFO',
            'class': 'logging.StreamHandler',
            'formatter': 'simple',
        },
    },
    'loggers': {
        'status_page': {
            'handlers': ['file', 'console'],
            'level': 'INFO',
            'propagate': False,
        },
        'django': {
            'handlers': ['file'],
            'level': 'WARNING',
            'propagate': False,
        },
    },
}

# Security Settings for Production
SECURE_SSL_REDIRECT = True
SECURE_PROXY_SSL_HEADER = ('HTTP_X_FORWARDED_PROTO', 'https')
USE_X_FORWARDED_HOST = True
USE_X_FORWARDED_PORT = True
SECURE_BROWSER_XSS_FILTER = True
SECURE_CONTENT_TYPE_NOSNIFF = True

# Status-Page specific settings
TIME_ZONE = 'UTC'
MAINTENANCE_MODE = False
BANNER_TOP = ''
BANNER_BOTTOM = ''
BANNER_LOGIN = ''

# API Settings for Status-Page
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework.authentication.TokenAuthentication',
        'rest_framework.authentication.SessionAuthentication',
    ],
    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.IsAuthenticated',
    ],
    'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.LimitOffsetPagination',
    'PAGE_SIZE': 50,
    'DEFAULT_THROTTLE_CLASSES': [
        'rest_framework.throttling.AnonRateThrottle',
        'rest_framework.throttling.UserRateThrottle'
    ],
    'DEFAULT_THROTTLE_RATES': {
        'anon': '100/hour',
        'user': '1000/hour'
    }
}

# Channels configuration for WebSocket support (if needed)
CHANNEL_LAYERS = {
    'default': {
        'BACKEND': 'channels_redis.core.RedisChannelLayer',
        'CONFIG': {
            'hosts': [(get_parameter('/statuspage/prod/redis_host'), 6379)],
        },
    },
}
AWSEOF

# Update configuration.py to use AWS settings
echo "from .aws_config import *" >> status_page/configuration.py

# Run database migrations
python manage.py migrate

# Create cache table if needed
python manage.py createcachetable

# Create superuser for Status-Page admin
echo "Creating superuser for Status-Page..."
python manage.py shell << 'PYEOF'
from django.contrib.auth import get_user_model
User = get_user_model()
if not User.objects.filter(username='admin').exists():
    User.objects.create_superuser('admin', 'admin@yourdomain.com', 'ChangeMe123!')
    print("Superuser created successfully")
else:
    print("Superuser already exists")
PYEOF

# Collect static files
python manage.py collectstatic --noinput

# Test the installation
echo "Testing Status-Page installation..."
python manage.py check

echo "Status-Page installation completed successfully!"
EOF

echo "Save this script and run it inside the EC2 instance after connecting via Session Manager"
```

### Step 3.5: Create Systemd Services for Status-Page
```bash
# Create these files on the EC2 instance after installation

# Gunicorn service for Status-Page
sudo tee /etc/systemd/system/statuspage.service << 'EOF'
[Unit]
Description=Status-Page WSGI Service
Documentation=https://docs.status-page.dev/
Wants=network-online.target
After=network-online.target postgresql.service redis.service

[Service]
Type=forking
User=status-page
Group=status-page
WorkingDirectory=/opt/status-page
Environment=PATH=/opt/status-page/venv/bin
Environment=DJANGO_SETTINGS_MODULE=status_page.settings
ExecStart=/opt/status-page/venv/bin/gunicorn \
    --config /opt/status-page/gunicorn.py \
    --bind 127.0.0.1:8000 \
    --workers 3 \
    --worker-class gthread \
    --threads 2 \
    --max-requests 1000 \
    --max-requests-jitter 100 \
    --preload \
    --user status-page \
    --group status-page \
    --log-level info \
    --access-logfile /opt/status-page/logs/gunicorn-access.log \
    --error-logfile /opt/status-page/logs/gunicorn-error.log \
    status_page.wsgi:application
ExecReload=/bin/kill -s HUP $MAINPID
KillMode=mixed
TimeoutStopSec=30
PrivateTmp=true
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

# Status-Page RQ Worker for background tasks
sudo tee /etc/systemd/system/statuspage-rq.service << 'EOF'
[Unit]
Description=Status-Page RQ Worker
After=network.target redis.service

[Service]
Type=simple
User=status-page
Group=status-page
WorkingDirectory=/opt/status-page
Environment=PATH=/opt/status-page/venv/bin
Environment=DJANGO_SETTINGS_MODULE=status_page.settings
ExecStart=/opt/status-page/venv/bin/python manage.py rqworker
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

# Status-Page Scheduler (if using RQ Scheduler)
sudo tee /etc/systemd/system/statuspage-scheduler.service << 'EOF'
[Unit]
Description=Status-Page RQ Scheduler
After=network.target redis.service

[Service]
Type=simple
User=status-page
Group=status-page
WorkingDirectory=/opt/status-page
Environment=PATH=/opt/status-page/venv/bin
Environment=DJANGO_SETTINGS_MODULE=status_page.settings
ExecStart=/opt/status-page/venv/bin/python manage.py rqscheduler
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

# Create Gunicorn configuration
sudo -u status-page tee /opt/status-page/gunicorn.py << 'EOF'
# Gunicorn configuration for Status-Page
import multiprocessing
import os

# Server socket
bind = "127.0.0.1:8000"
backlog = 2048

# Worker processes
workers = multiprocessing.cpu_count() * 2 + 1
worker_class = "gthread"
threads = 2
worker_connections = 1000
max_requests = 1000
max_requests_jitter = 100
preload_app = True
timeout = 30
keepalive = 2

# Security
limit_request_line = 4094
limit_request_fields = 100
limit_request_field_size = 8190

# Logging
accesslog = "/opt/status-page/logs/gunicorn-access.log"
errorlog = "/opt/status-page/logs/gunicorn-error.log"
loglevel = "info"
access_log_format = '%(h)s %(l)s %(u)s %(t)s "%(r)s" %(s)s %(b)s "%(f)s" "%(a)s" %(D)s'

# Process naming
proc_name = "statuspage-gunicorn"

# Server mechanics
daemon = False
pidfile = "/tmp/gunicorn-statuspage.pid"
user = "status-page"
group = "status-page"
tmp_upload_dir = "/tmp"

# SSL (handled by ALB, but keeping for reference)
# keyfile = "/path/to/keyfile"
# certfile = "/path/to/certfile"
EOF

# Enable and start services
sudo systemctl daemon-reload
sudo systemctl enable statuspage statuspage-rq statuspage-scheduler
sudo systemctl start statuspage statuspage-rq statuspage-scheduler

# Check service status
sudo systemctl status statuspage
```

---

## Phase 4: Load Balancer and SSL (Day 3, 2 hours)

### Step 4.1: Request SSL Certificate for Status-Page
```bash
# Request certificate from ACM for Status-Page domain
DOMAIN_NAME="status.yourdomain.com"
CERT_ARN=$(aws acm request-certificate \
    --domain-name $DOMAIN_NAME \
    --domain-name *.$DOMAIN_NAME \
    --validation-method DNS \
    --subject-alternative-names $DOMAIN_NAME \
    --tags Key=Name,Value=StatusPage-SSL-Cert Key=Project,Value=StatusPage \
    --query 'CertificateArn' --output text)

echo "Certificate ARN: $CERT_ARN"

# Get DNS validation records
echo "DNS validation records needed:"
aws acm describe-certificate --certificate-arn $CERT_ARN \
    --query 'Certificate.DomainValidationOptions[*].[DomainName,ResourceRecord.Name,ResourceRecord.Value]' \
    --output table

# Get Route 53 hosted zone ID
HOSTED_ZONE_ID=$(aws route53 list-hosted-zones-by-name \
    --dns-name yourdomain.com \
    --query 'HostedZones[0].Id' --output text | cut -d'/' -f3)

echo "Hosted Zone ID: $HOSTED_ZONE_ID"
echo "Add the CNAME records shown above to Route 53"

# Wait for certificate validation
echo "Waiting for certificate validation (this may take 5-30 minutes)..."
aws acm wait certificate-validated --certificate-arn $CERT_ARN --region us-east-1
```

### Step 4.2: Create Application Load Balancer for Status-Page
```bash
# Create ALB
ALB_ARN=$(aws elbv2 create-load-balancer \
    --name StatusPage-ALB \
    --subnets $PUBLIC_SUBNET_1_ID $PUBLIC_SUBNET_2_ID \
    --security-groups $ALB_SG_ID \
    --scheme internet-facing \
    --type application \
    --ip-address-type ipv4 \
    --tags Key=Name,Value=StatusPage-ALB Key=Project,Value=StatusPage \
    --query 'LoadBalancers[0].LoadBalancerArn' --output text)

echo "ALB ARN: $ALB_ARN"

# Get ALB DNS name
ALB_DNS=$(aws elbv2 describe-load-balancers \
    --load-balancer-arns $ALB_ARN \
    --query 'LoadBalancers[0].DNSName' --output text)

echo "ALB DNS: $ALB_DNS"

# Create Target Group for Status-Page
TARGET_GROUP_ARN=$(aws elbv2 create-target-group \
    --name StatusPage-TG \
    --protocol HTTP \
    --port 8000 \
    --vpc-id $VPC_ID \
    --target-type instance \
    --health-check-enabled \
    --health-check-path /health/ \
    --health-check-protocol HTTP \
    --health-check-port 8000 \
    --health-check-interval-seconds 30 \
    --health-check-timeout-seconds 5 \
    --healthy-threshold-count 2 \
    --unhealthy-threshold-count 3 \
    --matcher HttpCode=200 \
    --tags Key=Name,Value=StatusPage-TG Key=Project,Value=StatusPage \
    --query 'TargetGroups[0].TargetGroupArn' --output text)

echo "Target Group ARN: $TARGET_GROUP_ARN"

# Register EC2 instance with target group
aws elbv2 register-targets \
    --target-group-arn $TARGET_GROUP_ARN \
    --targets Id=$EC2_INSTANCE_ID,Port=8000

# Create HTTP listener (redirects to HTTPS)
HTTP_LISTENER_ARN=$(aws elbv2 create-listener \
    --load-balancer-arn $ALB_ARN \
    --protocol HTTP \
    --port 80 \
    --default-actions Type=redirect,RedirectConfig='{Protocol=HTTPS,Port=443,StatusCode=HTTP_301}' \
    --query 'Listeners[0].ListenerArn' --output text)

# Create HTTPS listener
HTTPS_LISTENER_ARN=$(aws elbv2 create-listener \
    --load-balancer-arn $ALB_ARN \
    --protocol HTTPS \
    --port 443 \
    --certificates CertificateArn=$CERT_ARN \
    --ssl-policy ELBSecurityPolicy-TLS-1-2-2017-01 \
    --default-actions Type=forward,TargetGroupArn=$TARGET_GROUP_ARN \
    --query 'Listeners[0].ListenerArn' --output text)

echo "HTTP Listener ARN: $HTTP_LISTENER_ARN"
echo "HTTPS Listener ARN: $HTTPS_LISTENER_ARN"
```

### Step 4.3: Update Route 53 DNS for Status-Page
```bash
# Get ALB hosted zone ID (for us-east-1)
ALB_HOSTED_ZONE_ID="Z35SXDOTRQ7X7K"

# Create alias record pointing to ALB
cat > statuspage-dns-record.json << EOF
{
    "Changes": [{
        "Action": "UPSERT",
        "ResourceRecordSet": {
            "Name": "status.yourdomain.com",
            "Type": "A",
            "AliasTarget": {
                "DNSName": "$ALB_DNS",
                "EvaluateTargetHealth": true,
                "HostedZoneId": "$ALB_HOSTED_ZONE_ID"
            }
        }
    }]
}
EOF

aws route53 change-resource-record-sets \
    --hosted-zone-id $HOSTED_ZONE_ID \
    --change-batch file://statuspage-dns-record.json

echo "DNS record created. Status-Page will be available at: https://status.yourdomain.com"
```

---

## Phase 5: S3, CloudFront and Static Files (Day 3, 1.5 hours)

### Step 5.1: Create S3 Bucket for Status-Page
```bash
# Create unique bucket name with timestamp
TIMESTAMP=$(date +%s)
BUCKET_NAME="statuspage-static-$TIMESTAMP"

# Create S3 bucket
aws s3 mb s3://$BUCKET_NAME --region us-east-1

# Store bucket name in Parameter Store
aws ssm put-parameter \
    --name "/statuspage/prod/s3_bucket" \
    --value "$BUCKET_NAME" \
    --type "String" \
    --overwrite

# Get AWS Account ID for bucket policy
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

# Create bucket policy for Status-Page access
cat > s3-statuspage-policy.json << EOF
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::$ACCOUNT_ID:role/StatusPageEC2Role"
            },
            "Action": [
                "s3:GetObject",
                "s3:PutObject",
                "s3:DeleteObject",
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws:s3:::$BUCKET_NAME",
                "arn:aws:s3:::$BUCKET_NAME/*"
            ]
        },
        {
            "Effect": "Allow",
            "Principal": {
                "Service": "cloudfront.amazonaws.com"
            },
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::$BUCKET_NAME/*"
        }
    ]
}
EOF

aws s3api put-bucket-policy --bucket $BUCKET_NAME --policy file://s3-statuspage-policy.json

# Configure bucket for static website hosting
aws s3api put-bucket-website \
    --bucket $BUCKET_NAME \
    --website-configuration IndexDocument={Suffix=index.html},ErrorDocument={Key=error.html}

# Enable versioning for rollback capabilities
aws s3api put-bucket-versioning \
    --bucket $BUCKET_NAME \
    --versioning-configuration Status=Enabled

echo "S3 Bucket created: $BUCKET_NAME"
```

### Step 5.2: Upload Status-Page Static Files
```bash
# Connect to EC2 instance to upload static files
echo "Connect to your EC2 instance and run:"
cat << 'EOF'
# Switch to status-page user
sudo -i -u status-page
cd /opt/status-page
source venv/bin/activate

# Collect all static files
python manage.py collectstatic --noinput --clear

# Upload static files to S3
BUCKET_NAME=$(aws ssm get-parameter --name "/statuspage/prod/s3_bucket" --query 'Parameter.Value' --output text)
aws s3 sync static/ s3://$BUCKET_NAME/static/ --delete --cache-control "max-age=86400"

# Upload media files (if any)
if [ -d "media" ]; then
    aws s3 sync media/ s3://$BUCKET_NAME/media/ --cache-control "max-age=3600"
fi

echo "Static files uploaded to S3"
EOF
```

### Step 5.3: Create CloudFront Distribution for Status-Page
```bash
# Create CloudFront origin access control
OAC_ID=$(aws cloudfront create-origin-access-control \
    --origin-access-control-config \
    Name=StatusPage-OAC,Description="OAC for Status-Page S3 bucket",OriginAccessControlOriginType=s3,SigningBehavior=always,SigningProtocol=sigv4 \
    --query 'OriginAccessControl.Id' --output text)

echo "Origin Access Control ID: $OAC_ID"

# Create CloudFront distribution configuration
cat > cloudfront-statuspage-config.json << EOF
{
    "CallerReference": "statuspage-$TIMESTAMP",
    "Comment": "Status-Page Static Content CDN",
    "DefaultRootObject": "index.html",
    "Origins": {
        "Quantity": 1,
        "Items": [
            {
                "Id": "S3-$BUCKET_NAME",
                "DomainName": "$BUCKET_NAME.s3.amazonaws.com",
                "S3OriginConfig": {
                    "OriginAccessIdentity": ""
                },
                "OriginAccessControlId": "$OAC_ID"
            }
        ]
    },
    "DefaultCacheBehavior": {
        "TargetOriginId": "S3-$BUCKET_NAME",
        "ViewerProtocolPolicy": "redirect-to-https",
        "MinTTL": 0,
        "DefaultTTL": 86400,
        "MaxTTL": 31536000,
        "Compress": true,
        "ForwardedValues": {
            "QueryString": false,
            "Cookies": {
                "Forward": "none"
            }
        },
        "TrustedSigners": {
            "Enabled": false,
            "Quantity": 0
        }
    },
    "CacheBehaviors": {
        "Quantity": 2,
        "Items": [
            {
                "PathPattern": "/static/*",
                "TargetOriginId": "S3-$BUCKET_NAME",
                "ViewerProtocolPolicy": "redirect-to-https",
                "MinTTL": 86400,
                "DefaultTTL": 86400,
                "MaxTTL": 31536000,
                "Compress": true,
                "ForwardedValues": {
                    "QueryString": false,
                    "Cookies": {
                        "Forward": "none"
                    }
                },
                "TrustedSigners": {
                    "Enabled": false,
                    "Quantity": 0
                }
            },
            {
                "PathPattern": "/media/*",
                "TargetOriginId": "S3-$BUCKET_NAME",
                "ViewerProtocolPolicy": "redirect-to-https",
                "MinTTL": 0,
                "DefaultTTL": 3600,
                "MaxTTL": 86400,
                "Compress": false,
                "ForwardedValues": {
                    "QueryString": false,
                    "Cookies": {
                        "Forward": "none"
                    }
                },
                "TrustedSigners": {
                    "Enabled": false,
                    "Quantity": 0
                }
            }
        ]
    },
    "Enabled": true,
    "PriceClass": "PriceClass_100",
    "ViewerCertificate": {
        "CloudFrontDefaultCertificate": true
    }
}
EOF

# Create CloudFront distribution
DISTRIBUTION_ID=$(aws cloudfront create-distribution \
    --distribution-config file://cloudfront-statuspage-config.json \
    --query 'Distribution.Id' --output text)

echo "CloudFront Distribution ID: $DISTRIBUTION_ID"
echo "Distribution will be available at: https://$DISTRIBUTION_ID.cloudfront.net"

# Wait for distribution to be deployed (this takes 10-15 minutes)
echo "Waiting for CloudFront distribution to be deployed..."
aws cloudfront wait distribution-deployed --id $DISTRIBUTION_ID
```

**Checkpoint 5**: Status-Page with CDN configured. Cost: $165/month

---

## Phase 6: Enhanced Auto Scaling and High Availability (Day 4, 2 hours)

### Step 6.1: Create AMI from Status-Page Instance
```bash
# Stop Status-Page services for clean AMI
aws ssm send-command \
    --document-name "AWS-RunShellScript" \
    --parameters 'commands=["sudo systemctl stop statuspage statuspage-rq statuspage-scheduler nginx"]' \
    --targets "Key=instanceids,Values=$EC2_INSTANCE_ID"

# Wait a moment for services to stop
sleep 30

# Create AMI
AMI_ID=$(aws ec2 create-image \
    --instance-id $EC2_INSTANCE_ID \
    --name "StatusPage-AMI-$(date +%Y%m%d-%H%M)" \
    --description "Status-Page Production AMI with full configuration" \
    --no-reboot \
    --tag-specifications "ResourceType=image,Tags=[{Key=Name,Value=StatusPage-AMI},{Key=Project,Value=StatusPage},{Key=Date,Value=$(date +%Y%m%d)}]" \
    --query 'ImageId' --output text)

echo "AMI ID: $AMI_ID"

# Restart services on original instance
aws ssm send-command \
    --document-name "AWS-RunShellScript" \
    --parameters 'commands=["sudo systemctl start statuspage statuspage-rq statuspage-scheduler nginx"]' \
    --targets "Key=instanceids,Values=$EC2_INSTANCE_ID"

# Wait for AMI to be available
echo "Waiting for AMI to be available..."
aws ec2 wait image-available --image-ids $AMI_ID
```

### Step 6.2: Create Launch Template for Status-Page
```bash
# Create launch template with enhanced configuration
cat > statuspage-launch-template.json << EOF
{
    "LaunchTemplateName": "StatusPage-LaunchTemplate",
    "LaunchTemplateData": {
        "ImageId": "$AMI_ID",
        "InstanceType": "t3.small",
        "SecurityGroupIds": ["$EC2_SG_ID"],
        "IamInstanceProfile": {
            "Name": "StatusPageInstanceProfile"
        },
        "BlockDeviceMappings": [
            {
                "DeviceName": "/dev/xvda",
                "Ebs": {
                    "VolumeSize": 30,
                    "VolumeType": "gp3",
                    "Encrypted": true,
                    "DeleteOnTermination": true
                }
            }
        ],
        "Monitoring": {
            "Enabled": true
        },
        "MetadataOptions": {
            "HttpTokens": "required",
            "HttpPutResponseHopLimit": 2,
            "HttpEndpoint": "enabled"
        },
        "UserData": "$(base64 -w 0 << 'USERDATA_EOF'
#!/bin/bash
# Start services on boot
systemctl enable statuspage statuspage-rq statuspage-scheduler nginx
systemctl start statuspage statuspage-rq statuspage-scheduler nginx

# Configure log rotation
systemctl enable logrotate

# Send startup notification
/opt/aws/bin/cfn-signal -e $? --stack StatusPage --resource AutoScalingGroup --region us-east-1 || true
USERDATA_EOF
)",
        "TagSpecifications": [
            {
                "ResourceType": "instance",
                "Tags": [
                    {
                        "Key": "Name",
                        "Value": "StatusPage-ASG-Instance"
                    },
                    {
                        "Key": "Project", 
                        "Value": "StatusPage"
                    }
                ]
            }
        ]
    }
}
EOF

# Create launch template
aws ec2 create-launch-template --cli-input-json file://statuspage-launch-template.json

echo "Launch Template created: StatusPage-LaunchTemplate"
```

### Step 6.3: Create Auto Scaling Group for Status-Page
```bash
# Create Auto Scaling Group
aws autoscaling create-auto-scaling-group \
    --auto-scaling-group-name StatusPage-ASG \
    --launch-template LaunchTemplateName=StatusPage-LaunchTemplate,Version='$Latest' \
    --min-size 1 \
    --max-size 3 \
    --desired-capacity 1 \
    --target-group-arns $TARGET_GROUP_ARN \
    --health-check-type ELB \
    --health-check-grace-period 300 \
    --vpc-zone-identifier "$PRIVATE_SUBNET_1_ID,$PRIVATE_SUBNET_2_ID" \
    --default-cooldown 300 \
    --tags "Key=Name,Value=StatusPage-ASG,PropagateAtLaunch=true,ResourceId=StatusPage-ASG,ResourceType=auto-scaling-group" \
           "Key=Project,Value=StatusPage,PropagateAtLaunch=true,ResourceId=StatusPage-ASG,ResourceType=auto-scaling-group"

# Create scaling policies
SCALE_UP_POLICY_ARN=$(aws autoscaling put-scaling-policy \
    --auto-scaling-group-name StatusPage-ASG \
    --policy-name StatusPage-ScaleUp \
    --policy-type TargetTrackingScaling \
    --target-tracking-configuration file://scale-up-policy.json \
    --query 'PolicyARN' --output text)

cat > scale-up-policy.json << EOF
{
    "TargetValue": 70.0,
    "PredefinedMetricSpecification": {
        "PredefinedMetricType": "ASGAverageCPUUtilization"
    },
    "DisableScaleIn": false,
    "ScaleOutCooldown": 300,
    "ScaleInCooldown": 300
}
EOF

aws autoscaling put-scaling-policy \
    --auto-scaling-group-name StatusPage-ASG \
    --policy-name StatusPage-ScaleUp \
    --policy-type TargetTrackingScaling \
    --target-tracking-configuration file://scale-up-policy.json

echo "Auto Scaling Group created with target tracking scaling policy"
```

**Checkpoint 6**: High availability configured. Cost: $165/month (same cost, better reliability)

Continue to Phase 7: Enhanced Monitoring and Security (Day 4, 2-3 hours) in the next part...
