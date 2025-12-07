# 📋 Low-Level Design (LLD)

## Document Information
| Attribute | Value |
|-----------|-------|
| **Document Title** | ML Platform Low-Level Design |
| **Author** | ML Platform Engineer (Amazon) |
| **Last Updated** | December 2024 |

---

## 1. CloudFormation Template Structure

```mermaid
graph TB
    subgraph "Template Sections"
        Header[Template Header]
        Params[Parameters]
        Conditions[Conditions]
        Resources[Resources]
        Outputs[Outputs]
    end

    subgraph "Resources Created"
        S3[S3 Bucket]
        ECR[ECR Repository]
        IAMUser[IAM User]
        StackRole[Stack Access Role]
        SMRole[SageMaker Role]
        CB[CodeBuild Project]
        CBRole[CodeBuild Role]
        Lambda[Lambda Function]
    end

    Header --> Params --> Conditions --> Resources --> Outputs
    Resources --> S3
    Resources --> ECR
    Resources --> IAMUser
    Resources --> StackRole
    Resources --> SMRole
    Resources --> CB
    Resources --> CBRole
    Resources --> Lambda
```

---

## 2. Parameter Specifications

| Parameter | Type | Constraints | Purpose |
|-----------|------|-------------|---------|
| `ResourceName` | String | 8-32 chars, `[a-z0-9-]+` | Base name for all resources |
| `ZenMLServerURL` | String | Valid URL pattern | ZenML server endpoint |
| `ZenMLServerAPIToken` | String | N/A | Authentication token |
| `TagName` | String | Default: `project` | Tag key |
| `TagValue` | String | Default: `zenml` | Tag value |
| `CodeBuild` | String | `true`/`false` | Enable image builder |

---

## 3. Condition Logic

```mermaid
flowchart TD
    CheckURL{ZenMLServerURL provided?}
    CheckToken{ZenMLServerAPIToken provided?}
    CheckCB{CodeBuild = true?}
    
    RegStack[RegisterZenMLStack = TRUE]
    NoRegStack[RegisterZenMLStack = FALSE]
    RegCB[RegisterCodeBuild = TRUE]
    NoRegCB[RegisterCodeBuild = FALSE]
    
    CheckURL -->|Yes| CheckToken
    CheckURL -->|No| NoRegStack
    CheckToken -->|Yes| RegStack
    CheckToken -->|No| NoRegStack
    
    CheckCB -->|Yes| RegCB
    CheckCB -->|No| NoRegCB
```

---

## 4. S3 Bucket Design

**Naming**: `${ResourceName}-${AWS::AccountId}`

**Structure**:
```
s3://ml-platform-prod-123456789012/
├── artifacts/          # Pipeline outputs
├── data/
│   ├── training/      # Training datasets
│   └── validation/    # Validation data
├── models/            # Model artifacts
├── sagemaker/         # SageMaker output
└── codebuild/         # Build sources
```

---

## 5. ECR Repository Design

**Naming**: `${ResourceName}`

**Image URI Format**: `{account_id}.dkr.ecr.{region}.amazonaws.com/{repository}:{tag}`

**Tagging Strategy**:
| Pattern | Purpose |
|---------|---------|
| `latest` | Most recent |
| `v{semver}` | Versioned releases |
| `{git-sha}` | Commit reference |

---

## 6. IAM Architecture

```mermaid
graph TB
    subgraph "Authentication"
        IAMUser[IAM User] --> AccessKey[Access Key]
        AccessKey --> STS[STS AssumeRole]
    end

    subgraph "Roles"
        STS --> StackRole[Stack Access Role]
        SM[SageMaker Service] --> SMRole[SageMaker Role]
        CB[CodeBuild Service] --> CBRole[CodeBuild Role]
    end

    subgraph "Permissions"
        StackRole --> S3Perms[S3: CRUD]
        StackRole --> ECRPerms[ECR: Push/Pull]
        StackRole --> SMPerms[SageMaker: Pipeline/Training]
        StackRole --> PassRole[iam:PassRole]
        
        SMRole --> S3Access[S3 Access]
        SMRole --> FullSM[SageMakerFullAccess]
        
        CBRole --> S3Read[S3 Read]
        CBRole --> ECRPush[ECR Push]
        CBRole --> Logs[CloudWatch Logs]
    end
```

---

## 7. CodeBuild Configuration

```yaml
Environment:
  Type: LINUX_CONTAINER
  ComputeType: BUILD_GENERAL1_SMALL
  Image: bentolor/docker-dind-awscli
  PrivilegedMode: false

Source:
  Type: S3
  Location: ${S3Bucket}/codebuild

TimeoutInMinutes: 20

LogsConfig:
  CloudWatchLogs:
    Status: ENABLED
```

---

## 8. Lambda Function Design

**Purpose**: Register stack with ZenML server via Custom Resource

```mermaid
sequenceDiagram
    CF->>Lambda: CustomResource Create
    Lambda->>Lambda: Build payload
    Lambda->>ZenML: POST /api/v1/workspaces/default/stacks
    ZenML-->>Lambda: 200 OK
    Lambda-->>CF: SUCCESS
```

**Stack Registration Payload**:
- Service Connector (AWS type, IAM role auth)
- Artifact Store (S3 flavor)
- Container Registry (AWS ECR flavor)
- Orchestrator (SageMaker flavor)
- Step Operator (SageMaker flavor)
- Image Builder (AWS or local flavor)

---

## 9. SageMaker Integration

```mermaid
graph LR
    subgraph "Pipeline Steps"
        Preprocess --> Train --> Evaluate --> Register --> Deploy
    end

    subgraph "Artifacts"
        S3Data[S3 Data] --> Preprocess
        ECRImage[ECR Image] --> Train
        Train --> S3Model[S3 Model]
        S3Model --> Deploy
    end
```

---

## 10. Stack Outputs

| Output | Value | Purpose |
|--------|-------|---------|
| `AWSRegion` | Region | Deployment region |
| `AWSAccessKeyID` | Access Key | Programmatic access |
| `AWSSecretAccessKey` | Secret Key | Authentication |
| `IAMRoleARN` | Role ARN | Stack operations |
| `SageMakerIAMRoleARN` | SM Role ARN | Training execution |

---

## 🔗 Related Documents
- [HLD](./HLD.md) - High-level architecture
- [Deployment Workflow](../03-deployment-workflow/README.md)
