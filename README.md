# Node.js ECS Fargate CI/CD Pipeline - DevOps Implementation



## 📋 Overview

This guide provides step-by-step instructions to set up a complete **DevOps CI/CD pipeline** for deploying the Node.js HelloWorld application from GitHub to **AWS ECS Fargate** using **AWS CodeBuild**.

> **Prerequisites**: Application code is already available in GitHub repository with proper structure including `buildspec.yml`, `Dockerfile`, and Node.js application files.

## 🏗️ DevOps Architecture

```
GitHub Repository  →  AWS CodeBuild  →  AWS ECR  →  AWS ECS Fargate
     │                      │              │            │
  Source Code         Build & Test     Docker Images   Running App
  buildspec.yml       Docker Build     Version Tags    Auto-scaling
  Dockerfile          Push to ECR      Image Registry  Health Checks
```

## 🚀 Implementation Steps


## 🔧 Step 0: Install and Configure AWS CLI


### 0.1 Install AWS CLI

**macOS (Homebrew):**
\`\`\`bash
brew install awscli
\`\`\`

**Ubuntu/Debian Linux:**
\`\`\`bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
\`\`\`

**Windows:**
Download the installer from [AWS CLI for Windows](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2-windows.html)

**Verify installation:**
\`\`\`bash
aws --version
\`\`\`

---

### 0.2 Configure AWS CLI

Run the following command to set up your credentials:

\`\`\`bash
aws configure
\`\`\`

You’ll be prompted to enter:

- AWS Access Key ID  
- AWS Secret Access Key  
- Default region (e.g., \`us-east-1\`)  
- Default output format (e.g., \`json\`)

**Test it:**
\`\`\`bash
aws sts get-caller-identity
\`\`\`

---

### Step 1: AWS Infrastructure Setup



#### 1.1 Create Core AWS Resources

```bash
# Create ECR Repository for Docker images
aws ecr create-repository \
    --repository-name nodejs-hello-world-ecs \
    --region us-east-1

# Create ECS Cluster with Fargate capacity
aws ecs create-cluster \
    --cluster-name nodejs-hello-world-ecs-cluster \
    --capacity-providers FARGATE \
    --default-capacity-provider-strategy capacityProvider=FARGATE,weight=1

# Create CloudWatch Log Group for application logs
aws logs create-log-group \
    --log-group-name /ecs/nodejs-hello-world-ecs \
    --region us-east-1
```

#### 1.2 Verify Resource Creation

```bash
# Verify ECR repository
aws ecr describe-repositories --repository-names nodejs-hello-world-ecs

# Verify ECS cluster
aws ecs describe-clusters --clusters nodejs-hello-world-ecs-cluster

# Verify CloudWatch log group
aws logs describe-log-groups --log-group-name-prefix "/ecs/nodejs-hello-world-ecs"
```

### Step 2: IAM Roles Configuration

#### 2.1 Create ECS Task Execution Role

```bash
# Create trust policy for ECS tasks
cat > ecs-task-trust-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "",
      "Effect": "Allow",
      "Principal": {
        "Service": "ecs-tasks.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

# Create the ECS task execution role
aws iam create-role \
  --role-name ecsTaskExecutionRole \
  --assume-role-policy-document file://ecs-task-trust-policy.json

# Attach the required AWS managed policy
aws iam attach-role-policy \
  --role-name ecsTaskExecutionRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy
```

#### 2.2 Create CodeBuild Service Role

```bash
# Create trust policy for CodeBuild
cat > codebuild-trust-policy.json << 'EOF'
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

# Create CodeBuild service role
aws iam create-role \
  --role-name NodejsHelloWorldCodeBuildRole \
  --assume-role-policy-document file://codebuild-trust-policy.json

# Create custom policy for CodeBuild with required permissions
cat > codebuild-policy.json << 'EOF'
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
      "Resource": "arn:aws:logs:*:*:*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "ecr:BatchCheckLayerAvailability",
        "ecr:GetDownloadUrlForLayer",
        "ecr:BatchGetImage",
        "ecr:GetAuthorizationToken",
        "ecr:PutImage",
        "ecr:InitiateLayerUpload",
        "ecr:UploadLayerPart",
        "ecr:CompleteLayerUpload"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "ecs:RegisterTaskDefinition",
        "ecs:UpdateService",
        "ecs:DescribeServices",
        "ecs:DescribeTaskDefinition",
        "ecs:DescribeClusters",
        "ecs:CreateService",
        "ecs:ListTasks",
        "ecs:DescribeTasks"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "iam:PassRole"
      ],
      "Resource": "arn:aws:iam::*:role/ecsTaskExecutionRole"
    },
    {
      "Effect": "Allow",
      "Action": [
        "ssm:GetParameter",
        "ssm:GetParameters"
      ],
      "Resource": "arn:aws:ssm:*:*:parameter/codebuild/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "ec2:DescribeVpcs",
        "ec2:DescribeSubnets",
        "ec2:DescribeSecurityGroups",
        "ec2:CreateSecurityGroup",
        "ec2:AuthorizeSecurityGroupIngress",
        "ec2:DescribeNetworkInterfaces"
      ],
      "Resource": "*"
    }
  ]
}
EOF

# Get your AWS account ID
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

# Create and attach the policy
aws iam create-policy \
  --policy-name NodejsHelloWorldCodeBuildPolicy \
  --policy-document file://codebuild-policy.json

# Attach the policy to the role
aws iam attach-role-policy \
  --role-name NodejsHelloWorldCodeBuildRole \
  --policy-arn arn:aws:iam::${ACCOUNT_ID}:policy/NodejsHelloWorldCodeBuildPolicy
```

### Step 3: Parameter Store Configuration

Store sensitive configuration values in AWS Systems Manager Parameter Store for secure access:

```bash
# Get your AWS account ID
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

# Store configuration parameters
aws ssm put-parameter \
  --name "/codebuild/aws-account-id" \
  --value "$ACCOUNT_ID" \
  --type "String" \
  --description "AWS Account ID for CodeBuild"

aws ssm put-parameter \
  --name "/codebuild/ecr-repository-name" \
  --value "nodejs-hello-world-ecs" \
  --type "String" \
  --description "ECR Repository Name"

aws ssm put-parameter \
  --name "/codebuild/ecs-cluster-name" \
  --value "nodejs-hello-world-ecs-cluster" \
  --type "String" \
  --description "ECS Cluster Name"

aws ssm put-parameter \
  --name "/codebuild/ecs-service-name" \
  --value "nodejs-hello-world-ecs-service" \
  --type "String" \
  --description "ECS Service Name"

aws ssm put-parameter \
  --name "/codebuild/ecs-task-definition-family" \
  --value "nodejs-hello-world-ecs-task" \
  --type "String" \
  --description "ECS Task Definition Family"

aws ssm put-parameter \
  --name "DOCKERHUB_USERNAME" \
  --value "your docker username" \
  --type "String" \
  --description "Username for odcker login"

aws ssm put-parameter \
  --name "DOCKERHUB_PASSWORD" \
  --value "your docker pass" \
  --type "String" \
  --description "[pass] for odcker login"

```

#### Verify Parameter Store Configuration

```bash
# List all parameters
aws ssm get-parameters-by-path \
  --path "/codebuild" \
  --query "Parameters[*].{Name:Name,Value:Value}"
```

### Step 4: CodeBuild Project Setup

#### 4.1 Create CodeBuild Project Configuration

```bash
# Create CodeBuild project configuration
cat > codebuild-project.json << EOF
{
  "name": "nodejs-hello-world-ecs-build",
  "description": "Build project for Node.js HelloWorld ECS application",
  "source": {
    "type": "GITHUB",
    "location": "https://github.com/YOUR_USERNAME/nodejs-hello-world-ecs.git",
    "gitCloneDepth": 1,
    "buildspec": "buildspec.yml",
    "auth": {
      "type": "OAUTH"
    }
  },
  "artifacts": {
    "type": "NO_ARTIFACTS"
  },
  "environment": {
    "type": "LINUX_CONTAINER",
    "image": "aws/codebuild/amazonlinux2-x86_64-standard:4.0",
    "computeType": "BUILD_GENERAL1_MEDIUM",
    "privilegedMode": true
  },
  "serviceRole": "arn:aws:iam::${ACCOUNT_ID}:role/NodejsHelloWorldCodeBuildRole"
}
EOF

# Replace placeholders with actual values
sed -i "s/YOUR_USERNAME/YOUR_ACTUAL_GITHUB_USERNAME/g" codebuild-project.json
sed -i "s/\${ACCOUNT_ID}/$ACCOUNT_ID/g" codebuild-project.json
```

#### 4.2 Create the CodeBuild Project

```bash
# Create the CodeBuild project
aws codebuild create-project --cli-input-json file://codebuild-project.json

# Verify project creation
aws codebuild batch-get-projects --names nodejs-hello-world-ecs-build
```

#### 4.3 Setup GitHub Webhook for Automatic Builds

```bash
# Create webhook for automatic builds on push to main branch
aws codebuild create-webhook \
  --project-name nodejs-hello-world-ecs-build-c \
  --filter-groups '[
    [
      {
        "type": "EVENT",
        "pattern": "PUSH"
      },
      {
        "type": "HEAD_REF",
        "pattern": "^refs/heads/main$"
      }
    ]
  ]'

# Verify webhook creation
aws codebuild batch-get-projects \
  --names nodejs-hello-world-ecs-build-c \
  --query "projects[0].webhook"
```

### Step 5: Initial Deployment

#### 5.1 Trigger First Build

```bash
# Start the initial build manually
BUILD_ID=$(aws codebuild start-build \
  --project-name nodejs-hello-world-ecs-build-c \
  --query "build.id" \
  --output text)

echo "Build started with ID: $BUILD_ID"
```

#### 5.2 Monitor Build Progress

```bash
# Monitor build status
aws codebuild batch-get-builds --ids "$BUILD_ID" \
  --query "builds[0].{Status:buildStatus,Phase:currentPhase,StartTime:startTime}"

# Follow build logs (replace LOG_GROUP and LOG_STREAM with actual values from build)
aws logs tail /aws/codebuild/nodejs-hello-world-ecs-build --follow
```

### Step 6: Verify Deployment

#### 6.1 Check ECS Service Status

```bash
# Check if ECS service was created and is running
aws ecs describe-services \
  --cluster nodejs-hello-world-ecs-cluster \
  --services nodejs-hello-world-ecs-service-c \
  --query "services[0].{Status:status,Running:runningCount,Desired:desiredCount,TaskDefinition:taskDefinition}"

# List running tasks
aws ecs list-tasks \
  --cluster nodejs-hello-world-ecs-cluster \
  --service-name nodejs-hello-world-ecs-service-c
```

#### 6.2 Get Application Access URL

```bash
# Get the task ARN
TASK_ARN=$(aws ecs list-tasks \
  --cluster nodejs-hello-world-ecs-cluster \
  --service-name nodejs-hello-world-ecs-service-c \
  --query "taskArns[0]" \
  --output text)

# Get the public IP address
PUBLIC_IP=$(aws ecs describe-tasks \
  --cluster nodejs-hello-world-ecs-cluster \
  --tasks $TASK_ARN \
  --query "tasks[0].attachments[0].details[?name=='networkInterfaceId'].value" \
  --output text | xargs -I {} aws ec2 describe-network-interfaces \
  --network-interface-ids {} \
  --query "NetworkInterfaces[0].Association.PublicIp" \
  --output text)

echo "🎉 Application is accessible at:"
echo "📱 Main App: http://$PUBLIC_IP:3000"
echo "🏥 Health Check: http://$PUBLIC_IP:3000/health"
```


## 📊 buildspec.yml Configuration Explained

The `buildspec.yml` file in your repository defines the CI/CD pipeline phases:

### **Pre-build Phase**
- **ECR Authentication**: Logs Docker into AWS ECR using temporary credentials
- **Environment Setup**: Configures repository URI, commit hash for tagging
- **Variable Preparation**: Sets up image tags and build metadata

### **Build Phase**
- **Docker Build**: Creates container image using multi-stage Dockerfile
- **Image Tagging**: Tags image with commit hash and latest for version control
- **Build Optimization**: Leverages Docker layer caching for faster builds

### **Post-build Phase**
- **Image Push**: Pushes tagged images to ECR repository
- **Task Definition Update**: Dynamically creates ECS task definition with new image
- **Service Update**: Updates ECS service to deploy new containers
- **Health Validation**: Ensures successful deployment through health checks

### **Key Features**
- **Zero-downtime Deployment**: Rolling updates with health checks
- **Automated Service Creation**: Creates ECS service and security groups if needed
- **Version Tracking**: Commit-based image tagging for rollbacks
- **Security**: Retrieves configuration from Parameter Store, no hardcoded values

## 🔧 Container Resource Specifications

### **Compute Resources**
- **CPU**: 256 units (0.25 vCPU) - Optimized for Node.js applications
- **Memory**: 512 MB - Sufficient for Express server with dependencies
- **Network**: awsvpc mode with public IP for external access
- **Storage**: 20 GB ephemeral storage (default)

### **Container Configuration**
- **Port Mapping**: Container port 3000 exposed for web traffic
- **Environment Variables**: NODE_ENV, APP_VERSION, PORT
- **Health Check**: HTTP endpoint `/health` with retry logic
- **Logging**: CloudWatch logs with structured log streams
- **Restart Policy**: Automatic restart on failure

### **Security Configuration**
- **User**: Non-root user execution (nodejs:1001)
- **Network**: VPC security groups controlling access
- **Secrets**: Environment-based configuration, no embedded secrets
- **Image**: Alpine Linux base for minimal attack surface


## 🧹 Resource Cleanup

When you're done with the project, clean up resources to avoid charges:

### **Quick Cleanup**

```bash
# Stop ECS service
aws ecs update-service \
  --cluster nodejs-hello-world-ecs-cluster \
  --service nodejs-hello-world-ecs-service-c \
  --desired-count 0

# Delete ECS service
aws ecs delete-service \
  --cluster nodejs-hello-world-ecs-cluster \
  --service nodejs-hello-world-ecs-service-c \
  --force

# Delete ECS cluster
aws ecs delete-cluster --cluster nodejs-hello-world-ecs-cluster

# Delete CodeBuild project
aws codebuild delete-project --name nodejs-hello-world-ecs-build

# Delete ECR repository
aws ecr delete-repository --repository-name nodejs-hello-world-ecs --force

# Delete CloudWatch logs
aws logs delete-log-group --log-group-name /ecs/nodejs-hello-world-ecs
```

### **Complete Cleanup Script**

For a comprehensive cleanup including IAM roles and Parameter Store values, use the automated cleanup script provided in the full documentation.

## 🎯 DevOps Best Practices Implemented

### ✅ **CI/CD Pipeline**
- Automated build triggers via GitHub webhooks
- Multi-stage Docker builds with layer caching
- Automated testing and deployment
- Zero-downtime rolling deployments

### ✅ **Security**
- IAM roles with least privilege access
- No hardcoded credentials in code or configuration
- Parameter Store for sensitive configuration
- Non-root container execution

### ✅ **Infrastructure as Code**
- Parameterized configurations
- Reproducible infrastructure setup
- Version-controlled deployment scripts
- Professional naming conventions

### ✅ **Monitoring & Observability**
- Centralized logging with CloudWatch
- Application health monitoring
- Build and deployment metrics
- Automated alerting on failures

### ✅ **Operational Excellence**
- Automated recovery and scaling
- Comprehensive documentation
- Troubleshooting guides
- Resource cleanup procedures

## 🚀 Success Metrics

After successful implementation, you should have:

- ✅ **Automated CI/CD Pipeline**: Code changes automatically deploy to production
- ✅ **Zero-Downtime Deployments**: Rolling updates without service interruption
- ✅ **Scalable Infrastructure**: ECS Fargate with auto-scaling capabilities
- ✅ **Secure Configuration**: No credentials in code, proper IAM roles
- ✅ **Monitoring Setup**: Logs and metrics for operational visibility
- ✅ **Professional Setup**: Enterprise-grade DevOps practices

