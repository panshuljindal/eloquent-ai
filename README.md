# Eloquent AI – RAG Chatbot (Full‑Stack)

This repository aggregates two modules (backend and frontend) built for the AI Engineer technical assignment. The system is a persisted chat application using Retrieval‑Augmented Generation (RAG) over a fintech FAQ knowledge base. It supports anonymous and returning users, retrieves relevant context from Pinecone, and generates answers via OpenAI.

- Backend: FastAPI (Python 3.11), SQLModel, OpenAI, Pinecone
- Frontend: React + TypeScript + TailwindCSS

Links to module READMEs:
- Backend: `backend/README.md`
- Frontend: `frontend/README.md`

---

## Live Backend API

- Public API base: `http://18.223.20.255:5000`
  - Auth: `/api/auth/signup`, `/api/auth/login`
  - Chat: `/api/chat/conversations`, `/api/chat/messages/:id`, `/api/chat/create`, `/api/chat/delete/:id`, `/api/chat/summarize/:id`
- Hosting: EC2 behind a reverse proxy. Health check at `/health`.
- Throughput: On a single EC2 instance (for example, t3.medium) with Gunicorn + Uvicorn workers, the service can sustain on the order of a few dozen requests per second for chat creation under typical OpenAI latency and account rate limits. Horizontal scaling (see plan below) raises this linearly. Replace with your measured RPS for your instance type.

Note: The frontend is currently run locally

---

## What’s implemented

### Backend (FastAPI)
- RAG chat flow
  - Retrieves top‑k context from Pinecone before answering.
  - Orchestrates prompts with a strict System prompt to avoid hallucinations.
  - Uses OpenAI `gpt-4o` to generate answers.
- Persistence (SQLite by default via SQLModel)
  - Users, conversations, and messages.
  - Soft delete for conversations and listing by user.
- Guardrails and safety
  - Prompt‑injection detection and rejection.
  - PII/secret redaction in inputs and outputs (emails, phones, common secret patterns).
  - Output sanitation to strip active content/HTML.
- Summarization
  - Per‑conversation summary endpoint to produce a concise recap with next steps.
- Simple authentication
  - Signup and login with bcrypt password hashing. Returning users can see prior chats.
- CORS configured for development; adjust for production.

---

## Run locally

Backend
1) Python 3.11, create venv and install:
```
cd backend
uv sync
source .venv/bin/activate
```
2) Create `backend/.env`:
```
DATABASE_URL=sqlite:///./chat.db
OPENAI_API_KEY=sk-...
PINECONE_API_KEY=pcn-...
PINECONE_HOST=your-index-host
PINECONE_NAMESPACE=__default__
```
3) Start API:
```
uvicorn main:app --host 0.0.0.0 --port 8000 --reload
```

Frontend
1) Install deps:
```
cd frontend
npm ci
```
2) Set API base in `frontend/.env`:
```
REACT_APP_API_BASE=http://localhost:8000
```
3) Run dev server:
```
npm start
```

---

## Deployment plan (AWS)

Current state
- Backend is deployed on a single EC2 instance running the Dockerized FastAPI app. Requests terminate at a reverse proxy, which forwards to Gunicorn/Uvicorn. API base exposed at `http://18.223.20.255:5000`.
- Frontend is not deployed yet; run locally pointing to the API base.

Target architecture (scalable, production‑ready)
1) Containerization & registry
- Add a Dockerfile (Gunicorn + Uvicorn worker class) for the backend image.
- Build and push to Amazon ECR on each merge to `main` via GitHub Actions.

2) Network & security
- VPC with public ALB and private subnets for ECS tasks.
- Security Groups: ALB (80/443) → ECS service target group; egress to OpenAI/Pinecone.

3) Compute layer (ECS on EC2 or Fargate)
- Use ECS with an EC2 capacity provider and an Auto Scaling Group created via a Launch Template (ECS‑optimized AMI). Start with t3.medium/t3.large.
- Alternatively, use Fargate to avoid managing EC2 capacity.

4) Service definition
- ECS Task Definition exposing container port 8000; environment from SSM Parameter Store/Secrets Manager (OpenAI key, Pinecone config, DATABASE_URL).
- ECS Service behind an Application Load Balancer with health check on `/health`.
- Autoscaling: target tracking on CPU 60–70% and/or ALB RequestCountPerTarget; min 2 tasks across AZs.

5) Data layer
- Replace SQLite with Amazon RDS (PostgreSQL). Introduce Alembic for migrations.
- Use IAM auth where appropriate; restrict security groups and subnets.

6) Observability & ops
- CloudWatch Logs for app logs; ALB access logs to S3.
- CloudWatch metrics and alarms (5xx rate, high latency, CPU/memory, OpenAI/Pinecone error spikes).
- Optional tracing with OpenTelemetry → X-Ray.

7) CI/CD
- GitHub Actions pipelines:
  - Backend: test, build, push ECR, deploy ECS (blue/green via CodeDeploy or rolling update).
  - Frontend: build, upload to S3, invalidate CloudFront cache.

8) Frontend hosting (when ready)
- S3 static hosting + CloudFront (HTTPS, caching, compression).
- Environment config via build‑time variables (`REACT_APP_API_BASE` set to the ALB domain or API gateway).

9) Security hardening
- Restrict CORS to known origins, enable HTTPS end‑to‑end, rotate secrets.
- Add JWT‑based auth (e.g., Cognito) and enforce conversation ownership on all endpoints.
- Add rate limiting (API Gateway or NGINX, or app‑level limiter backed by Redis/ElastiCache).

Throughput and scaling
- A single t3.medium typically handles tens of RPS for the chat endpoints; overall throughput is dominated by model and Pinecone latencies and provider rate limits. Horizontal scaling via ECS increases capacity linearly. Use request queues or concurrency control to respect OpenAI rate limits.

---

## Notable features beyond the baseline
- Guardrails: prompt‑injection detection, PII/secret redaction, safe output formatting.
- Conversation summary endpoint for quick recaps.
- RAG with Pinecone (category + text fields returned as context to the model).
- Message filtering so system and guardrail messages are hidden/normalized in the client.
- Clean UI with dark mode, Markdown rendering, and guest mode.

---

## Next steps
- Swap SQLite for Postgres + Alembic migrations.
- JWT/OAuth login and authorization on conversation resources.
- Rate limiting and abuse protection.
- Caching layer for repeated queries (Redis) where appropriate.

---

## License
Proprietary – All rights reserved (update if needed).
