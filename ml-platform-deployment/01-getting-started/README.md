# 🚀 Getting Started with the ML Platform

## Overview

This guide walks you through the prerequisites, setup, and initial deployment of the ML Platform infrastructure. By the end of this guide, you'll have a fully functional ML platform with ECR, S3, and SageMaker integration.

---

## 📋 Prerequisites

### AWS Account Requirements

| Requirement | Description | Verification Command |
|-------------|-------------|---------------------|
| AWS Account | Active AWS account with billing enabled | `aws sts get-caller-identity` |
| IAM Permissions | Administrator or CloudFormation full access | Check IAM console |
| Service Quotas | Sufficient quotas for EC2, SageMaker, S3 | Check Service Quotas console |
| Region Access | Target region enabled (default: eu-central-1) | Check region selector |

### Local Development Environment

```bash
# Required tools and versions
aws-cli >= 2.0.0
python >= 3.8
docker >= 20.10
zenml >= 0.70.0
```

### Installation Commands

```bash
# Install AWS CLI v2 (Windows)
msiexec.exe /i https://awscli.amazonaws.com/AWSCLIV2.msi

# Install AWS CLI v2 (macOS)
curl "https://awscli.amazonaws.com/AWSCLIV2.pkg" -o "AWSCLIV2.pkg"
sudo installer -pkg AWSCLIV2.pkg -target /

# Install AWS CLI v2 (Linux)
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# Install ZenML
pip install zenml[server]

# Verify installations
aws --version
zenml version
docker --version
```

---

## 🔐 AWS Configuration

### Step 1: Configure AWS Credentials

```bash
# Configure default profile
aws configure

# Or configure named profile
aws configure --profile ml-platform

# Verify configuration
aws sts get-caller-identity
```

### Step 2: Verify Service Access

```bash
# Check S3 access
aws s3 ls

# Check ECR access
aws ecr describe-repositories

# Check SageMaker access
aws sagemaker list-training-jobs --max-results 1

# Check CloudFormation access
aws cloudformation list-stacks --stack-status-filter CREATE_COMPLETE
```

---

## 📦 CloudFormation Template Parameters

### Required Parameters

| Parameter | Type | Description | Example |
|-----------|------|-------------|---------|
| `ResourceName` | String | Unique name for all resources | `ml-platform-prod` |
| `ZenMLServerURL` | String | ZenML server endpoint | `https://zenml.example.com` |
| `ZenMLServerAPIToken` | String | API token for ZenML authentication | `zen_xxx...` |

### Optional Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `TagName` | String | `project` | Tag key for all resources |
| `TagValue` | String | `zenml` | Tag value for all resources |
| `CodeBuild` | Boolean | `false` | Enable CodeBuild for image building |

---

## 🚀 Deployment Methods

### Method 1: AWS Console (Recommended for First-Time Setup)

1. **Navigate to CloudFormation Console**
   ```
   https://console.aws.amazon.com/cloudformation/home?region=eu-central-1#/stacks/create/review
   ```

2. **Upload Template**
   - Select "Upload a template file"
   - Choose `aws-ecr-s3-sagemaker.yaml`

3. **Configure Parameters**
   - Stack name: `zenml-ml-platform`
   - ResourceName: `ml-platform-prod`
   - Fill in ZenML server details

4. **Configure Stack Options**
   - Enable "I acknowledge that AWS CloudFormation might create IAM resources with custom names"

5. **Create Stack**
   - Review and click "Create stack"

### Method 2: AWS CLI

```bash
# Basic deployment
aws cloudformation create-stack \
  --stack-name zenml-ml-platform \
  --template-body file://infra/aws/aws-ecr-s3-sagemaker.yaml \
  --parameters \
    ParameterKey=ResourceName,ParameterValue=ml-platform-prod \
  --capabilities CAPABILITY_NAMED_IAM

# Full deployment with ZenML integration
aws cloudformation create-stack \
  --stack-name zenml-ml-platform \
  --template-body file://infra/aws/aws-ecr-s3-sagemaker.yaml \
  --parameters \
    ParameterKey=ResourceName,ParameterValue=ml-platform-prod \
    ParameterKey=ZenMLServerURL,ParameterValue=https://zenml.example.com \
    ParameterKey=ZenMLServerAPIToken,ParameterValue=your-api-token \
    ParameterKey=CodeBuild,ParameterValue=true \
    ParameterKey=TagName,ParameterValue=environment \
    ParameterKey=TagValue,ParameterValue=production \
  --capabilities CAPABILITY_NAMED_IAM \
  --tags Key=Owner,Value=mlops-team

# Monitor stack creation
aws cloudformation describe-stacks \
  --stack-name zenml-ml-platform \
  --query 'Stacks[0].StackStatus'

# Wait for completion
aws cloudformation wait stack-create-complete \
  --stack-name zenml-ml-platform
```

### Method 3: Infrastructure as Code (Terraform Alternative)

```hcl
# main.tf - Terraform equivalent
module "ml_platform" {
  source = "./modules/ml-platform"
  
  resource_name        = "ml-platform-prod"
  zenml_server_url     = var.zenml_server_url
  zenml_api_token      = var.zenml_api_token
  enable_codebuild     = true
  
  tags = {
    Environment = "production"
    Project     = "ml-platform"
    Owner       = "mlops-team"
  }
}
```

---

## ✅ Post-Deployment Verification

### Step 1: Retrieve Stack Outputs

```bash
# Get all outputs
aws cloudformation describe-stacks \
  --stack-name zenml-ml-platform \
  --query 'Stacks[0].Outputs'

# Get specific outputs
aws cloudformation describe-stacks \
  --stack-name zenml-ml-platform \
  --query 'Stacks[0].Outputs[?OutputKey==`AWSAccessKeyID`].OutputValue' \
  --output text
```

### Step 2: Verify Resources Created

```bash
# Verify S3 bucket
aws s3 ls | grep ml-platform

# Verify ECR repository
aws ecr describe-repositories --repository-names ml-platform-prod

# Verify IAM role
aws iam get-role --role-name ml-platform-prod

# Verify SageMaker role
aws iam get-role --role-name ml-platform-prod-sagemaker

# Verify CodeBuild project (if enabled)
aws codebuild batch-get-projects --names ml-platform-prod
```

### Step 3: Test ZenML Integration

```bash
# Connect ZenML client
zenml connect --url https://zenml.example.com --api-key <your-key>

# List stacks
zenml stack list

# Set active stack
zenml stack set zenml-ml-platform

# Verify stack components
zenml stack describe
```

---

## 🔧 Initial Configuration

### Configure ZenML Stack

```python
# Python script to verify stack configuration
from zenml.client import Client

client = Client()

# Get active stack
stack = client.active_stack

print(f"Stack Name: {stack.name}")
print(f"Artifact Store: {stack.artifact_store.flavor}")
print(f"Container Registry: {stack.container_registry.flavor}")
print(f"Orchestrator: {stack.orchestrator.flavor}")
```

### Test Pipeline Execution

```python
# test_pipeline.py
from zenml import pipeline, step

@step
def hello_world() -> str:
    """Simple test step."""
    return "Hello from ML Platform!"

@step
def print_message(message: str) -> None:
    """Print the message."""
    print(message)

@pipeline
def test_pipeline():
    """Test pipeline to verify stack."""
    message = hello_world()
    print_message(message)

if __name__ == "__main__":
    test_pipeline()
```

```bash
# Run test pipeline
python test_pipeline.py
```

---

## 📊 Resource Inventory

After successful deployment, you'll have:

```mermaid
graph LR
    subgraph "Created Resources"
        S3[S3 Bucket<br/>ml-platform-prod-{account-id}]
        ECR[ECR Repository<br/>ml-platform-prod]
        IAM1[IAM User<br/>ml-platform-prod]
        IAM2[IAM Role<br/>ml-platform-prod]
        IAM3[SageMaker Role<br/>ml-platform-prod-sagemaker]
        CB[CodeBuild Project<br/>ml-platform-prod]
        Lambda[Lambda Function<br/>ZenML API Integration]
    end

    IAM1 --> IAM2
    IAM2 --> S3
    IAM2 --> ECR
    IAM2 --> CB
    IAM3 --> S3
    Lambda --> IAM2
```

---

## ⚠️ Common Setup Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Stack creation failed | Insufficient IAM permissions | Ensure CAPABILITY_NAMED_IAM |
| Resource name conflict | Name already exists | Use unique ResourceName |
| ZenML registration failed | Invalid URL/token | Verify ZenML server connectivity |
| ECR push failed | Authentication expired | Run `aws ecr get-login-password` |

---

## 🔗 Next Steps

1. **[Architecture Overview](../02-architecture/README.md)** - Understand the system design
2. **[Deployment Workflow](../03-deployment-workflow/README.md)** - Learn the CI/CD pipeline
3. **[Best Practices](../07-best-practices/README.md)** - Production recommendations

---

## 📚 Additional Resources

- [AWS CloudFormation Documentation](https://docs.aws.amazon.com/cloudformation/)
- [AWS SageMaker Developer Guide](https://docs.aws.amazon.com/sagemaker/)
- [ZenML Documentation](https://docs.zenml.io/)
- [AWS Well-Architected ML Lens](https://docs.aws.amazon.com/wellarchitected/latest/machine-learning-lens/)
