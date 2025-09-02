# Status-Page AWS Deployment Guide
## Repository: https://github.com/Status-Page/Status-Page/

### Application Overview
This is the official Status-Page project - a professional open-source status page software with enterprise features including:
- Components monitoring and incident reporting
- JSON API for integrations  
- Real-time metrics and dashboards
- Two-Factor Authentication
- Markdown support for incident messages
- Email/SMS notification subscriptions
- Custom plugin support
- Multi-language support

### Updated System Requirements
Based on the official Status-Page repository:

| Dependency | Minimum Version | Status |
|-----------|----------------|--------|
| Python | 3.10+ | Required |
| PostgreSQL | 12+ | Required |
| Redis | 4.0+ | Required |
| SMTP Mail Server | Any | Optional |

### Key Differences from Generic Django Apps
1. **NetBox-based architecture** - Uses NetBox patterns for scalability
2. **Advanced plugin system** - Requires proper plugin directory structure  
3. **Complex notification system** - Needs SMTP/Email configuration
4. **Real-time features** - Requires WebSocket support for live updates
5. **API-heavy** - Extensive REST API for external integrations

---

## Updated AWS Architecture for Status-Page

### Enhanced EC2 Configuration
```yaml
Instance Type: t3.small (minimum for Status-Page)
Memory: 2GB minimum (Status-Page is more resource-intensive)
Storage: 30GB EBS (for logs, plugins, media files)
Python Version: 3.10+ (strict requirement)
```

### Redis Configuration (Enhanced)
```yaml
ElastiCache Redis:
  Instance: cache.t3.micro → cache.t3.small (recommended)
  Reason: Status-Page uses Redis heavily for:
    - Session management
    - Real-time notifications
    - API caching
    - Background task queuing
    - WebSocket session storage
```

### PostgreSQL Configuration (Enhanced)
```yaml
RDS PostgreSQL:
  Instance: db.t3.micro → db.t3.small (recommended)
  Version: PostgreSQL 14+ (Status-Page optimized)
  Storage: 30GB minimum (for metrics data)
  Backup: 14 days (for compliance requirements)
```

### Additional AWS Services Required
```yaml
SES (Simple Email Service): $1/month
  - For notification emails
  - Incident alerts
  - User registration emails
  - Subscription confirmations

SNS (Simple Notification Service): $2/month
  - SMS notifications (optional)
  - Push notifications
  - Webhook notifications

CloudWatch Events: $5/month
  - Scheduled health checks
  - Automated incident detection
  - Metrics collection triggers
```

---

## Updated Installation Process for Amazon Linux 2023

### Step 1: Enhanced System Package Installation
```bash
#!/bin/bash
# Enhanced User Data Script for Status-Page

# Update system
sudo dnf update -y

# Install Status-Page specific dependencies
sudo dnf install -y \
    python3.10 python3.10-pip python3.10-devel \
    gcc gcc-c++ make \
    libxml2-devel libxslt-devel \
    libffi-devel libjpeg-devel libpng-devel \
    postgresql-devel openssl-devel \
    redis nginx git \
    graphviz graphviz-devel \
    libldap-devel libsasl2-devel

# Install Node.js for frontend assets
curl -fsSL https://rpm.nodesource.com/setup_18.x | sudo bash -
sudo dnf install -y nodejs

# Create status-page user
sudo groupadd --system status-page
sudo useradd --system -g status-page status-page -d /opt/status-page

# Create directories with proper permissions
sudo mkdir -p /opt/status-page/{media,static,plugins}
sudo chown -R status-page:status-page /opt/status-page
```

### Step 2: Clone and Setup Status-Page
```bash
# Switch to status-page user
sudo su - status-page
cd /opt/status-page

# Clone the official repository
git clone https://github.com/Status-Page/Status-Page.git .

# Create virtual environment with Python 3.10
python3.10 -m venv venv
source venv/bin/activate

# Upgrade pip and install wheel
pip install --upgrade pip wheel

# Install Status-Page requirements
pip install -r requirements.txt

# Install additional production dependencies
pip install gunicorn boto3 django-storages[boto3] redis psycopg2-binary
```

### Step 3: Status-Page Specific Configuration
```python
# /opt/status-page/status_page/configuration.py
import os
import boto3
from .configuration_example import *

def get_parameter(name):
    ssm = boto3.client('ssm', region_name='us-east-1')
    response = ssm.get_parameter(Name=name, WithDecryption=True)
    return response['Parameter']['Value']

# Required Status-Page Configuration
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

# Redis Configuration (Status-Page specific)
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

# Email Configuration (Status-Page notifications)
EMAIL_BACKEND = 'django_ses.SESBackend'
AWS_SES_REGION_NAME = 'us-east-1'
AWS_SES_REGION_ENDPOINT = 'email.us-east-1.amazonaws.com'
DEFAULT_FROM_EMAIL = get_parameter('/statuspage/prod/from_email')
SERVER_EMAIL = get_parameter('/statuspage/prod/from_email')

# Media and Static Files (S3)
AWS_STORAGE_BUCKET_NAME = get_parameter('/statuspage/prod/s3_bucket')
AWS_S3_REGION_NAME = 'us-east-1'
AWS_DEFAULT_ACL = None
AWS_S3_OBJECT_PARAMETERS = {
    'CacheControl': 'max-age=86400',
}

# Status-Page specific media handling
DEFAULT_FILE_STORAGE = 'django_storages.backends.s3boto3.S3Boto3Storage'
STATICFILES_STORAGE = 'django_storages.backends.s3boto3.S3StaticStorage'

# Status-Page Plugin Directory
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
    },
    'handlers': {
        'file': {
            'level': 'INFO',
            'class': 'logging.FileHandler',
            'filename': '/opt/status-page/logs/status-page.log',
            'formatter': 'verbose',
        },
        'console': {
            'level': 'INFO',
            'class': 'logging.StreamHandler',
            'formatter': 'verbose',
        },
    },
    'loggers': {
        'status_page': {
            'handlers': ['file', 'console'],
            'level': 'INFO',
            'propagate': False,
        },
    },
}

# Time Zone
TIME_ZONE = 'UTC'

# Status-Page specific settings
MAINTENANCE_MODE = False
BANNER_TOP = ''
BANNER_BOTTOM = ''
BANNER_LOGIN = ''

# API Settings
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
}

# Security Settings
SECURE_SSL_REDIRECT = True
SECURE_PROXY_SSL_HEADER = ('HTTP_X_FORWARDED_PROTO', 'https')
USE_X_FORWARDED_HOST = True
USE_X_FORWARDED_PORT = True
SECURE_BROWSER_XSS_FILTER = True
SECURE_CONTENT_TYPE_NOSNIFF = True
```

### Step 4: Database and Initial Setup
```bash
# Create logs directory
mkdir -p /opt/status-page/logs

# Run database migrations
cd /opt/status-page
source venv/bin/activate
python manage.py migrate

# Create superuser for Status-Page admin
python manage.py createsuperuser

# Collect static files
python manage.py collectstatic --noinput

# Load initial data (if available)
python manage.py loaddata initial_data

# Test the installation
python manage.py runserver 0.0.0.0:8000
```

---

## Updated AWS Parameter Store Configuration

### Required Parameters for Status-Page
```bash
# Database parameters
aws ssm put-parameter --name "/statuspage/prod/db_host" --value "$RDS_ENDPOINT" --type "String"
aws ssm put-parameter --name "/statuspage/prod/db_password" --value "$DB_PASSWORD" --type "SecureString"

# Redis parameters
aws ssm put-parameter --name "/statuspage/prod/redis_host" --value "$REDIS_ENDPOINT" --type "String"

# Application parameters
aws ssm put-parameter --name "/statuspage/prod/secret_key" --value "$SECRET_KEY" --type "SecureString"
aws ssm put-parameter --name "/statuspage/prod/domain_name" --value "status.yourdomain.com" --type "String"

# Email parameters (for notifications)
aws ssm put-parameter --name "/statuspage/prod/from_email" --value "noreply@yourdomain.com" --type "String"

# S3 parameters
aws ssm put-parameter --name "/statuspage/prod/s3_bucket" --value "$BUCKET_NAME" --type "String"
```

---

## Updated Cost Analysis

### Enhanced Infrastructure Costs
```yaml
Production Environment: $165/month (increased from $136.50)
├── EC2 t3.small: $15/month
├── RDS db.t3.small: $20/month (upgraded from micro)
├── ElastiCache t3.small: $18/month (upgraded from micro)
├── Application Load Balancer: $18/month
├── NAT Gateway: $32/month
├── S3 Standard: $8/month (increased for media files)
├── CloudFront: $12/month (increased traffic)
├── Route 53: $0.50/month
├── CloudWatch: $15/month (enhanced monitoring)
├── WAF: $15/month
├── SES (Email): $1/month
├── SNS (Notifications): $2/month
├── AWS Backup: $8/month (larger snapshots)

Development Environment: $35/month (same)
CI/CD & DevOps: $30/month (same)
Buffer: $70/month (reduced due to higher base costs)

Total: $300/month (same budget)
```

---

## Status-Page Specific Features Configuration

### 1. Email Notifications Setup
```bash
# Configure SES for email notifications
aws ses verify-email-identity --email-address noreply@yourdomain.com
aws ses verify-domain-identity --domain yourdomain.com

# Create SES configuration set for tracking
aws ses create-configuration-set --configuration-set Name=statuspage-notifications
```

### 2. Plugin System Setup
```bash
# Create plugin directory structure
sudo mkdir -p /opt/status-page/plugins/{installed,custom}
sudo chown -R status-page:status-page /opt/status-page/plugins

# Set plugin permissions in configuration.py
PLUGINS_ROOT = '/opt/status-page/plugins'
PLUGINS_CONFIG = {
    'installed': '/opt/status-page/plugins/installed',
    'custom': '/opt/status-page/plugins/custom',
}
```

### 3. API Rate Limiting (for heavy API usage)
```python
# Add to configuration.py
REST_FRAMEWORK.update({
    'DEFAULT_THROTTLE_CLASSES': [
        'rest_framework.throttling.AnonRateThrottle',
        'rest_framework.throttling.UserRateThrottle'
    ],
    'DEFAULT_THROTTLE_RATES': {
        'anon': '100/hour',
        'user': '1000/hour'
    }
})
```

### 4. WebSocket Support (for real-time updates)
```bash
# Install additional requirements for WebSocket support
pip install channels channels-redis

# Update configuration for WebSocket support
INSTALLED_APPS += ['channels']
ASGI_APPLICATION = 'status_page.routing.application'
CHANNEL_LAYERS = {
    'default': {
        'BACKEND': 'channels_redis.core.RedisChannelLayer',
        'CONFIG': {
            'hosts': [('redis-endpoint', 6379)],
        },
    },
}
```

---

## Monitoring and Alerting for Status-Page

### Application-Specific Metrics
```bash
# CloudWatch custom metrics for Status-Page
aws cloudwatch put-metric-alarm \
    --alarm-name "StatusPage-ActiveIncidents" \
    --alarm-description "Number of active incidents" \
    --metric-name ActiveIncidents \
    --namespace StatusPage/Application \
    --statistic Average \
    --period 300 \
    --threshold 1 \
    --comparison-operator GreaterThanThreshold

# API Response Time Monitoring
aws cloudwatch put-metric-alarm \
    --alarm-name "StatusPage-API-ResponseTime" \
    --alarm-description "API response time too high" \
    --metric-name ResponseTime \
    --namespace StatusPage/API \
    --statistic Average \
    --period 300 \
    --threshold 2000 \
    --comparison-operator GreaterThanThreshold
```

### Health Check Endpoints
Status-Page provides these health check endpoints:
- `/health/` - Basic health check
- `/api/status/` - API status check  
- `/api/health/database/` - Database connectivity
- `/api/health/redis/` - Redis connectivity

---

## Security Considerations for Status-Page

### 1. Two-Factor Authentication
Enable 2FA in the admin panel after deployment:
- Navigate to Admin Panel → Users → Enable 2FA
- Configure TOTP settings
- Test 2FA with admin user

### 2. API Security
```python
# Enhanced API security in configuration.py
REST_FRAMEWORK.update({
    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.IsAuthenticated',
    ],
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework.authentication.TokenAuthentication',
    ],
})

# Rate limiting per API endpoint
API_RATE_LIMITS = {
    'incidents': '50/hour',
    'components': '100/hour',
    'metrics': '200/hour',
}
```

### 3. Content Security Policy
```python
# CSP headers for XSS protection
SECURE_CONTENT_SECURITY_POLICY = (
    "default-src 'self'; "
    "script-src 'self' 'unsafe-inline' https://cdn.jsdelivr.net; "
    "style-src 'self' 'unsafe-inline' https://fonts.googleapis.com; "
    "font-src 'self' https://fonts.gstatic.com; "
    "img-src 'self' data: https:; "
)
```

This updated deployment guide is specifically tailored for the Status-Page application from the GitHub repository you provided, with all the enhanced features and requirements properly configured for AWS deployment.
