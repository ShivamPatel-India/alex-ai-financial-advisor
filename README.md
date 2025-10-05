# 🧠 Alex – The AI Financial Advisor  
*An Agentic, Cloud-Native Financial Intelligence Platform*

[![AWS](https://img.shields.io/badge/Cloud-AWS-orange?logo=amazonaws)](https://aws.amazon.com)
[![Python](https://img.shields.io/badge/Backend-Python_3.12-blue?logo=python)](https://www.python.org)
[![Next.js](https://img.shields.io/badge/Frontend-Next.js-black?logo=next.js)](https://nextjs.org/)
[![Terraform](https://img.shields.io/badge/IaC-Terraform-623ce4?logo=terraform)](https://www.terraform.io/)
[![LangFuse](https://img.shields.io/badge/Observability-LangFuse-purple)](https://www.langfuse.com)
[![License](https://img.shields.io/badge/license-MIT-green)](LICENSE)
---

## 🔗 Live Demo
🌐 [Access Alex Live](YOUR_LIVE_URL)

---

## 🧩 Overview
**Alex** is an **AI-powered financial advisor** that delivers personalized portfolio analysis, market insights, and retirement planning through a collaborative team of specialized AI agents.

It combines **AWS’s serverless infrastructure** with **agentic orchestration**, **explainable reasoning**, and **enterprise-grade security** — creating a platform that’s powerful, scalable, and cost-efficient (typically <$50/month to operate).

Alex automatically researches financial markets, enriches knowledge through embeddings, and produces rich, interactive financial analyses for end users.

---

## 🚀 Key Features
- Conversational AI powered by **AWS Bedrock Nova models** (can be changed in .env file)
- Multi-agent orchestration for **research, reporting, visualization, and projections**
- Fully serverless architecture built on **AWS Lambda, S3, API Gateway, and Aurora**
- Infrastructure as Code with **Terraform**
- **Secure, observable, and scalable** with LangFuse and CloudWatch integration

---

## 🧩 Repository Structure
```
/frontend/          # Next.js frontend
/backend/           # FastAPI backend
/terraform/         # Infrastructure as Code
/docs/              # Documentation (UI guide & architecture)
/screenshot/        # Screenshots
```

---

## 📘 Documentation

| Guide | Description |
|-------|-------------|
| [UI & Usage Guide](./docs/ui_guide.md) | Screenshots and user instructions |
| [Agentic System Architecture](./docs/agentic_architecture.md) | Infrastructure, orchestration, and scalability overview |
| [AWS Infra](./docs/architecture.md) | Core system prompt for Alex |

---

## 🧑‍💻 Tech Stack
**Frontend:** Next.js, Tailwind CSS  
**Backend:** FastAPI, Python 3.12  
**AI/ML:** AWS Bedrock, SageMaker, LangFuse  
**Infrastructure:** Terraform, AWS Lambda, S3, CloudFront, Aurora  

---

## 🧠 Architecture Snapshot
![Architecture](/screenshot/architecture-diagram.png)

---

## 📈 Deployment
1. Deploy AWS infrastructure via Terraform:  
   ```bash
   terraform apply
   ```
3. Access deployed application via CloudFront endpoint  

