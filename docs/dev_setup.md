# 🧑‍💻 Developer Setup Guide – Alex (Portfolio AI Assistant)

Welcome to the **developer setup guide** for the Portfolio AI Assistant — *Alex*.  
This document helps you or other developers set up, test, and deploy the system both locally and on AWS.

---

## 🧠 Overview

Alex is an **AI-powered Financial Advisor** that answers questions about your experience, portfolio, and financial analysis using AWS Bedrock models and a multi-agent orchestration system.

It’s built for scalability, observability, and modularity — combining **FastAPI**, **Next.js**, and **AWS Serverless** technologies.

---

## ⚙️ Prerequisites

### 🧩 Core Requirements

| Tool | Version | Purpose |
|------|----------|----------|
| [Python](https://www.python.org/downloads/) | 3.12+ | Backend (FastAPI + AWS SDK) |
| [uv](https://docs.astral.sh/uv/) | latest | Python package manager |
| [Node.js](https://nodejs.org/en/download/) | 20+ | Frontend (Next.js) |
| [Docker](https://www.docker.com/) | latest | Packaging Lambda functions |
| [Terraform](https://developer.hashicorp.com/terraform/downloads) | 1.5+ | Infrastructure as Code |
| [AWS CLI](https://aws.amazon.com/cli/) | v2 | Access and manage AWS resources |
| [GitHub Actions](https://docs.github.com/en/actions) | optional | CI/CD pipeline |
| [Clerk](https://clerk.com) | free tier | Authentication & user management |

---

## 🧩 System Architecture

Alex follows a modular **multi-agent, serverless architecture** built entirely on AWS.  

### Core Components
- **Frontend:** Next.js 15 + Tailwind (S3 + CloudFront)
- **Backend:** FastAPI on AWS Lambda
- **AI Models:** AWS Bedrock (Nova Pro / Nova Lite)
- **Database:** Aurora Serverless v2 with Data API
- **Vector Store:** AWS S3 Vectors (90% cheaper than OpenSearch)
- **Orchestration:** SQS + EventBridge Scheduler
- **Monitoring:** CloudWatch + LangFuse

### Agent Roles
| Agent | Function |
|--------|-----------|
| Planner | Orchestrates & coordinates portfolio analysis |
| Tagger | Classifies instruments and allocations |
| Reporter | Generates portfolio reports and recommendations |
| Charter | Creates charts and JSON visualization data |
| Retirement Specialist | Runs Monte Carlo projections |
| Researcher | Autonomous web research, updates knowledge base |

---

## 📂 Repository Structure

```
/frontend/        # Next.js + Clerk frontend
/backend/         # FastAPI backend + Bedrock integrations
/backend/agents/  # Lambda-based AI agent services
/terraform/       # AWS infrastructure (SageMaker, Lambda, Aurora, etc.)
/docs/            # Documentation
/screenshot/      # UI screenshots
```

---

## 🔧 1. Environment Setup

### Step 1: Clone Repository
```bash
git clone https://github.com/YOUR_USERNAME/portfolio-ai-assistant.git
cd portfolio-ai-assistant
```

### Step 2: Configure Environment Variables
Create a `.env` file in the **project root**:

```bash
# AWS Configuration
AWS_ACCOUNT_ID=123456789012
DEFAULT_AWS_REGION=us-east-1

# Bedrock Configuration
BEDROCK_MODEL_ID=amazon.nova-lite-v1:0
BEDROCK_REGION=us-west-2

# SageMaker
SAGEMAKER_ENDPOINT=Alex-embedding-endpoint

# Vector Store (S3 Vectors)
VECTOR_BUCKET=Alex-vectors-123456789012
ALEX_API_ENDPOINT=https://xxxxxxxxxx.execute-api.us-east-1.amazonaws.com/prod/ingest
ALEX_API_KEY=your-api-key-here

# Clerk Authentication
CLERK_JWKS_URL=https://your-app.clerk.accounts.dev/.well-known/jwks.json
CLERK_ISSUER=https://your-app.clerk.accounts.dev

# Optional Observability
LANGFUSE_PUBLIC_KEY=pk-xxxxx
LANGFUSE_SECRET_KEY=sk-xxxxx
```

> 📝 You can also copy from `.env.example` if available.

---

## 🐍 2. Backend Setup (FastAPI + Bedrock)

### Step 1: Install Dependencies
```bash
cd backend
uv init --bare
uv python pin 3.12
uv add -r requirements.txt
```

### Step 2: Run Locally
```bash
uv run uvicorn server:app --reload
```

✅ Backend available at `http://localhost:8000`  
Visit `http://localhost:8000/docs` for the FastAPI Swagger UI.

---

## 💻 3. Frontend Setup (Next.js + Clerk)

### Step 1: Navigate & Install Dependencies
```bash
cd frontend
npm install
```

### Step 2: Configure `.env.local`
```bash
NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=pk_test_your-key
CLERK_SECRET_KEY=sk_test_your-secret
NEXT_PUBLIC_API_URL=http://localhost:8000
```

### Step 3: Start Frontend
```bash
npm run dev
```

✅ Frontend available at `http://localhost:3000`.

---

## ☁️ 4. AWS Deployment (Terraform + Docker)

### Step 1: Configure AWS CLI
```bash
aws configure
```
Enter:
- AWS Access Key ID  
- AWS Secret Access Key  
- Default region: `us-east-1`

### Step 2: Deploy Infrastructure Sequentially
```bash
# 1. IAM + Permissions
cd terraform/1_permissions
terraform init && terraform apply

# 2. SageMaker
cd ../2_sagemaker
terraform init && terraform apply

# 3. Ingestion (S3 Vectors + Lambda + API Gateway)
cd ../3_ingestion
terraform apply

# 4. Researcher Agent (App Runner)
cd ../4_researcher
terraform apply

# 5. Database (Aurora Serverless v2)
cd ../5_database
terraform apply

# 6. Agents (Lambda + SQS)
cd ../6_agents
terraform apply

# 7. Frontend + API (CloudFront)
cd ../7_frontend
terraform apply
```

### Cost Optimization

To reduce costs when not actively developing:

```bash
# Stop Aurora to save ~$43/month
# In alex/terraform/5_database directory
terraform destroy

# Or destroy everything
# Run in each terraform directory in reverse order (7, 6, 5, 4, 3, 2)
terraform destroy
```

> 💡 Each Terraform directory is independent and safe to deploy or destroy individually.

---

## 🧱 5. Packaging & Deploying Lambda Agents

### Step 1: Build All Agents
```bash
cd backend
uv run package_docker.py
```

Creates:
```
tagger_lambda.zip
reporter_lambda.zip
charter_lambda.zip
retirement_lambda.zip
planner_lambda.zip
```

### Step 2: Deploy to AWS
```bash
uv run deploy_all_lambdas.py
```

Verify:
```bash
aws lambda list-functions --query 'Functions[*].FunctionName'
```

---

## 🧪 6. Testing & Verification

### Local Testing
```bash
uv run test_simple.py
```
Tests all agents locally with mock data.

### AWS Testing
```bash
uv run test_full.py
```
End-to-end test of SQS → Planner → Reporter → Charter → Retirement → Aurora.

### Key API Endpoints
| Endpoint | Description |
|-----------|--------------|
| `GET /api/user` | Returns user data |
| `GET /api/accounts` | Lists investment accounts |
| `POST /api/positions` | Adds a position |
| `POST /api/analyze` | Triggers full agent analysis |
| `GET /api/jobs/{job_id}` | Retrieves results |

---

## 🔁 7. CI/CD Integration (GitHub Actions)

Alex uses **GitHub Actions** for zero-key AWS deployments via **OIDC**.

Example workflow:
```yaml
on:
  push:
    branches: [ main ]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: aws-actions/configure-aws-credentials@v3
        with:
          role-to-assume: arn:aws:iam::${{ secrets.AWS_ACCOUNT_ID }}:role/github-actions
          aws-region: us-east-1
      - name: Deploy Infrastructure
        run: |
          cd terraform/7_frontend
          terraform init
          terraform apply -auto-approve
```

---

## 🔒 8. Security Best Practices

| Category | Description |
|-----------|-------------|
| IAM | Least privilege roles for every Lambda |
| Auth | JWT verification via Clerk JWKS |
| CORS | Restricted origins to prevent CSRF/XSS |
| WAF | Protects API Gateway from DDoS |
| Secrets | Stored securely in AWS Secrets Manager |
| Observability | LangFuse traces all agent events |

---

## 📊 9. Monitoring & Observability

### CloudWatch Logs
```bash
aws logs tail /aws/lambda/Alex-planner --follow
```

### LangFuse Dashboard
- View token usage, cost, and latency metrics  
- Inspect traces for every agent’s execution  

---

## 💰 10. Cost Summary

| Service | Monthly Est. | Notes |
|----------|---------------|-------|
| Aurora Serverless v2 | ~$43 | Database |
| S3 Vectors | ~$20 | Vector Store |
| Lambda + API Gateway | ~$5 | Compute + Networking |
| CloudFront + S3 | <$1 | Frontend CDN |
| Bedrock + SageMaker | ~$10 | AI & Embeddings |
| **Total** | **~$60–70** | Development mode |

---

## 🧹 11. Cleanup

To stop all resources and avoid charges:
```bash
cd terraform
terraform destroy
```

Destroy in reverse order:
```
7_frontend → 6_agents → 5_database → 4_researcher → 3_ingestion → 2_sagemaker → 1_permissions
```

---

## 🧩 12. Common Issues & Fixes

| Issue | Fix |
|--------|-----|
| **CORS Errors** | Ensure allowed origins in API Gateway match frontend domain |
| **Model Access Denied** | Check Bedrock model permissions (Nova Lite / Pro) |
| **Lambda Timeout** | Increase memory or timeout in Terraform configs |
| **Terraform Lock** | Delete `.terraform.lock.hcl` and re-run `terraform init` |
| **Missing Traces in LangFuse** | Ensure `OPENAI_API_KEY` and LangFuse keys are set |
| **CloudFront 403** | Revalidate cache or fix bucket policy permissions |

---

## 🧠 13. Advanced Features

- **Multi-agent orchestration** for parallel execution  
- **Explainability** via rationale in Tagger & Reporter outputs  
- **Guardrails**: prompt injection detection and structured validation  
- **Observability**: end-to-end tracing with LangFuse  
- **Retry logic**: exponential backoff for transient errors  

---

## 🏁 Summary

You now have a complete, end-to-end **Agentic AI system** powered by AWS:  
✅ FastAPI backend with Bedrock integration  
✅ Multi-agent orchestration (Planner, Reporter, Charter, etc.)  
✅ Serverless, scalable, and observable architecture  
✅ Terraform-managed deployment and CI/CD  
✅ Enterprise-grade security and explainability  

---

**Author:** Shivam Patel  
**Year:** 2025  
© All rights reserved.
