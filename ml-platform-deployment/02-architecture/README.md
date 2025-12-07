# 🏗️ Architecture Overview

## Introduction

This document provides a comprehensive overview of the ML Platform architecture that I designed and implemented at Amazon. The platform leverages AWS native services to provide a secure, scalable, and cost-effective solution for machine learning operations.

---

## 📊 System Context Diagram

```mermaid
graph TB
    subgraph Users["👥 Users"]
        DS[👨‍💻 Data Scientist<br/>Trains and deploys ML models]
        MLE[👩‍💻 ML Engineer<br/>Builds and maintains ML pipelines]
        DevOps[🔧 DevOps Engineer<br/>Manages infrastructure]
    end

    subgraph MLPlatform["🚀 ML Platform"]
        Platform[AWS-based ML Infrastructure<br/>Training & Serving Models]
    end

    subgraph External["🌐 External Systems"]
        GitHub[📦 GitHub<br/>Source code repository]
        ZenML[⚡ ZenML Server<br/>Pipeline orchestration]
        Monitoring[📊 Monitoring Stack<br/>Observability tools]
    end

    DS --> Platform
    MLE --> Platform
    DevOps --> Platform
    
    Platform --> GitHub
    Platform --> ZenML
    Platform --> Monitoring
```

---

## 🎯 Architecture Principles

### 1. **Infrastructure as Code (IaC)**
- All infrastructure defined in CloudFormation templates
- Version-controlled, auditable, and reproducible
- Enables GitOps workflows

### 2. **Least Privilege Access**
- Fine-grained IAM policies
- Role-based access control
- Service-specific execution roles

### 3. **Separation of Concerns**
- Distinct roles for different operations
- Isolated resources per environment
- Clear service boundaries

### 4. **Scalability by Design**
- Serverless where possible (Lambda, SageMaker)
- Auto-scaling capabilities
- Managed services for reduced operational overhead

### 5. **Security First**
- Encryption at rest and in transit
- Private resources (S3, ECR)
- IAM role assumption for cross-service access

---

## 🔧 Core Components

### Component Overview

```mermaid
graph TB
    subgraph "Infrastructure Layer"
        CF[CloudFormation<br/>Infrastructure as Code]
    end

    subgraph "Storage Layer"
        S3[S3 Bucket<br/>Artifact Store]
        ECR[ECR Repository<br/>Container Registry]
    end

    subgraph "Compute Layer"
        SM[SageMaker<br/>Training & Inference]
        CB[CodeBuild<br/>Image Builder]
        Lambda[Lambda<br/>API Integration]
    end

    subgraph "Security Layer"
        IAMUser[IAM User<br/>Programmatic Access]
        IAMRole[IAM Role<br/>Stack Access]
        SMRole[SageMaker Role<br/>Execution]
        CBRole[CodeBuild Role<br/>Build Execution]
    end

    subgraph "Orchestration Layer"
        ZenML[ZenML Server<br/>Pipeline Management]
    end

    CF --> S3
    CF --> ECR
    CF --> SM
    CF --> CB
    CF --> Lambda
    CF --> IAMUser
    CF --> IAMRole
    CF --> SMRole
    CF --> CBRole

    IAMUser --> IAMRole
    IAMRole --> S3
    IAMRole --> ECR
    IAMRole --> SM
    IAMRole --> CB

    SMRole --> S3
    SMRole --> SM

    CBRole --> S3
    CBRole --> ECR

    Lambda --> ZenML
```

---

## 📁 Resource Details

### 1. S3 Bucket (Artifact Store)

**Purpose**: Central storage for all ML artifacts

```yaml
Resource: AWS::S3::Bucket
Name: ${ResourceName}-${AWS::AccountId}
Access: Private
```

**Stored Data**:
- Training datasets
- Model artifacts (.pkl, .h5, .pt)
- Pipeline outputs
- SageMaker output data
- CodeBuild source archives

**Key Features**:
| Feature | Configuration | Benefit |
|---------|--------------|---------|
| Private Access | `AccessControl: Private` | Security compliance |
| Unique Naming | Account ID suffix | Cross-account uniqueness |
| Tagging | Project tags | Cost allocation |

---

### 2. ECR Repository (Container Registry)

**Purpose**: Store Docker images for ML pipelines

```yaml
Resource: AWS::ECR::Repository
Name: ${ResourceName}
```

**Image Types**:
- Training container images
- Inference container images
- Pipeline step images
- Custom algorithm containers

**Key Features**:
| Feature | Configuration | Benefit |
|---------|--------------|---------|
| Private Registry | Default | Security |
| Image Scanning | Available | Vulnerability detection |
| Lifecycle Policies | Configurable | Cost optimization |

---

### 3. SageMaker Integration

**Purpose**: Managed ML training and inference

**Components**:

```mermaid
graph LR
    subgraph "SageMaker Services"
        Pipelines[SageMaker Pipelines<br/>Orchestration]
        Training[Training Jobs<br/>Model Training]
        Processing[Processing Jobs<br/>Data Processing]
        Endpoints[Endpoints<br/>Model Serving]
    end

    subgraph "Supporting Services"
        SM_S3[S3<br/>Data Storage]
        SM_ECR[ECR<br/>Container Images]
        SM_IAM[IAM Role<br/>Execution]
    end

    SM_S3 --> Training
    SM_S3 --> Processing
    SM_ECR --> Training
    SM_ECR --> Endpoints
    SM_IAM --> Pipelines
    SM_IAM --> Training
    SM_IAM --> Endpoints

    Pipelines --> Training
    Pipelines --> Processing
    Training --> Endpoints
```

**IAM Permissions**:
- `CreatePipeline`, `StartPipelineExecution`
- `DescribePipeline`, `DescribePipelineExecution`
- `CreateTrainingJob`, `DescribeTrainingJob`
- Full SageMaker access via managed policy

---

### 4. CodeBuild (Image Builder)

**Purpose**: Build Docker images for ML pipelines

```yaml
Resource: AWS::CodeBuild::Project
Condition: RegisterCodeBuild (optional)
Environment: LINUX_CONTAINER / BUILD_GENERAL1_SMALL
Image: bentolor/docker-dind-awscli
Timeout: 20 minutes
```

**Build Process**:

```mermaid
sequenceDiagram
    participant ZenML
    participant S3
    participant CodeBuild
    participant ECR

    ZenML->>S3: Upload build context
    ZenML->>CodeBuild: Start build
    CodeBuild->>S3: Download source
    CodeBuild->>CodeBuild: Build Docker image
    CodeBuild->>ECR: Push image
    CodeBuild->>ZenML: Build complete
```

---

### 5. IAM Architecture

**Purpose**: Secure access control

```mermaid
graph TB
    subgraph "External Access"
        User[IAM User<br/>Programmatic Access]
        AccessKey[Access Key<br/>Credentials]
    end

    subgraph "Role Assumption"
        StackRole[Stack Access Role<br/>Primary Operations]
    end

    subgraph "Service Roles"
        SMRole[SageMaker Role<br/>ML Execution]
        CBRole[CodeBuild Role<br/>Build Execution]
    end

    subgraph "Resource Access"
        S3Access[S3 Permissions]
        ECRAccess[ECR Permissions]
        SMAccess[SageMaker Permissions]
    end

    User --> AccessKey
    AccessKey --> StackRole
    StackRole --> S3Access
    StackRole --> ECRAccess
    StackRole --> SMAccess
    StackRole -.->|PassRole| SMRole
    StackRole -.->|PassRole| CBRole
    
    SMRole --> S3Access
    CBRole --> S3Access
    CBRole --> ECRAccess
```

**Role Details**:

| Role | Purpose | Trust Policy | Key Permissions |
|------|---------|--------------|-----------------|
| `${ResourceName}` | Stack operations | IAM User | S3, ECR, SageMaker, CodeBuild |
| `${ResourceName}-sagemaker` | SageMaker execution | sagemaker.amazonaws.com | S3, SageMakerFullAccess |
| `${ResourceName}-codebuild` | CodeBuild execution | codebuild.amazonaws.com | S3, ECR, CloudWatch Logs |

---

### 6. Lambda Function (ZenML Integration)

**Purpose**: Register stack with ZenML server

```yaml
Resource: AWS::Serverless::Function
Runtime: python3.8
Memory: 512MB
Timeout: 60s
```

**Workflow**:

```mermaid
sequenceDiagram
    participant CF as CloudFormation
    participant Lambda
    participant ZenML as ZenML Server

    CF->>Lambda: CustomResource Create
    Lambda->>Lambda: Build stack payload
    Lambda->>ZenML: POST /api/v1/workspaces/default/stacks
    ZenML-->>Lambda: 200 OK
    Lambda-->>CF: SUCCESS response
```

**Payload Structure**:
```json
{
  "name": "${StackName}",
  "description": "Deployed by CloudFormation...",
  "labels": {
    "zenml:provider": "aws",
    "zenml:deployment": "cloud-formation"
  },
  "service_connectors": [...],
  "components": {
    "artifact_store": [...],
    "container_registry": [...],
    "orchestrator": [...],
    "step_operator": [...],
    "image_builder": [...]
  }
}
```

---

## 🔐 Security Architecture

### Authentication Flow

```mermaid
sequenceDiagram
    participant Client
    participant IAMUser
    participant STS
    participant StackRole
    participant AWSService

    Client->>IAMUser: Access Key + Secret
    IAMUser->>STS: AssumeRole request
    STS->>STS: Validate permissions
    STS-->>Client: Temporary credentials
    Client->>AWSService: API call with temp credentials
    AWSService->>StackRole: Validate permissions
    AWSService-->>Client: Response
```

### Permission Boundaries

```mermaid
graph TB
    subgraph "IAM User Permissions"
        AssumeRole[sts:AssumeRole<br/>Only allowed action]
    end

    subgraph "Stack Role Permissions"
        S3Perms[S3: List, Get, Put, Delete]
        ECRPerms[ECR: Push, Pull, Describe]
        SMPerms[SageMaker: Pipeline, Training]
        CBPerms[CodeBuild: Start, BatchGet]
        PassRole[iam:PassRole to SageMaker]
    end

    subgraph "Resource Scope"
        SpecificS3[Only stack S3 bucket]
        SpecificECR[Only stack ECR repo]
        SpecificSM[SageMaker execution role]
        SpecificCB[Only stack CodeBuild project]
    end

    AssumeRole --> S3Perms
    AssumeRole --> ECRPerms
    AssumeRole --> SMPerms
    AssumeRole --> CBPerms

    S3Perms --> SpecificS3
    ECRPerms --> SpecificECR
    SMPerms --> SpecificSM
    CBPerms --> SpecificCB
    PassRole --> SpecificSM
```

---

## 📈 Scalability Considerations

### Horizontal Scaling

| Component | Scaling Strategy | Limits |
|-----------|-----------------|--------|
| S3 | Automatic | Unlimited objects |
| ECR | Automatic | 10,000 repositories/region |
| SageMaker Training | Instance count | 20 instances default |
| SageMaker Endpoints | Auto-scaling policies | Based on metrics |
| CodeBuild | Concurrent builds | 60 default |

### Vertical Scaling

| Component | Options | Recommendation |
|-----------|---------|----------------|
| SageMaker Training | ml.m5.large to ml.p4d.24xlarge | Start small, scale as needed |
| SageMaker Endpoints | ml.t2.medium to ml.g5.48xlarge | Based on inference requirements |
| CodeBuild | SMALL, MEDIUM, LARGE | SMALL for most builds |

---

## 🔗 Related Documentation

- **[High-Level Design (HLD)](./HLD.md)** - System architecture and component interactions
- **[Low-Level Design (LLD)](./LLD.md)** - Detailed component specifications
- **[Deployment Workflow](../03-deployment-workflow/README.md)** - CI/CD pipeline details
