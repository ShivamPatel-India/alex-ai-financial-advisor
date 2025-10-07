# Alex AWS Infrastructure Overview
The Alex platform uses a modern serverless architecture on AWS, combining AI services with cost-effective infrastructure:

![Homepage](../screenshots/AWS%20Infra.png)

---

## System Architecture of Researcher Agent

As shown in the image above, there are mainly 6 Agents. Planner, Tagger, Reporter, Charter and Retirement analyser are deployed using AWS Lambda. On the other hand, Researcher agent which runs every two hours is deployed using AWS AppRunner and the detailed diagram of which is shown below:

```mermaid
graph TB
    %% API Gateway
    APIGW[fa:fa-shield-alt API Gateway<br/>REST API<br/>API Key Auth]
    
    %% Backend Services
    Lambda[fa:fa-bolt Lambda<br/>alex-ingest<br/>Document Processing]
    AppRunner[fa:fa-server App Runner<br/>alex-researcher<br/>AI Agent Service]
    
    %% Scheduler Components
    EventBridge[fa:fa-clock EventBridge<br/>Scheduler<br/>Every 2 Hours]
    SchedulerLambda[fa:fa-bolt Lambda<br/>alex-scheduler<br/>Trigger Research]
    
    %% AI Services
    SageMaker[fa:fa-brain SageMaker<br/>Embedding Model<br/>all-MiniLM-L6-v2]
    Bedrock[fa:fa-robot AWS Bedrock<br/>OSS 120B Model<br/>us-west-2]
    
    %% Data Storage
    S3Vectors[fa:fa-database S3 Vectors<br/>Vector Storage<br/>90% Cost Reduction!]
    ECR[fa:fa-archive ECR<br/>Docker Registry<br/>Researcher Images]
    
    %% Connections
    AppRunner -->|Store Research| APIGW
    AppRunner -->|Generate| Bedrock
    APIGW -->|Invoke| Lambda
    
    EventBridge -->|Every 2hrs| SchedulerLambda
    SchedulerLambda -->|Call /research/auto| AppRunner
    
    Lambda -->|Get Embeddings| SageMaker
    Lambda -->|Store Vectors| S3Vectors
    
    AppRunner -.->|Pull Image| ECR
    
    %% Styling
    classDef aws fill:#FF9900,stroke:#232F3E,stroke-width:2px,color:#fff
    classDef ai fill:#10B981,stroke:#047857,stroke-width:2px,color:#fff
    classDef storage fill:#3B82F6,stroke:#1E40AF,stroke-width:2px,color:#fff
    classDef highlight fill:#90EE90,stroke:#228B22,stroke-width:3px,color:#000
    classDef scheduler fill:#9333EA,stroke:#6B21A8,stroke-width:2px,color:#fff
    
    class APIGW,Lambda,AppRunner,SageMaker,ECR,SchedulerLambda aws
    class Bedrock ai
    class S3Vectors storage
    class S3Vectors highlight
    class EventBridge scheduler
```

## Component Details

### 1. **S3 Vectors** (90% Cost Reduction)
- **Purpose**: Native vector storage in S3
- **Features**: 
  - Sub-second similarity search
  - Automatic optimization
  - No minimum charges
  - Strongly consistent writes
- **Cost**: ~$30/month (vs ~$300/month for OpenSearch)
- **Scale**: Millions of vectors per index

### 2. **API Gateway**
- **Type**: REST API
- **Auth**: API Key authentication
- **Endpoints**: `/ingest` (POST)
- **Purpose**: Secure access to Lambda functions

### 3. **Lambda Functions**
- **alex-ingest**: Processes documents and stores embeddings
  - Runtime: Python 3.12
  - Memory: 512MB
  - Timeout: 30 seconds
- **alex-scheduler**: Triggers automated research
  - Runtime: Python 3.11
  - Memory: 128MB
  - Timeout: 150 seconds

### 4. **App Runner**
- **Service**: alex-researcher
- **Purpose**: Hosts the AI research agent
- **Resources**: 1 vCPU, 2GB RAM
- **Features**: Auto-scaling, HTTPS endpoint

### 5. **SageMaker Serverless**
- **Model**: sentence-transformers/all-MiniLM-L6-v2
- **Purpose**: Generate 384-dimensional embeddings
- **Memory**: 3GB
- **Concurrency**: 10 max

### 6. **EventBridge Scheduler**
- **Rule**: alex-research-schedule
- **Schedule**: Every 2 hours
- **Target**: alex-scheduler Lambda
- **Purpose**: Automated research generation

### 7. **AWS Bedrock**
- **Provider**: AWS Bedrock
- **Model**: OpenAI OSS 120B (open-weight model)
- **Region**: us-west-2 (model only available here)
- **Purpose**: Research generation and analysis
- **Features**: 128K context window, cross-region access

## Security Features

- **API Gateway**: API key authentication
- **IAM Roles**: Least privilege access
- **S3 Vectors**: Always private (no public access)
- **App Runner**: HTTPS by default
- **Secrets**: Environment variables for API keys

## Technology Stack

- **Infrastructure**: Terraform
- **Compute**: Lambda, App Runner
- **AI/ML**: SageMaker, AWS Bedrock
- **Storage**: S3 Vectors
- **API**: API Gateway
- **Languages**: Python 3.12
- **Container**: Docker

## Key Advantages of S3 Vectors

1. **Cost**: 90% reduction vs traditional vector databases
2. **Simplicity**: Just S3 - no complex infrastructure
3. **Scale**: Handles millions of vectors
4. **Performance**: Sub-second queries
5. **Integration**: Native AWS service

---

## Frontend & API

This section describes the deployment of frontend part - a modern React application with real-time agent visualization, portfolio management, and comprehensive financial analysis displays.

### SaaS frontend with:
- **Authentication**: Clerk-based sign-in/sign-up with automatic user creation
- **Portfolio Management**: Add accounts, track positions, edit holdings
- **AI Analysis**: Trigger and monitor multi-agent analysis with real-time progress
- **Interactive Reports**: Markdown reports, dynamic charts, retirement projections
- **Production Infrastructure**: CloudFront CDN, API Gateway, Lambda backend

Here's the complete architecture:

```mermaid
graph TB
    User[User Browser] -->|HTTPS| CF[CloudFront CDN]
    CF -->|Static Files| S3[S3 Static Site]
    CF -->|/api/*| APIG[API Gateway]

    User -->|Auth| Clerk[Clerk Auth]
    APIG -->|JWT| Lambda[API Lambda]

    Lambda -->|Data API| Aurora[(Aurora DB)]
    Lambda -->|Trigger| SQS[SQS Queue]

    SQS -->|Process| Agents[AI Agents]
    Agents -->|Results| Aurora

    style CF fill:#FF9900
    style S3 fill:#569A31
    style Lambda fill:#FF9900
    style Clerk fill:#6C5CE7
```
---