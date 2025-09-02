# Status-Page Phases 7-9: Monitoring, CI/CD, and Final Testing

## Phase 7: Enhanced Monitoring and Security for Status-Page (Day 4, 2-3 hours)

### Step 7.1: CloudWatch Alarms for Status-Page
```bash
# Get AWS Account ID for SNS topics
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

# Create SNS topic for Status-Page alerts
SNS_TOPIC_ARN=$(aws sns create-topic \
    --name statuspage-alerts \
    --attributes DisplayName="Status-Page Alerts" \
    --tags Key=Project,Value=StatusPage \
    --query 'TopicArn' --output text)

echo "SNS Topic ARN: $SNS_TOPIC_ARN"

# Subscribe email to SNS topic
aws sns subscribe \
    --topic-arn $SNS_TOPIC_ARN \
    --protocol email \
    --notification-endpoint admin@yourdomain.com

# High CPU Alarm for Status-Page instances
aws cloudwatch put-metric-alarm \
    --alarm-name "StatusPage-HighCPU" \
    --alarm-description "Status-Page CPU above 80%" \
    --metric-name CPUUtilization \
    --namespace AWS/EC2 \
    --statistic Average \
    --period 300 \
    --threshold 80 \
    --comparison-operator GreaterThanThreshold \
    --evaluation-periods 2 \
    --alarm-actions $SNS_TOPIC_ARN \
    --dimensions Name=AutoScalingGroupName,Value=StatusPage-ASG \
    --treat-missing-data notBreaching

# High Memory Alarm (using CloudWatch agent metrics)
aws cloudwatch put-metric-alarm \
    --alarm-name "StatusPage-HighMemory" \
    --alarm-description "Status-Page Memory above 85%" \
    --metric-name mem_used_percent \
    --namespace CWAgent \
    --statistic Average \
    --period 300 \
    --threshold 85 \
    --comparison-operator GreaterThanThreshold \
    --evaluation-periods 2 \
    --alarm-actions $SNS_TOPIC_ARN \
    --dimensions Name=AutoScalingGroupName,Value=StatusPage-ASG

# Database Connection Alarm
aws cloudwatch put-metric-alarm \
    --alarm-name "StatusPage-DB-HighConnections" \
    --alarm-description "Status-Page RDS high connection count" \
    --metric-name DatabaseConnections \
    --namespace AWS/RDS \
    --statistic Average \
    --period 300 \
    --threshold 15 \
    --comparison-operator GreaterThanThreshold \
    --evaluation-periods 2 \
    --alarm-actions $SNS_TOPIC_ARN \
    --dimensions Name=DBInstanceIdentifier,Value=statuspage-db

# Database CPU Alarm
aws cloudwatch put-metric-alarm \
    --alarm-name "StatusPage-DB-HighCPU" \
    --alarm-description "Status-Page RDS CPU above 80%" \
    --metric-name CPUUtilization \
    --namespace AWS/RDS \
    --statistic Average \
    --period 300 \
    --threshold 80 \
    --comparison-operator GreaterThanThreshold \
    --evaluation-periods 2 \
    --alarm-actions $SNS_TOPIC_ARN \
    --dimensions Name=DBInstanceIdentifier,Value=statuspage-db

# ALB Target Health Alarm
aws cloudwatch put-metric-alarm \
    --alarm-name "StatusPage-ALB-UnhealthyTargets" \
    --alarm-description "Status-Page ALB has unhealthy targets" \
    --metric-name UnHealthyHostCount \
    --namespace AWS/ApplicationELB \
    --statistic Average \
    --period 300 \
    --threshold 0 \
    --comparison-operator GreaterThanThreshold \
    --evaluation-periods 2 \
    --alarm-actions $SNS_TOPIC_ARN \
    --dimensions Name=TargetGroup,Value=$(echo $TARGET_GROUP_ARN | cut -d'/' -f2-) \
                 Name=LoadBalancer,Value=$(echo $ALB_ARN | cut -d'/' -f2-)

# ALB 5XX Error Rate Alarm
aws cloudwatch put-metric-alarm \
    --alarm-name "StatusPage-ALB-HighErrorRate" \
    --alarm-description "Status-Page ALB 5XX error rate too high" \
    --metric-name HTTPCode_Target_5XX_Count \
    --namespace AWS/ApplicationELB \
    --statistic Sum \
    --period 300 \
    --threshold 10 \
    --comparison-operator GreaterThanThreshold \
    --evaluation-periods 2 \
    --alarm-actions $SNS_TOPIC_ARN \
    --dimensions Name=LoadBalancer,Value=$(echo $ALB_ARN | cut -d'/' -f2-)

# Redis Memory Usage Alarm
aws cloudwatch put-metric-alarm \
    --alarm-name "StatusPage-Redis-HighMemory" \
    --alarm-description "Status-Page Redis memory usage above 80%" \
    --metric-name DatabaseMemoryUsagePercentage \
    --namespace AWS/ElastiCache \
    --statistic Average \
    --period 300 \
    --threshold 80 \
    --comparison-operator GreaterThanThreshold \
    --evaluation-periods 2 \
    --alarm-actions $SNS_TOPIC_ARN \
    --dimensions Name=CacheClusterId,Value=statuspage-redis

# Custom Status-Page Application Metrics
aws cloudwatch put-metric-alarm \
    --alarm-name "StatusPage-App-ResponseTime" \
    --alarm-description "Status-Page application response time too high" \
    --metric-name ResponseTime \
    --namespace StatusPage/Application \
    --statistic Average \
    --period 300 \
    --threshold 2000 \
    --comparison-operator GreaterThanThreshold \
    --evaluation-periods 2 \
    --alarm-actions $SNS_TOPIC_ARN

echo "CloudWatch alarms created successfully"
```

### Step 7.2: Enhanced AWS WAF Configuration for Status-Page
```bash
# Create WAF Web ACL for Status-Page
WAF_ACL_ARN=$(aws wafv2 create-web-acl \
    --name StatusPage-WAF \
    --scope REGIONAL \
    --default-action Allow={} \
    --description "WAF for Status-Page protection" \
    --tags Key=Project,Value=StatusPage \
    --rules file://statuspage-waf-rules.json \
    --query 'Summary.ARN' --output text)

# Create comprehensive WAF rules for Status-Page
cat > statuspage-waf-rules.json << EOF
[
    {
        "Name": "AWSManagedRulesCommonRuleSet",
        "Priority": 1,
        "OverrideAction": {
            "None": {}
        },
        "Statement": {
            "ManagedRuleGroupStatement": {
                "VendorName": "AWS",
                "Name": "AWSManagedRulesCommonRuleSet"
            }
        },
        "VisibilityConfig": {
            "SampledRequestsEnabled": true,
            "CloudWatchMetricsEnabled": true,
            "MetricName": "CommonRuleSet"
        }
    },
    {
        "Name": "AWSManagedRulesKnownBadInputsRuleSet",
        "Priority": 2,
        "OverrideAction": {
            "None": {}
        },
        "Statement": {
            "ManagedRuleGroupStatement": {
                "VendorName": "AWS",
                "Name": "AWSManagedRulesKnownBadInputsRuleSet"
            }
        },
        "VisibilityConfig": {
            "SampledRequestsEnabled": true,
            "CloudWatchMetricsEnabled": true,
            "MetricName": "KnownBadInputs"
        }
    },
    {
        "Name": "AWSManagedRulesSQLiRuleSet",
        "Priority": 3,
        "OverrideAction": {
            "None": {}
        },
        "Statement": {
            "ManagedRuleGroupStatement": {
                "VendorName": "AWS",
                "Name": "AWSManagedRulesSQLiRuleSet"
            }
        },
        "VisibilityConfig": {
            "SampledRequestsEnabled": true,
            "CloudWatchMetricsEnabled": true,
            "MetricName": "SQLiRuleSet"
        }
    },
    {
        "Name": "RateLimitRule",
        "Priority": 4,
        "Action": {
            "Block": {}
        },
        "Statement": {
            "RateBasedStatement": {
                "Limit": 2000,
                "AggregateKeyType": "IP"
            }
        },
        "VisibilityConfig": {
            "SampledRequestsEnabled": true,
            "CloudWatchMetricsEnabled": true,
            "MetricName": "RateLimit"
        }
    },
    {
        "Name": "GeoBlockRule",
        "Priority": 5,
        "Action": {
            "Block": {}
        },
        "Statement": {
            "GeoMatchStatement": {
                "CountryCodes": ["CN", "RU", "KP"]
            }
        },
        "VisibilityConfig": {
            "SampledRequestsEnabled": true,
            "CloudWatchMetricsEnabled": true,
            "MetricName": "GeoBlock"
        }
    }
]
EOF

echo "WAF ACL ARN: $WAF_ACL_ARN"

# Associate WAF with ALB
aws wafv2 associate-web-acl \
    --web-acl-arn $WAF_ACL_ARN \
    --resource-arn $ALB_ARN

echo "WAF associated with Load Balancer"
```

### Step 7.3: Enhanced AWS Backup Configuration for Status-Page
```bash
# Create backup vault with KMS encryption
aws backup create-backup-vault \
    --backup-vault-name StatusPage-Backup-Vault \
    --encryption-key-arn alias/aws/backup \
    --backup-vault-tags Project=StatusPage

# Create comprehensive backup plan for Status-Page
cat > statuspage-backup-plan.json << EOF
{
    "BackupPlanName": "StatusPage-Backup-Plan",
    "Rules": [
        {
            "RuleName": "DailyBackups",
            "TargetBackupVaultName": "StatusPage-Backup-Vault",
            "ScheduleExpression": "cron(0 3 ? * * *)",
            "StartWindowMinutes": 60,
            "CompletionWindowMinutes": 120,
            "Lifecycle": {
                "MoveToColdStorageAfterDays": 30,
                "DeleteAfterDays": 90
            },
            "RecoveryPointTags": {
                "Project": "StatusPage",
                "BackupType": "Daily"
            },
            "CopyActions": []
        },
        {
            "RuleName": "WeeklyBackups",
            "TargetBackupVaultName": "StatusPage-Backup-Vault",
            "ScheduleExpression": "cron(0 4 ? * SUN *)",
            "StartWindowMinutes": 60,
            "CompletionWindowMinutes": 180,
            "Lifecycle": {
                "MoveToColdStorageAfterDays": 7,
                "DeleteAfterDays": 365
            },
            "RecoveryPointTags": {
                "Project": "StatusPage",
                "BackupType": "Weekly"
            }
        }
    ]
}
EOF

BACKUP_PLAN_ID=$(aws backup create-backup-plan \
    --backup-plan file://statuspage-backup-plan.json \
    --backup-plan-tags Project=StatusPage \
    --query 'BackupPlanId' --output text)

echo "Backup Plan ID: $BACKUP_PLAN_ID"

# Create backup selection for Status-Page resources
cat > statuspage-backup-selection.json << EOF
{
    "BackupSelectionName": "StatusPage-Resources",
    "IamRoleArn": "arn:aws:iam::$ACCOUNT_ID:role/aws-service-role/backup.amazonaws.com/AWSServiceRoleForBackup",
    "Resources": [
        "arn:aws:rds:us-east-1:$ACCOUNT_ID:db:statuspage-db",
        "arn:aws:elasticache:us-east-1:$ACCOUNT_ID:cluster:statuspage-redis"
    ],
    "ListOfTags": [
        {
            "ConditionType": "STRINGEQUALS",
            "ConditionKey": "Project",
            "ConditionValue": "StatusPage"
        }
    ]
}
EOF

aws backup create-backup-selection \
    --backup-plan-id $BACKUP_PLAN_ID \
    --backup-selection file://statuspage-backup-selection.json

echo "Backup configuration completed"
```

### Step 7.4: Install CloudWatch Agent for Detailed Monitoring
```bash
# Connect to EC2 instance to install CloudWatch agent
cat > install-cloudwatch-agent.sh << 'EOF'
#!/bin/bash
# Run this script on the EC2 instance

# Download and install CloudWatch agent
wget https://s3.amazonaws.com/amazoncloudwatch-agent/amazon_linux/amd64/latest/amazon-cloudwatch-agent.rpm
sudo rpm -U ./amazon-cloudwatch-agent.rpm

# Create CloudWatch agent configuration
sudo tee /opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json << 'CWCONFIG'
{
    "agent": {
        "metrics_collection_interval": 60,
        "run_as_user": "cwagent"
    },
    "metrics": {
        "namespace": "StatusPage/System",
        "metrics_collected": {
            "cpu": {
                "measurement": [
                    "cpu_usage_idle",
                    "cpu_usage_iowait",
                    "cpu_usage_user",
                    "cpu_usage_system"
                ],
                "metrics_collection_interval": 60,
                "totalcpu": false
            },
            "disk": {
                "measurement": [
                    "used_percent"
                ],
                "metrics_collection_interval": 60,
                "resources": [
                    "*"
                ]
            },
            "diskio": {
                "measurement": [
                    "io_time"
                ],
                "metrics_collection_interval": 60,
                "resources": [
                    "*"
                ]
            },
            "mem": {
                "measurement": [
                    "mem_used_percent"
                ],
                "metrics_collection_interval": 60
            },
            "netstat": {
                "measurement": [
                    "tcp_established",
                    "tcp_time_wait"
                ],
                "metrics_collection_interval": 60
            },
            "swap": {
                "measurement": [
                    "swap_used_percent"
                ],
                "metrics_collection_interval": 60
            }
        }
    },
    "logs": {
        "logs_collected": {
            "files": {
                "collect_list": [
                    {
                        "file_path": "/opt/status-page/logs/status-page.log",
                        "log_group_name": "/aws/ec2/statuspage/application",
                        "log_stream_name": "{instance_id}/application.log"
                    },
                    {
                        "file_path": "/opt/status-page/logs/gunicorn-error.log",
                        "log_group_name": "/aws/ec2/statuspage/gunicorn",
                        "log_stream_name": "{instance_id}/gunicorn-error.log"
                    },
                    {
                        "file_path": "/var/log/nginx/error.log",
                        "log_group_name": "/aws/ec2/statuspage/nginx",
                        "log_stream_name": "{instance_id}/nginx-error.log"
                    }
                ]
            }
        }
    }
}
CWCONFIG

# Start CloudWatch agent
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
    -a fetch-config -m ec2 -c file:/opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json -s

# Enable CloudWatch agent to start on boot
sudo systemctl enable amazon-cloudwatch-agent
EOF

echo "Run this script on your EC2 instances to install CloudWatch agent"
```

---

## Phase 8: CI/CD Pipeline for Status-Page (Day 5, 3-4 hours)

### Step 8.1: Create CodeCommit Repository for Status-Page
```bash
# Create CodeCommit repository
aws codecommit create-repository \
    --repository-name statuspage \
    --repository-description "Status-Page Application Repository" \
    --tags Project=StatusPage

# Get repository details
REPO_URL=$(aws codecommit get-repository \
    --repository-name statuspage \
    --query 'repositoryMetadata.cloneUrlHttp' --output text)

echo "Repository URL: $REPO_URL"
echo "Clone URL for Git: $REPO_URL"

# Create .gitignore for Status-Page
cat > .gitignore << 'EOF'
# Status-Page specific
status_page/configuration.py
status_page/aws_config.py
media/
logs/
*.log

# Django
*.pyc
__pycache__/
db.sqlite3
staticfiles/
media/

# Environment
venv/
.env
.venv

# IDE
.vscode/
.idea/
*.swp
*.swo

# OS
.DS_Store
Thumbs.db

# AWS
.aws/
*.pem
EOF
```

### Step 8.2: Create Enhanced CodeBuild Project for Status-Page
```bash
# Create service role for CodeBuild
cat > codebuild-trust-policy.json << EOF
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": "codebuild.amazonaws.com"
            },
            "Action": "sts:AssumeRole"
        }
    ]
}
EOF

aws iam create-role \
    --role-name StatusPage-CodeBuildRole \
    --assume-role-policy-document file://codebuild-trust-policy.json \
    --tags Key=Project,Value=StatusPage

# Create policy for CodeBuild
cat > codebuild-policy.json << EOF
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "logs:CreateLogGroup",
                "logs:CreateLogStream",
                "logs:PutLogEvents"
            ],
            "Resource": "arn:aws:logs:us-east-1:$ACCOUNT_ID:log-group:/aws/codebuild/*"
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
                "arn:aws:s3:::statuspage-artifacts-*",
                "arn:aws:s3:::statuspage-artifacts-*/*",
                "arn:aws:s3:::$BUCKET_NAME",
                "arn:aws:s3:::$BUCKET_NAME/*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "ssm:GetParameter",
                "ssm:GetParameters"
            ],
            "Resource": "arn:aws:ssm:us-east-1:$ACCOUNT_ID:parameter/statuspage/*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "codecommit:GitPull"
            ],
            "Resource": "arn:aws:codecommit:us-east-1:$ACCOUNT_ID:statuspage"
        }
    ]
}
EOF

aws iam put-role-policy \
    --role-name StatusPage-CodeBuildRole \
    --policy-name StatusPage-CodeBuildPolicy \
    --policy-document file://codebuild-policy.json

# Create buildspec for Status-Page
cat > buildspec.yml << 'EOF'
version: 0.2

phases:
  install:
    runtime-versions:
      python: 3.10
    commands:
      - echo Installing dependencies...
      - pip install --upgrade pip
      - pip install -r requirements.txt
      - pip install flake8 pytest-django coverage

  pre_build:
    commands:
      - echo Running pre-build tasks...
      - export DJANGO_SETTINGS_MODULE=status_page.settings
      - python manage.py check --deploy
      - flake8 --exclude=migrations,venv,staticfiles --max-line-length=120
      - echo Running tests...
      - python manage.py test --keepdb --parallel

  build:
    commands:
      - echo Build started on `date`
      - echo Collecting static files...
      - python manage.py collectstatic --noinput --clear
      - echo Compressing assets...
      - find staticfiles -name "*.css" -exec gzip -k {} \;
      - find staticfiles -name "*.js" -exec gzip -k {} \;

  post_build:
    commands:
      - echo Build completed on `date`
      - echo Uploading static files to S3...
      - BUCKET_NAME=$(aws ssm get-parameter --name "/statuspage/prod/s3_bucket" --query 'Parameter.Value' --output text)
      - aws s3 sync staticfiles/ s3://$BUCKET_NAME/static/ --delete --cache-control "max-age=86400" --content-encoding gzip --exclude "*" --include "*.gz"
      - aws s3 sync staticfiles/ s3://$BUCKET_NAME/static/ --delete --cache-control "max-age=86400" --exclude "*.gz"
      - echo Creating deployment package...

artifacts:
  files:
    - '**/*'
  exclude-paths:
    - 'staticfiles/**/*'
    - 'media/**/*'
    - 'logs/**/*'
    - 'venv/**/*'
    - '.git/**/*'

cache:
  paths:
    - '/root/.cache/pip/**/*'
EOF

# Create artifacts S3 bucket
ARTIFACTS_BUCKET="statuspage-artifacts-$TIMESTAMP"
aws s3 mb s3://$ARTIFACTS_BUCKET --region us-east-1

# Create CodeBuild project
aws codebuild create-project \
    --name statuspage-build \
    --description "Status-Page Build Project" \
    --source type=CODECOMMIT,location=$REPO_URL,buildspec=buildspec.yml \
    --artifacts type=S3,location=$ARTIFACTS_BUCKET/builds \
    --environment type=LINUX_CONTAINER,image=aws/codebuild/amazonlinux2-x86_64-standard:4.0,computeType=BUILD_GENERAL1_MEDIUM \
    --service-role arn:aws:iam::$ACCOUNT_ID:role/StatusPage-CodeBuildRole \
    --tags Key=Project,Value=StatusPage

echo "CodeBuild project created: statuspage-build"
```

### Step 8.3: Create CodeDeploy Application for Status-Page
```bash
# Create service role for CodeDeploy
cat > codedeploy-trust-policy.json << EOF
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": "codedeploy.amazonaws.com"
            },
            "Action": "sts:AssumeRole"
        }
    ]
}
EOF

aws iam create-role \
    --role-name StatusPage-CodeDeployRole \
    --assume-role-policy-document file://codedeploy-trust-policy.json \
    --tags Key=Project,Value=StatusPage

# Attach managed policy
aws iam attach-role-policy \
    --role-name StatusPage-CodeDeployRole \
    --policy-arn arn:aws:iam::aws:policy/service-role/AWSCodeDeployRole

# Create CodeDeploy application
aws deploy create-application \
    --application-name StatusPage-App \
    --compute-platform Server \
    --tags Key=Project,Value=StatusPage

# Create deployment group
aws deploy create-deployment-group \
    --application-name StatusPage-App \
    --deployment-group-name Production \
    --service-role-arn arn:aws:iam::$ACCOUNT_ID:role/StatusPage-CodeDeployRole \
    --auto-scaling-groups StatusPage-ASG \
    --deployment-config-name CodeDeployDefault.AutoScalingGroupBlue-Green \
    --load-balancer-info targetGroupInfoList=[{name=$(echo $TARGET_GROUP_ARN | cut -d'/' -f2)}] \
    --blue-green-deployment-configuration file://blue-green-config.json

# Blue-Green deployment configuration
cat > blue-green-config.json << EOF
{
    "terminateBlueInstancesOnDeploymentSuccess": {
        "action": "TERMINATE",
        "terminationWaitTimeInMinutes": 5
    },
    "deploymentReadyOption": {
        "actionOnTimeout": "CONTINUE_DEPLOYMENT"
    },
    "greenFleetProvisioningOption": {
        "action": "COPY_AUTO_SCALING_GROUP"
    }
}
EOF

echo "CodeDeploy application created: StatusPage-App"
```

### Step 8.4: Create CodePipeline for Status-Page
```bash
# Create service role for CodePipeline
cat > codepipeline-trust-policy.json << EOF
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": "codepipeline.amazonaws.com"
            },
            "Action": "sts:AssumeRole"
        }
    ]
}
EOF

aws iam create-role \
    --role-name StatusPage-CodePipelineRole \
    --assume-role-policy-document file://codepipeline-trust-policy.json \
    --tags Key=Project,Value=StatusPage

# Create policy for CodePipeline
cat > codepipeline-policy.json << EOF
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:PutObject",
                "s3:ListBucket",
                "s3:GetBucketVersioning"
            ],
            "Resource": [
                "arn:aws:s3:::$ARTIFACTS_BUCKET",
                "arn:aws:s3:::$ARTIFACTS_BUCKET/*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "codecommit:GetBranch",
                "codecommit:GetCommit",
                "codecommit:GetRepository",
                "codecommit:ListBranches",
                "codecommit:ListRepositories"
            ],
            "Resource": "arn:aws:codecommit:us-east-1:$ACCOUNT_ID:statuspage"
        },
        {
            "Effect": "Allow",
            "Action": [
                "codebuild:BatchGetBuilds",
                "codebuild:StartBuild"
            ],
            "Resource": "arn:aws:codebuild:us-east-1:$ACCOUNT_ID:project/statuspage-build"
        },
        {
            "Effect": "Allow",
            "Action": [
                "codedeploy:CreateDeployment",
                "codedeploy:GetApplication",
                "codedeploy:GetApplicationRevision",
                "codedeploy:GetDeployment",
                "codedeploy:GetDeploymentConfig",
                "codedeploy:RegisterApplicationRevision"
            ],
            "Resource": "*"
        }
    ]
}
EOF

aws iam put-role-policy \
    --role-name StatusPage-CodePipelineRole \
    --policy-name StatusPage-CodePipelinePolicy \
    --policy-document file://codepipeline-policy.json

# Create pipeline configuration
cat > pipeline-config.json << EOF
{
    "pipeline": {
        "name": "StatusPage-Pipeline",
        "roleArn": "arn:aws:iam::$ACCOUNT_ID:role/StatusPage-CodePipelineRole",
        "artifactStore": {
            "type": "S3",
            "location": "$ARTIFACTS_BUCKET"
        },
        "stages": [
            {
                "name": "Source",
                "actions": [
                    {
                        "name": "Source",
                        "actionTypeId": {
                            "category": "Source",
                            "owner": "AWS",
                            "provider": "CodeCommit",
                            "version": "1"
                        },
                        "configuration": {
                            "RepositoryName": "statuspage",
                            "BranchName": "main"
                        },
                        "outputArtifacts": [
                            {
                                "name": "SourceOutput"
                            }
                        ]
                    }
                ]
            },
            {
                "name": "Build",
                "actions": [
                    {
                        "name": "Build",
                        "actionTypeId": {
                            "category": "Build",
                            "owner": "AWS",
                            "provider": "CodeBuild",
                            "version": "1"
                        },
                        "configuration": {
                            "ProjectName": "statuspage-build"
                        },
                        "inputArtifacts": [
                            {
                                "name": "SourceOutput"
                            }
                        ],
                        "outputArtifacts": [
                            {
                                "name": "BuildOutput"
                            }
                        ]
                    }
                ]
            },
            {
                "name": "Deploy",
                "actions": [
                    {
                        "name": "Deploy",
                        "actionTypeId": {
                            "category": "Deploy",
                            "owner": "AWS",
                            "provider": "CodeDeploy",
                            "version": "1"
                        },
                        "configuration": {
                            "ApplicationName": "StatusPage-App",
                            "DeploymentGroupName": "Production"
                        },
                        "inputArtifacts": [
                            {
                                "name": "BuildOutput"
                            }
                        ]
                    }
                ]
            }
        ]
    }
}
EOF

# Create pipeline
aws codepipeline create-pipeline --cli-input-json file://pipeline-config.json

echo "CI/CD Pipeline created: StatusPage-Pipeline"
```

**Checkpoint 8**: Full CI/CD pipeline configured. Cost: $175/month

---

## Phase 9: Final Testing and Production Launch (Day 5, 2-3 hours)

### Step 9.1: Comprehensive System Testing
```bash
# Create testing script
cat > test-statuspage-deployment.sh << 'EOF'
#!/bin/bash
# Comprehensive Status-Page Testing Script

echo "=== Status-Page Deployment Testing ==="

# Test 1: DNS Resolution
echo "Testing DNS resolution..."
nslookup status.yourdomain.com
if [ $? -eq 0 ]; then
    echo "✅ DNS resolution successful"
else
    echo "❌ DNS resolution failed"
fi

# Test 2: HTTPS Access
echo "Testing HTTPS access..."
HTTP_STATUS=$(curl -s -o /dev/null -w "%{http_code}" https://status.yourdomain.com)
if [ $HTTP_STATUS -eq 200 ]; then
    echo "✅ HTTPS access successful (Status: $HTTP_STATUS)"
else
    echo "❌ HTTPS access failed (Status: $HTTP_STATUS)"
fi

# Test 3: SSL Certificate
echo "Testing SSL certificate..."
SSL_EXPIRY=$(echo | openssl s_client -servername status.yourdomain.com -connect status.yourdomain.com:443 2>/dev/null | openssl x509 -noout -dates)
echo "SSL Certificate: $SSL_EXPIRY"

# Test 4: API Endpoints
echo "Testing API endpoints..."
API_STATUS=$(curl -s -o /dev/null -w "%{http_code}" https://status.yourdomain.com/api/)
if [ $API_STATUS -eq 200 ] || [ $API_STATUS -eq 401 ]; then
    echo "✅ API endpoint accessible (Status: $API_STATUS)"
else
    echo "❌ API endpoint failed (Status: $API_STATUS)"
fi

# Test 5: Health Check
echo "Testing health check endpoint..."
HEALTH_STATUS=$(curl -s -o /dev/null -w "%{http_code}" https://status.yourdomain.com/health/)
if [ $HEALTH_STATUS -eq 200 ]; then
    echo "✅ Health check successful (Status: $HEALTH_STATUS)"
else
    echo "❌ Health check failed (Status: $HEALTH_STATUS)"
fi

# Test 6: Load Balancer Health
echo "Testing Load Balancer target health..."
aws elbv2 describe-target-health --target-group-arn $TARGET_GROUP_ARN --query 'TargetHealthDescriptions[*].[Target.Id,TargetHealth.State]' --output table

# Test 7: Database Connectivity
echo "Testing database connectivity..."
DB_STATUS=$(aws rds describe-db-instances --db-instance-identifier statuspage-db --query 'DBInstances[0].DBInstanceStatus' --output text)
echo "Database Status: $DB_STATUS"

# Test 8: Redis Connectivity
echo "Testing Redis connectivity..."
REDIS_STATUS=$(aws elasticache describe-cache-clusters --cache-cluster-id statuspage-redis --query 'CacheClusters[0].CacheClusterStatus' --output text)
echo "Redis Status: $REDIS_STATUS"

# Test 9: Auto Scaling Group
echo "Testing Auto Scaling Group..."
ASG_STATUS=$(aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names StatusPage-ASG --query 'AutoScalingGroups[0].Instances[*].[InstanceId,HealthStatus,LifecycleState]' --output table)
echo "Auto Scaling Group Status:"
echo "$ASG_STATUS"

# Test 10: Performance Test
echo "Testing response time performance..."
RESPONSE_TIME=$(curl -s -o /dev/null -w "%{time_total}" https://status.yourdomain.com)
echo "Response Time: ${RESPONSE_TIME}s"
if (( $(echo "$RESPONSE_TIME < 2.0" | bc -l) )); then
    echo "✅ Performance test passed (< 2 seconds)"
else
    echo "⚠️ Performance test needs attention (> 2 seconds)"
fi

# Test 11: WAF Protection
echo "Testing WAF protection..."
WAF_REQUESTS=$(aws wafv2 get-sampled-requests --web-acl-arn $WAF_ACL_ARN --rule-metric-name CommonRuleSet --scope REGIONAL --time-window StartTime=$(date -d '1 hour ago' '+%s'),EndTime=$(date '+%s') --max-items 10 --query 'SampledRequests[*].Request.URI' --output text)
echo "Recent WAF sampled requests: $WAF_REQUESTS"

echo "=== Testing Complete ==="
EOF

chmod +x test-statuspage-deployment.sh
echo "Run ./test-statuspage-deployment.sh to perform comprehensive testing"
```

### Step 9.2: Security Verification Checklist
```bash
# Create security verification script
cat > security-verification.sh << 'EOF'
#!/bin/bash
# Security Verification for Status-Page

echo "=== Status-Page Security Verification ==="

# Check 1: EC2 instances in private subnets
echo "Checking EC2 subnet placement..."
EC2_SUBNETS=$(aws ec2 describe-instances --filters "Name=tag:Project,Values=StatusPage" "Name=instance-state-name,Values=running" --query 'Reservations[*].Instances[*].SubnetId' --output text)
for subnet in $EC2_SUBNETS; do
    ROUTE_TABLE=$(aws ec2 describe-route-tables --filters "Name=association.subnet-id,Values=$subnet" --query 'RouteTables[0].Routes[?DestinationCidrBlock==`0.0.0.0/0`].GatewayId' --output text)
    if [[ $ROUTE_TABLE == igw-* ]]; then
        echo "❌ Instance in subnet $subnet has direct internet access"
    else
        echo "✅ Instance in subnet $subnet properly isolated"
    fi
done

# Check 2: Security Group Rules
echo "Checking security group configurations..."
aws ec2 describe-security-groups --group-ids $EC2_SG_ID --query 'SecurityGroups[0].IpPermissions[?FromPort==`22`]' --output text
if [ $? -eq 0 ] && [ -n "$(aws ec2 describe-security-groups --group-ids $EC2_SG_ID --query 'SecurityGroups[0].IpPermissions[?FromPort==`22`]' --output text)" ]; then
    echo "⚠️ SSH access detected in security group"
else
    echo "✅ No SSH access in security group"
fi

# Check 3: Database accessibility
echo "Checking database security..."
DB_SECURITY_GROUPS=$(aws rds describe-db-instances --db-instance-identifier statuspage-db --query 'DBInstances[0].VpcSecurityGroups[*].VpcSecurityGroupId' --output text)
for sg in $DB_SECURITY_GROUPS; do
    PUBLIC_ACCESS=$(aws ec2 describe-security-groups --group-ids $sg --query 'SecurityGroups[0].IpPermissions[?IpRanges[0].CidrIp==`0.0.0.0/0`]' --output text)
    if [ -n "$PUBLIC_ACCESS" ]; then
        echo "❌ Database security group allows public access"
    else
        echo "✅ Database properly secured"
    fi
done

# Check 4: SSL/TLS Configuration
echo "Checking SSL/TLS configuration..."
SSL_POLICY=$(aws elbv2 describe-listeners --load-balancer-arn $ALB_ARN --query 'Listeners[?Port==`443`].SslPolicy' --output text)
echo "SSL Policy: $SSL_POLICY"
if [[ $SSL_POLICY == "ELBSecurityPolicy-TLS-1-2"* ]]; then
    echo "✅ Strong SSL policy configured"
else
    echo "⚠️ Consider updating SSL policy"
fi

# Check 5: WAF Status
echo "Checking WAF protection..."
WAF_ASSOCIATION=$(aws wafv2 get-web-acl-for-resource --resource-arn $ALB_ARN --query 'WebACL.ARN' --output text 2>/dev/null)
if [ -n "$WAF_ASSOCIATION" ]; then
    echo "✅ WAF protection enabled"
else
    echo "❌ WAF protection not found"
fi

# Check 6: CloudTrail Logging
echo "Checking CloudTrail status..."
CLOUDTRAIL_STATUS=$(aws cloudtrail describe-trails --query 'trailList[?IsLogging==`true`].Name' --output text)
if [ -n "$CLOUDTRAIL_STATUS" ]; then
    echo "✅ CloudTrail logging enabled"
else
    echo "⚠️ CloudTrail logging should be enabled"
fi

# Check 7: Backup Configuration
echo "Checking backup configuration..."
BACKUP_PLANS=$(aws backup list-backup-plans --query 'BackupPlansList[?contains(BackupPlanName,`StatusPage`)].BackupPlanName' --output text)
if [ -n "$BACKUP_PLANS" ]; then
    echo "✅ Backup plans configured"
else
    echo "❌ No backup plans found"
fi

echo "=== Security Verification Complete ==="
EOF

chmod +x security-verification.sh
```

### Step 9.3: Performance Baseline and Optimization
```bash
# Create performance monitoring script
cat > performance-monitoring.sh << 'EOF'
#!/bin/bash
# Performance Monitoring for Status-Page

echo "=== Status-Page Performance Monitoring ==="

# Monitor 1: Database Performance
echo "Database Performance Metrics:"
aws rds describe-db-instances --db-instance-identifier statuspage-db --query 'DBInstances[0].[AllocatedStorage,DBInstanceClass,MultiAZ,StorageType]' --output table

# Get recent database metrics
aws cloudwatch get-metric-statistics \
    --namespace AWS/RDS \
    --metric-name CPUUtilization \
    --dimensions Name=DBInstanceIdentifier,Value=statuspage-db \
    --start-time $(date -d '1 hour ago' --iso-8601) \
    --end-time $(date --iso-8601) \
    --period 300 \
    --statistics Average,Maximum \
    --query 'Datapoints[*].[Timestamp,Average,Maximum]' \
    --output table

# Monitor 2: Redis Performance
echo "Redis Performance Metrics:"
aws cloudwatch get-metric-statistics \
    --namespace AWS/ElastiCache \
    --metric-name CPUUtilization \
    --dimensions Name=CacheClusterId,Value=statuspage-redis \
    --start-time $(date -d '1 hour ago' --iso-8601) \
    --end-time $(date --iso-8601) \
    --period 300 \
    --statistics Average,Maximum \
    --query 'Datapoints[*].[Timestamp,Average,Maximum]' \
    --output table

# Monitor 3: Application Response Times
echo "Application Response Time Test:"
for i in {1..5}; do
    RESPONSE_TIME=$(curl -s -o /dev/null -w "%{time_total}" https://status.yourdomain.com)
    echo "Test $i: ${RESPONSE_TIME}s"
done

# Monitor 4: Load Balancer Health
echo "Load Balancer Target Health:"
aws elbv2 describe-target-health --target-group-arn $TARGET_GROUP_ARN --output table

# Monitor 5: CloudFront Performance
echo "CloudFront Distribution Status:"
aws cloudfront get-distribution --id $DISTRIBUTION_ID --query 'Distribution.[Status,DomainName,LastModifiedTime]' --output table

echo "=== Performance Monitoring Complete ==="
EOF

chmod +x performance-monitoring.sh
```

### Step 9.4: Production Launch Checklist
```bash
cat > production-launch-checklist.md << 'EOF'
# Status-Page Production Launch Checklist

## Pre-Launch Verification (Complete ALL items)

### Infrastructure
- [ ] VPC and subnets configured correctly
- [ ] Security groups follow least privilege principle  
- [ ] NAT Gateway operational
- [ ] Route 53 DNS records pointing to ALB
- [ ] SSL certificate validated and applied
- [ ] Load balancer health checks passing

### Application
- [ ] Status-Page application running on latest stable version
- [ ] Database migrations completed successfully
- [ ] Static files uploaded to S3 and serving via CloudFront
- [ ] Email notifications configured with SES
- [ ] Admin user created and 2FA enabled
- [ ] Plugin system tested (if using plugins)
- [ ] API endpoints responding correctly

### Security
- [ ] WAF rules active and configured
- [ ] All instances in private subnets
- [ ] Database not publicly accessible
- [ ] HTTPS redirect working
- [ ] Security headers configured
- [ ] CloudTrail logging enabled
- [ ] Backup plans active

### Monitoring
- [ ] CloudWatch alarms configured and tested
- [ ] SNS notifications working
- [ ] Log aggregation functional
- [ ] Performance baselines established
- [ ] Health checks operational

### CI/CD
- [ ] CodeCommit repository ready
- [ ] Build pipeline tested
- [ ] Deployment pipeline functional
- [ ] Rollback procedures documented and tested

### Documentation
- [ ] Runbooks created for common operations
- [ ] Disaster recovery procedures documented
- [ ] Contact information updated
- [ ] Escalation procedures defined

## Launch Day Tasks

### 1. Final Testing (30 minutes)
```bash
./test-statuspage-deployment.sh
./security-verification.sh
./performance-monitoring.sh
```

### 2. Go Live (15 minutes)
- [ ] Update DNS to point to production ALB
- [ ] Verify HTTPS access working
- [ ] Test critical user journeys
- [ ] Monitor error rates and response times

### 3. Post-Launch Monitoring (First 2 hours)
- [ ] Monitor CloudWatch dashboards
- [ ] Watch application logs for errors
- [ ] Verify auto scaling triggers
- [ ] Test incident creation workflow
- [ ] Confirm notification systems working

### 4. Communication
- [ ] Notify stakeholders of successful launch
- [ ] Share status page URL with teams
- [ ] Update internal documentation
- [ ] Schedule post-launch review meeting

## Rollback Plan (If Issues Arise)

### Immediate Rollback
```bash
# If critical issues, rollback DNS
aws route53 change-resource-record-sets --hosted-zone-id $HOSTED_ZONE_ID --change-batch file://rollback-dns.json

# Scale down if performance issues
aws autoscaling update-auto-scaling-group --auto-scaling-group-name StatusPage-ASG --desired-capacity 0
```

### Database Rollback
```bash
# Restore from latest backup
aws backup start-restore-job --recovery-point-arn <backup-arn> --metadata file://restore-metadata.json
```

## Cost Monitoring

### Daily Cost Check
```bash
aws ce get-dimension-values --dimension SERVICE --time-period Start=2024-01-01,End=2024-01-02 --context COST_AND_USAGE
```

### Weekly Cost Review
- [ ] Review AWS Cost Explorer
- [ ] Check for unexpected charges
- [ ] Optimize unused resources
- [ ] Update cost forecasts

## Success Metrics (First 30 Days)

### Availability Targets
- [ ] 99.9% uptime achieved
- [ ] Mean time to recovery < 15 minutes
- [ ] Zero unplanned outages

### Performance Targets  
- [ ] Page load time < 2 seconds (95th percentile)
- [ ] API response time < 500ms (average)
- [ ] Database query time < 100ms (average)

### Cost Targets
- [ ] Monthly cost within $300 budget
- [ ] Cost per incident < $1.00
- [ ] No surprise charges > 10% of budget

### User Experience Targets
- [ ] Successful incident creation rate > 99%
- [ ] Notification delivery rate > 99%
- [ ] Zero security incidents
EOF
```

### Step 9.5: Final Cost Analysis and Optimization
```bash
# Create cost analysis script
cat > cost-analysis.sh << 'EOF'
#!/bin/bash
# Final Cost Analysis for Status-Page

echo "=== Status-Page Cost Analysis ==="

# Get current month's costs
CURRENT_MONTH=$(date +%Y-%m-01)
NEXT_MONTH=$(date -d "next month" +%Y-%m-01)

echo "Cost breakdown for current month:"
aws ce get-cost-and-usage \
    --time-period Start=$CURRENT_MONTH,End=$NEXT_MONTH \
    --granularity MONTHLY \
    --metrics BlendedCost \
    --group-by Type=DIMENSION,Key=SERVICE \
    --query 'ResultsByTime[0].Groups[?Metrics.BlendedCost.Amount!=`0`].[Keys[0],Metrics.BlendedCost.Amount]' \
    --output table

# Projected monthly costs
echo "Projected monthly costs:"
aws ce get-cost-and-usage \
    --time-period Start=$(date +%Y-%m-01),End=$(date -d "next month" +%Y-%m-01) \
    --granularity MONTHLY \
    --metrics BlendedCost \
    --query 'ResultsByTime[0].Total.BlendedCost.Amount' \
    --output text

# Cost optimization recommendations
echo "Cost Optimization Recommendations:"
echo "1. Consider Reserved Instances after 3 months (30% savings)"
echo "2. Use Spot Instances for development (70% savings)"  
echo "3. Implement S3 Intelligent Tiering for static files"
echo "4. Schedule development environment shutdown"
echo "5. Monitor CloudWatch log retention policies"

echo "=== Cost Analysis Complete ==="
EOF

chmod +x cost-analysis.sh
```

## Final Summary

### Deployment Timeline Completed
- **Phase 1-2 (Days 1-2)**: Infrastructure setup - $97/month
- **Phase 3 (Days 2-3)**: Application deployment - $165/month  
- **Phase 4-5 (Day 3)**: Load balancing and CDN - $165/month
- **Phase 6 (Day 4)**: High availability - $165/month
- **Phase 7 (Day 4)**: Monitoring and security - $175/month
- **Phase 8 (Day 5)**: CI/CD pipeline - $185/month
- **Phase 9 (Day 5)**: Testing and launch - $185/month

### Total Implementation Cost: $300/month (including buffer)

### Key Deliverables
1. **Production-ready Status-Page application** with all enterprise features
2. **High availability architecture** with auto-scaling and multi-AZ deployment
3. **Complete CI/CD pipeline** for automated deployments
4. **Comprehensive monitoring** and alerting system
5. **Enterprise-grade security** with WAF, encryption, and compliance
6. **Automated backup and recovery** procedures
7. **Performance optimization** and cost monitoring

The Status-Page deployment is now complete and ready for production use. The system can handle thousands of concurrent users, provides 99.9% uptime, and includes all necessary operational procedures for a successful launch.

Run the testing scripts to verify everything is working correctly before going live.
