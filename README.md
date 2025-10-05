# 🧠 Alex – The AI Financial Advisor  

🌐 [Live link](YOUR_LIVE_URL)

**Alex** is an **AI-powered financial advisor** that delivers personalized portfolio analysis, market insights, and retirement planning through a collaborative team of specialized AI agents.

It combines **AWS’s serverless infrastructure** with **agentic orchestration**, **explainable reasoning**, and **enterprise-grade security** — creating a platform that’s powerful, scalable, and cost-efficient

Alex automatically researches financial markets (uses [Polygon](https://polygon.io/) APIs), enriches knowledge through embeddings, and produces rich, interactive financial analyses for end users.

---

## 📘 Comprehensive READMEs

| Guide | Description |
|-------|-------------|
| [Agentic System Architecture](./docs/agentic_architecture.md) | Infrastructure, orchestration, and scalability overview |
| [Developer setup](./docs/dev_setup.md) | Instructions for developer setup |
| [UI & Usage Guide](./docs/ui_guide.md) | Screenshots and user instructions |
| [AWS Infra](./docs/architecture.md) | AWS Infrastructure used for deplyment|

---

## 🚀 Key Features
- Conversational AI powered by **AWS Bedrock Nova models** (can be changed in .env file)
- Multi-agent orchestration for **research, reporting, visualization, and projections**
- Fully serverless architecture built on **AWS Lambda, S3, API Gateway, and Aurora**
- Infrastructure as Code with **Terraform**
- **Secure, observable, and scalable** with LangFuse and CloudWatch integration

---

## 🧠 Architecture Snapshot
![Architecture](./screenshots/architecture-diagram.png)

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
## 🧑‍💻 Tech Stack
**Frontend:** Next.js, Tailwind CSS  
**Backend:** FastAPI, Python 3.12  
**AI/ML:** AWS Bedrock, SageMaker, LangFuse  
**Infrastructure:** Terraform, AWS Lambda, S3, CloudFront, Aurora  

---



