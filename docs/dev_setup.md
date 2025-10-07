# Alex Developer Setup Guide

This guide provides complete step-by-step instructions for deploying the Alex AI Financial Advisor platform on AWS.

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Initial Setup](#initial-setup)
3. [Part 1: AWS Permissions](#part-1-aws-permissions)
4. [Part 2: SageMaker Embeddings](#part-2-sagemaker-embeddings)
5. [Part 3: Ingestion Pipeline](#part-3-ingestion-pipeline)
6. [Part 4: Researcher Agent](#part-4-researcher-agent)
7. [Part 5: Database Setup](#part-5-database-setup)
8. [Part 6: Agent Orchestra](#part-6-agent-orchestra)
9. [Part 7: Frontend & API](#part-7-frontend--api)
10. [Part 8: Cloudwatch Monitoring and LangFuse Observability](#part-8-cloudwatch-monitoring-and-langFuse-observability)
11. [Testing](#testing)
12. [Troubleshooting](#troubleshooting)
13. [Cost Management](#cost-management)
14. [Cleanup](#cleanup)

## Prerequisites

### Required Tools

- **AWS Account** with root access
- **AWS CLI** (version 2.x+)
  ```bash
  aws --version  # Should show 2.x or higher
  ```
- **Node.js** (20+) and npm
  ```bash
  node --version  # Should show v20.x or higher
  npm --version
  ```
- **Python** (3.12+) with `uv` package manager
  ```bash
  python --version  # Should show 3.12 or higher
  pip install uv
  ```
- **Terraform** (1.5+)
  ```bash
  terraform --version  # Should show 1.5 or higher
  ```
- **Docker Desktop** installed and running
  ```bash
  docker --version
  docker ps  # Should show no errors
  ```
- **Git** for version control

### Required Accounts

- **AWS Account**: Active account with billing enabled
- **OpenAI Account**: API key for agent tracing ([platform.openai.com](https://platform.openai.com))
- **Clerk Account**: Free tier for authentication ([clerk.com](https://clerk.com))
- **Polygon.io Account**: Free tier for stock prices ([polygon.io](https://polygon.io))
- **LangFuse Account**: Free tier for observability ([cloud.langfuse.com](https://cloud.langfuse.com)) - optional

### Skills Needed

- Basic AWS knowledge
- Command line proficiency
- Understanding of environment variables
- Familiarity with Infrastructure as Code concepts

## Initial Setup

### Clone the Repository

```bash
git clone <your-repository-url>
cd alex
```

### Create Environment File

```bash
# Copy the example environment file
cp .env.example .env
```

You'll populate this `.env` file as you progress through each part of the setup.

### Verify AWS CLI Configuration

```bash
# Get your AWS account ID
aws sts get-caller-identity

# This should show:
# - UserId
# - Account (your AWS account ID)
# - Arn
```

Save your AWS account ID - you'll need it throughout the setup.

## Part 1: AWS Permissions

### 1.1 Create IAM User and Group

1. Sign in to [AWS Console](https://aws.amazon.com/console/) as **root user**
2. Navigate to **IAM** service
3. Create IAM user named `aiengineer`:
   - Go to **Users** → **Create user**
   - Username: `aiengineer`
   - Enable **Provide user access to the AWS Management Console**
   - Choose **I want to create an IAM user**
   - Set custom password or auto-generate
   - Uncheck "User must create a new password at next sign-in" if desired
   - Click **Next**

4. Create and attach required policies:
   - Create a custom policy for S3 Vectors:
     - Go to **Policies** → **Create policy**
     - Use JSON tab and paste:
       ```json
       {
           "Version": "2012-10-17",
           "Statement": [
               {
                   "Effect": "Allow",
                   "Action": ["s3vectors:*"],
                   "Resource": "*"
               }
           ]
       }
       ```
     - Name it `AlexS3VectorsAccess`
   
   - Create user group `AlexAccess`:
     - Go to **User groups** → **Create group**
     - Group name: `AlexAccess`
     - Attach these AWS managed policies:
       - `AmazonSageMakerFullAccess`
       - `AmazonBedrockFullAccess`
       - `CloudWatchEventsFullAccess`
       - `AmazonS3FullAccess`
       - `AWSLambda_FullAccess`
       - `AmazonAPIGatewayAdministrator`
       - `CloudWatchFullAccess`
       - `AmazonRDSDataFullAccess`
       - `AmazonSQSFullAccess`
       - `AmazonEventBridgeFullAccess`
       - `SecretsManagerReadWrite`
     - Also attach the custom policy `AlexS3VectorsAccess`
   
   - Add `aiengineer` user to `AlexAccess` group

5. Create custom RDS policy (for Part 5):
   - Go to **Policies** → **Create policy**
   - Use JSON tab and paste:
     ```json
     {
         "Version": "2012-10-17",
         "Statement": [
             {
                 "Effect": "Allow",
                 "Action": [
                     "rds:CreateDBCluster",
                     "rds:CreateDBInstance",
                     "rds:DeleteDBCluster",
                     "rds:DeleteDBInstance",
                     "rds:DescribeDBClusters",
                     "rds:DescribeDBInstances",
                     "rds:ModifyDBCluster",
                     "rds:ModifyDBInstance",
                     "rds:AddTagsToResource",
                     "rds-data:ExecuteStatement",
                     "rds-data:BatchExecuteStatement",
                     "ec2:DescribeVpcs",
                     "ec2:DescribeSubnets",
                     "ec2:DescribeSecurityGroups",
                     "ec2:CreateSecurityGroup",
                     "ec2:AuthorizeSecurityGroupIngress",
                     "secretsmanager:CreateSecret",
                     "secretsmanager:GetSecretValue",
                     "kms:CreateGrant",
                     "kms:Decrypt"
                 ],
                 "Resource": "*"
             }
         ]
     }
     ```
   - Name it `AlexRDSCustomPolicy`
   - Attach to `AlexAccess` group

### 1.2 Configure AWS CLI with IAM User

```bash
# Configure AWS CLI with your IAM user credentials
aws configure

# Enter when prompted:
# AWS Access Key ID: [from IAM user security credentials]
# AWS Secret Access Key: [from IAM user security credentials]
# Default region name: us-east-1  # or your preferred region
# Default output format: json

# Verify configuration
aws sts get-caller-identity
```

### 1.3 Update Environment File

Add to your `.env` file:

```bash
# Part 1 - AWS Configuration
AWS_ACCOUNT_ID=123456789012  # Your AWS account ID
DEFAULT_AWS_REGION=us-east-1  # Your preferred region
```

## Part 2: SageMaker Embeddings

Deploy a serverless SageMaker endpoint for generating text embeddings using the `all-MiniLM-L6-v2` model.

### 2.1 Request Bedrock Model Access

1. Sign in to AWS Console
2. Navigate to **Amazon Bedrock**
3. Switch to **us-west-2** region (required for OpenAI OSS models)
4. Click **Model access** → **Manage model access**
5. Request access to:
   - OpenAI GPT OSS 120B (`gpt-oss-120b`)
   - Alternative: Amazon Nova Pro in your region

### 2.2 Configure and Deploy

```bash
# Navigate to SageMaker terraform directory
cd terraform/2_sagemaker

# Copy and edit variables
cp terraform.tfvars.example terraform.tfvars
```

Edit `terraform.tfvars`:
```hcl
aws_region = "us-east-1"  # Your region
```

Deploy:
```bash
# Initialize Terraform
terraform init

# Review plan
terraform plan

# Deploy
terraform apply  # Type 'yes' when prompted
```

### 2.3 Test the Endpoint

```bash
# Navigate to backend directory
cd ../../backend

# Test the endpoint
aws sagemaker-runtime invoke-endpoint \
  --endpoint-name alex-embedding-endpoint \
  --content-type application/json \
  --body fileb://vectorize_me.json \
  --output json /dev/stdout
```

Expected output: JSON array with 384 float values (embeddings).

### 2.4 Update Environment File

Add to `.env`:
```bash
# Part 2 - SageMaker
SAGEMAKER_ENDPOINT=alex-embedding-endpoint
```

## Part 3: Ingestion Pipeline

Deploy S3 Vectors storage and Lambda function for document ingestion.

### 3.1 Create S3 Vector Bucket

1. Go to [S3 Console](https://console.aws.amazon.com/s3/)
2. Look for **"Vector buckets"** (separate from regular S3)
3. Click **"Create vector bucket"**
4. Configuration:
   - Bucket name: `alex-vectors-{your-account-id}`
   - Encryption: Default (SSE-S3)
5. After creating bucket, create index:
   - Index name: `financial-research`
   - Dimension: `384`
   - Distance metric: `Cosine`

### 3.2 Package Lambda Function

```bash
# Navigate to ingest directory
cd backend/ingest

# Create deployment package
uv run package.py
```

This creates `lambda_function.zip` (~15 MB).

### 3.3 Configure and Deploy

```bash
# Navigate to ingestion terraform directory
cd ../../terraform/3_ingestion

# Copy and edit variables
cp terraform.tfvars.example terraform.tfvars
```

Edit `terraform.tfvars`:
```hcl
aws_region = "us-east-1"
sagemaker_endpoint_name = "alex-embedding-endpoint"
```

Deploy:
```bash
terraform init
terraform apply  # Type 'yes' when prompted
```

### 3.4 Get and Save API Key

```bash
# Get the API key ID from terraform output
terraform output api_key_id

# Retrieve the actual key value
aws apigateway get-api-key \
  --api-key [your-api-key-id] \
  --include-value \
  --query 'value' \
  --output text
```

### 3.5 Update Environment File

Add to `.env`:
```bash
# Part 3 - Ingestion
VECTOR_BUCKET=alex-vectors-YOUR_ACCOUNT_ID
ALEX_API_ENDPOINT=https://xxxxxxxxxx.execute-api.us-east-1.amazonaws.com/prod/ingest
ALEX_API_KEY=your-api-key-here
```

### 3.6 Test Ingestion

```bash
# Navigate to ingest directory
cd ../../backend/ingest

# Test direct S3 Vectors ingestion
uv run test_ingest_s3vectors.py

# Test search functionality
uv run test_search_s3vectors.py
```

## Part 4: Researcher Agent

Deploy the AI researcher service on AWS App Runner with optional automated scheduling.

### 4.1 Update Researcher Configuration

Before deploying, update the model configuration:

```bash
# Edit backend/researcher/server.py
# Update these variables to match your setup:
REGION = "us-east-1"  # Your AWS region
MODEL = "bedrock/us.amazon.nova-lite-v1:0"  # Or your chosen model
```

Available model options:
- `bedrock/us.amazon.nova-lite-v1:0` (US regions)
- `bedrock/eu.amazon.nova-lite-v1:0` (EU regions)
- `bedrock/openai.gpt-oss-120b-1:0` (OpenAI OSS, us-west-2 only)
- `bedrock/converse/us.anthropic.claude-sonnet-4-20250514-v1:0` (Claude)

### 4.2 Get OpenAI API Key

1. Go to [platform.openai.com](https://platform.openai.com)
2. Sign in or create account
3. Navigate to **API Keys**
4. Click **Create new secret key**
5. Copy the key (starts with `sk-`)

Add to `.env`:
```bash
# Part 4 - OpenAI (required for agent tracing)
OPENAI_API_KEY=sk-...
```

### 4.3 Deploy Infrastructure (ECR First)

```bash
# Navigate to researcher terraform directory
cd terraform/4_researcher

# Copy and edit variables
cp terraform.tfvars.example terraform.tfvars
```

Edit `terraform.tfvars`:
```hcl
aws_region = "us-east-1"
openai_api_key = "sk-..."  # Your OpenAI key
alex_api_endpoint = "https://..."  # From Part 3
alex_api_key = "..."  # From Part 3
scheduler_enabled = false  # Keep false initially
```

Deploy ECR and IAM:
```bash
terraform init
terraform apply -target=aws_ecr_repository.researcher -target=aws_iam_role.app_runner_role
```

### 4.4 Build and Push Docker Image

```bash
# Navigate to researcher directory
cd ../../backend/researcher

# Build and deploy
uv run deploy.py
```

This script:
- Builds Docker image for linux/amd64
- Pushes to ECR
- Takes 3-5 minutes

### 4.5 Deploy App Runner Service

```bash
# Navigate back to terraform directory
cd ../../terraform/4_researcher

# Deploy complete infrastructure
terraform apply  # Type 'yes' when prompted
```

App Runner service creation takes 3-5 minutes.

### 4.6 Test Researcher

```bash
# Navigate to researcher directory
cd ../../backend/researcher

# Test research generation
uv run test_research.py

# Test with specific topic
uv run test_research.py "Tesla competitive advantages"
```

### 4.7 Verify Data Storage

```bash
# Navigate to ingest directory
cd ../ingest

# Check stored research
uv run test_search_s3vectors.py

# Search for specific topics
uv run test_search_s3vectors.py "electric vehicles"
```

### 4.8 Enable Automated Research (Optional)

To enable research every 2 hours:

Edit `terraform.tfvars`:
```hcl
scheduler_enabled = true  # Changed from false
```

Apply changes:
```bash
cd ../../terraform/4_researcher
terraform apply
```

## Part 5: Database Setup

Deploy Aurora Serverless v2 PostgreSQL with Data API enabled.

### 5.1 Configure and Deploy

```bash
# Navigate to database terraform directory
cd terraform/5_database

# Copy and edit variables
cp terraform.tfvars.example terraform.tfvars
```

Edit `terraform.tfvars`:
```hcl
aws_region = "us-east-1"
min_capacity = 0.5  # Minimum ACUs (~$43/month)
max_capacity = 1.0  # Keep low for development
```

Deploy:
```bash
terraform init
terraform apply  # Takes 10-15 minutes
```

### 5.2 Save Database ARNs

After deployment, get the ARNs:
```bash
terraform output
```

Add to `.env`:
```bash
# Part 5 - Database
AURORA_CLUSTER_ARN=arn:aws:rds:...
AURORA_SECRET_ARN=arn:aws:secretsmanager:...
```

### 5.3 Test Database Connection

```bash
# Navigate to database directory
cd ../../backend/database

# Test Data API connection
uv run test_data_api.py
```

Expected output: "Successfully connected to Aurora using Data API!"

### 5.4 Run Migrations and Seed Data

```bash
# Run database migrations
uv run run_migrations.py

# Load seed data (22 popular ETFs)
uv run seed_data.py
```

### 5.5 Create Test Data (Optional)

```bash
# Reset database with test user and portfolio
uv run reset_db.py --with-test-data
```

## Part 6: Agent Orchestra

Deploy five specialized AI agents that collaborate for portfolio analysis.

### 6.1 Get Polygon API Key

1. Go to [polygon.io](https://polygon.io)
2. Sign up for free account
3. Verify email
4. Copy API key from dashboard

Free tier includes:
- 5 API calls/minute
- End-of-day prices
- Perfect for development

### 6.2 Configure Agents

Add to `.env`:
```bash
# Part 6 - Agents
BEDROCK_MODEL_ID=us.amazon.nova-pro-v1:0
BEDROCK_REGION=us-west-2
POLYGON_API_KEY=your_polygon_key
POLYGON_PLAN=free
```

### 6.3 Test Agents Locally

Test each agent individually:

```bash
# InstrumentTagger
cd backend/tagger
uv run test_simple.py

# Report Writer
cd ../reporter
uv run test_simple.py

# Chart Maker
cd ../charter
uv run test_simple.py

# Retirement Specialist
cd ../retirement
uv run test_simple.py

# Financial Planner
cd ../planner
uv run test_simple.py
```

All tests should pass (5/5).

### 6.4 Package Lambda Functions

```bash
# Navigate to backend directory
cd backend

# Package all agents (takes 2-3 minutes)
uv run package_docker.py
```

This creates five deployment packages:
- `tagger_lambda.zip` (~52 MB)
- `reporter_lambda.zip` (~68 MB)
- `charter_lambda.zip` (~54 MB)
- `retirement_lambda.zip` (~55 MB)
- `planner_lambda.zip` (~72 MB)

### 6.5 Configure and Deploy Infrastructure

```bash
# Navigate to agents terraform directory
cd ../terraform/6_agents

# Copy and edit variables
cp terraform.tfvars.example terraform.tfvars
```

Edit `terraform.tfvars`:
```hcl
aws_region = "us-east-1"
aurora_cluster_arn = ""  # Leave empty - auto-discovered
aurora_secret_arn = ""   # Leave empty - auto-discovered
vector_bucket = "alex-vectors-YOUR_ACCOUNT_ID"
bedrock_model_id = "us.amazon.nova-pro-v1:0"
bedrock_region = "us-west-2"
sagemaker_endpoint = "alex-embedding-endpoint"
polygon_api_key = "your_polygon_key"
polygon_plan = "free"
```

Deploy:
```bash
terraform init
terraform apply  # Takes 3-5 minutes
```

### 6.6 Deploy Lambda Code

```bash
# Navigate to backend directory
cd ../../backend

# Update all Lambda functions
uv run deploy_all_lambdas.py
```

### 6.7 Test Deployed Agents

Test each agent in AWS (run 3 times each for reliability):

```bash
# Test each agent
cd backend/tagger
uv run test_full.py

cd ../reporter
uv run test_full.py

cd ../charter
uv run test_full.py

cd ../retirement
uv run test_full.py

cd ../planner
uv run test_full.py
```

### 6.8 Test Complete System

```bash
# Navigate to backend directory
cd backend

# Test via SQS
uv run test_full.py
```

This sends a message to SQS and the complete agent orchestra processes it (90-120 seconds).

Add to `.env`:
```bash
# Part 6 - SQS
SQS_QUEUE_URL=https://sqs.us-east-1.amazonaws.com/.../alex-analysis-jobs
```

## Part 7: Frontend & API

Deploy the React frontend and FastAPI backend.

### 7.1 Set Up Clerk Authentication

1. Sign up at [clerk.com](https://clerk.com)
2. Create new application or use existing
3. Enable **Email** sign-in (optionally add Google)
4. Get credentials from **API Keys**:
   - Publishable Key (starts with `pk_`)
   - Secret Key (starts with `sk_`)
   - JWKS Endpoint URL (under "Show JWT Public Key")

### 7.2 Configure Frontend Environment

```bash
# Create frontend environment file
cd frontend
cp .env.local.example .env.local
```

Edit `frontend/.env.local`:
```bash
# Clerk Authentication
NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=pk_test_...
CLERK_SECRET_KEY=sk_test_...

# API URL (for production, will be updated after deployment)
NEXT_PUBLIC_API_URL=http://localhost:8000
```

### 7.3 Configure Backend Environment

Add to root `.env`:
```bash
# Part 7 - Clerk Authentication
CLERK_JWKS_URL=https://your-app.clerk.accounts.dev/.well-known/jwks.json
```

### 7.4 Test Locally

```bash
# Navigate to frontend directory
cd frontend

# Install dependencies
npm install

# Navigate to scripts directory
cd ../scripts

# Start both frontend and backend
uv run run_local.py
```

Access application:
- Frontend: http://localhost:3000
- API Docs: http://localhost:8000/docs

Test features:
1. Sign in with Clerk
2. Navigate to Accounts page
3. Click "Populate Test Data"
4. Explore account management

### 7.5 Package API Lambda

```bash
# Navigate to API directory
cd backend/api

# Package Lambda function
uv run package_docker.py
```

Creates `api_lambda.zip` with dependencies.

### 7.6 Configure and Deploy Infrastructure

```bash
# Navigate to frontend terraform directory
cd ../../terraform/7_frontend

# Copy and edit variables
cp terraform.tfvars.example terraform.tfvars
```

Edit `terraform.tfvars`:
```hcl
aws_region = "us-east-1"
clerk_jwks_url = "https://your-app.clerk.accounts.dev/.well-known/jwks.json"
clerk_issuer = "https://your-app.clerk.accounts.dev"
```

Deploy:
```bash
terraform init
terraform apply  # Takes 10-15 minutes (CloudFront is slow)
```

Save outputs:
```bash
terraform output
```

Note the `cloudfront_url` and `api_gateway_url`.

### 7.7 Build and Deploy Frontend

```bash
# Navigate to frontend directory
cd ../../frontend

# Update API URL in .env.local
# Edit NEXT_PUBLIC_API_URL to use the api_gateway_url from terraform output

# Build production version
npm run build

# Navigate to scripts directory
cd ../scripts

# Deploy to S3 and invalidate CloudFront
uv run deploy.py
```

### 7.8 Test Production Deployment

1. Open CloudFront URL in browser
2. Sign in with Clerk
3. Navigate to Accounts and populate test data
4. Go to **Advisor Team**
5. Click **Start New Analysis**
6. Watch agent progress visualization (60-90 seconds)
7. Review analysis results in tabs:
   - Overview
   - Charts
   - Retirement
   - Recommendations

## Part 8: Cloudwatch Monitoring and LangFuse Observability

Add monitoring, observability, and enterprise-grade features.

### 8.1 Set Up LangFuse (Optional)

1. Go to [cloud.langfuse.com](https://cloud.langfuse.com)
2. Create account and organization
3. Create project: "alex-financial-advisor"
4. Navigate to **Settings** → **API Keys**
5. Create new API key pair
6. Copy Public Key and Secret Key

Add to `.env`:
```bash
# Part 8 - LangFuse
LANGFUSE_PUBLIC_KEY=pk-lf-...
LANGFUSE_SECRET_KEY=sk-lf-...
LANGFUSE_HOST=https://cloud.langfuse.com
```

Update `terraform/6_agents/terraform.tfvars`:
```hcl
langfuse_public_key = "pk-lf-..."
langfuse_secret_key = "sk-lf-..."
langfuse_host = "https://cloud.langfuse.com"
openai_api_key = "sk-..."  # Required for trace export
```

### 8.2 Redeploy Agents with Observability

```bash
# Package all agents with observability
cd backend
uv run package_docker.py

# Deploy infrastructure with LangFuse variables
cd ../terraform/6_agents
terraform apply

# Update Lambda code
cd ../../backend
uv run deploy_all_lambdas.py
```

### 8.3 Configure Monitoring Dashboards

```bash
# Navigate to enterprise terraform directory
cd terraform/8_enterprise

# Copy and edit variables
cp terraform.tfvars.example terraform.tfvars
```

Edit `terraform.tfvars` with your deployment details.

Deploy dashboards:
```bash
terraform init
terraform apply
```

This creates:
- Bedrock & SageMaker metrics dashboard
- Agent activity monitoring dashboard
- CloudWatch alarms

### 8.4 Watch Agent Logs

```bash
# Navigate to backend directory
cd ../../backend

# Watch all agent logs in real-time
uv run watch_agents.py
```

Options:
```bash
uv run watch_agents.py --lookback 10  # Look back 10 minutes
uv run watch_agents.py --interval 1   # Poll every 1 second
```

### 8.5 Set Up Billing Alerts

1. Sign in to AWS Console as root
2. Go to **Billing Dashboard**
3. Click **Budgets** → **Create budget**
4. Choose **Cost budget**
5. Set amount (e.g., $100/month)
6. Configure alerts at 50%, 80%, 100%
7. Enter email for notifications

## Testing

### Local Testing

Test all components locally before deploying:

```bash
# Test SageMaker endpoint
cd backend
aws sagemaker-runtime invoke-endpoint --endpoint-name alex-embedding-endpoint --body fileb://vectorize_me.json /dev/stdout

# Test ingestion
cd backend/ingest
uv run test_ingest_s3vectors.py

# Test search
uv run test_search_s3vectors.py "technology stocks"

# Test agents locally
cd backend/tagger
uv run test_simple.py
# Repeat for other agents

# Test complete system
cd backend
uv run test_simple.py
```

### Production Testing

Test deployed services:

```bash
# Test API health
curl https://your-api-gateway-url/health

# Test researcher
cd backend/researcher
uv run test_research.py "AI market trends"

# Test agents via AWS
cd backend/tagger
uv run test_full.py
# Repeat for other agents

# Test complete pipeline
cd backend
uv run test_full.py

# Test frontend
# Open CloudFront URL in browser and run full analysis
```

### Scale Testing

Test with multiple users:

```bash
# Test multiple accounts
cd backend
uv run test_multiple_accounts.py

# Test concurrent requests
uv run test_scale.py
```

## Troubleshooting

### Common Issues

**Terraform State Issues**
```bash
# If endpoint already exists
terraform import aws_sagemaker_endpoint.embedding_endpoint alex-embedding-endpoint
terraform apply

# Or delete and recreate
aws sagemaker delete-endpoint --endpoint-name alex-embedding-endpoint
terraform apply
```

**Lambda Handler Errors**
```bash
# Check CloudWatch logs
aws logs tail /aws/lambda/alex-api --follow

# Common fixes:
# - Verify environment variables
# - Check Lambda IAM permissions
# - Ensure correct handler configuration
```

**Database Connection Failed**
```bash
# Check Aurora status
aws rds describe-db-clusters --db-cluster-identifier alex-aurora-cluster

# Check Data API enabled
aws rds describe-db-clusters --db-cluster-identifier alex-aurora-cluster --query 'DBClusters[0].EnableHttpEndpoint'

# Should return: true
```

**API Gateway 401 Errors**
- Verify Clerk JWKS URL in Lambda environment
- Check JWT token validity
- Ensure Clerk keys match in frontend and backend

**Agent Timeout Issues**
- Check Lambda timeout settings (60s for agents, 300s for planner)
- Verify Bedrock model access in us-west-2
- Review CloudWatch logs for specific errors

**CloudFront 403 Errors**
- Check S3 bucket policy
- Verify CloudFront OAI permissions
- Wait 15 minutes for propagation
- Try incognito browser window

**LangFuse Traces Not Appearing**
```bash
# Check environment variables
aws lambda get-function-configuration --function-name alex-planner | grep LANGFUSE

# Verify OPENAI_API_KEY is set (required)
aws lambda get-function-configuration --function-name alex-planner | grep OPENAI_API_KEY

# Watch logs for LangFuse messages
cd backend
uv run watch_agents.py --lookback 5
```

### Debugging Tips

1. **Check CloudWatch Logs**
   ```bash
   # API logs
   aws logs tail /aws/lambda/alex-api --follow
   
   # Agent logs
   aws logs tail /aws/lambda/alex-planner --follow
   ```

2. **Verify Environment Variables**
   ```bash
   # Check Lambda environment
   aws lambda get-function-configuration --function-name alex-api
   ```

3. **Test Connectivity**
   ```bash
   # Test SQS
   aws sqs send-message --queue-url $SQS_QUEUE_URL --message-body "test"
   
   # Test Aurora
   cd backend/database
   uv run test_data_api.py
   ```

4. **Monitor Metrics**
   - CloudWatch dashboards for real-time metrics
   - SQS queue depth
   - Lambda concurrent executions
   - Bedrock throttling

## Cost Management

### Monthly Cost Estimate

Development environment:
- Aurora Serverless v2: $43-60
- Lambda: $1-2
- API Gateway: $1-4
- SageMaker Serverless: $1-2
- S3 + CloudFront: $1-2
- Bedrock: $0.01-0.10 per analysis
- **Total: ~$50-70/month**

### Cost Optimization

1. **Pause Aurora when not developing**
   ```bash
   cd terraform/5_database
   terraform destroy
   # Recreate when needed
   terraform apply
   ```

2. **Monitor costs weekly**
   ```bash
   aws ce get-cost-and-usage \
     --time-period Start=2024-01-01,End=2024-01-31 \
     --granularity MONTHLY \
     --metrics "UnblendedCost" \
     --group-by Type=DIMENSION,Key=SERVICE
   ```

3. **Set up billing alerts**
   - AWS Budgets at 50%, 80%, 100% of monthly budget
   - Email notifications for unexpected charges

4. **Use AWS Free Tier**
   - Lambda: 1M requests/month free
   - API Gateway: 1M requests/month free
   - CloudWatch: Basic monitoring free

## Cleanup

### Destroy Individual Parts

To remove specific components:

```bash
# Part 8 - Enterprise
cd terraform/8_enterprise
terraform destroy

# Part 7 - Frontend
cd terraform/7_frontend
terraform destroy

# Part 6 - Agents
cd terraform/6_agents
terraform destroy

# Part 5 - Database
cd terraform/5_database
terraform destroy

# Part 4 - Researcher
cd terraform/4_researcher
terraform destroy

# Part 3 - Ingestion
cd terraform/3_ingestion
terraform destroy

# Part 2 - SageMaker
cd terraform/2_sagemaker
terraform destroy
```

### Complete Cleanup

To remove everything:

```bash
# Navigate to terraform directory
cd terraform

# Destroy in reverse order (8 → 2)
for dir in 8_enterprise 7_frontend 6_agents 5_database 4_researcher 3_ingestion 2_sagemaker; do
  cd $dir
  terraform destroy -auto-approve
  cd ..
done
```

### Manual Cleanup

Some resources require manual deletion:

1. **S3 Vector Bucket**
   - Go to S3 Console → Vector buckets
   - Delete `alex-vectors-{account-id}`

2. **CloudWatch Log Groups**
   ```bash
   aws logs describe-log-groups --log-group-name-prefix "/aws/lambda/alex"
   # Delete each log group
   aws logs delete-log-group --log-group-name [name]
   ```

3. **ECR Repository**
   ```bash
   aws ecr delete-repository --repository-name alex-researcher --force
   ```

## Next Steps

After completing the setup:

1. **Explore the Application**
   - Create multiple accounts with different strategies
   - Test with various ETFs and stocks
   - Run multiple analyses
   - Export reports

2. **Customize Features**
   - Modify agent prompts for different analysis styles
   - Add new chart types
   - Implement portfolio rebalancing
   - Integrate with brokerages

3. **Production Readiness**
   - Enable WAF for API protection
   - Set up VPC endpoints for private communication
   - Implement automated backups
   - Add error tracking (e.g., Sentry)
   - Set up CI/CD pipelines

4. **Monitor and Optimize**
   - Review LangFuse traces
   - Analyze CloudWatch metrics
   - Optimize agent prompts
   - Fine-tune Lambda memory settings

## Support

For issues or questions:
- Review troubleshooting section
- Check CloudWatch logs
- Create GitHub issue
- Review deployment guides in `/guides` directory

## Additional Resources

- [AWS Documentation](https://docs.aws.amazon.com/)
- [Terraform AWS Provider](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)
- [OpenAI Agents SDK](https://github.com/openai/openai-agents)
- [LangFuse Documentation](https://langfuse.com/docs)
- [Clerk Documentation](https://clerk.com/docs)