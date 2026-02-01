# TutorGraph — Product Spec + Greenfield Build Plan (Local-first, AWS-shaped)

**Document purpose:** This is a durable “pick-it-up-later” blueprint for building **TutorGraph**, a production-shaped GenAI learning platform project designed to align tightly to an **AI Software Engineer / AI Platform Engineering** job description (LLM + RAG + full-stack + cloud + CI/CD + accessibility + security).

**Primary target:** Local demo (free) that feels like AWS so the AWS deployment can be “turned on” later with minimal rework.

**Last updated:** 2026-02-01 (America/New_York)


---

## 0) One-paragraph pitch (for README / recruiter / interview)

**TutorGraph** is a **multi-tenant adaptive learning platform** that ingests course content, enables **grounded Q&A with citations** via **RAG + reranking**, generates **quizzes**, and adjusts **difficulty per learner** based on performance. It’s built as a **production-grade platform service** with org management (invites, roles, audit logs), evaluation harness, CI, accessibility checks, and an AWS-shaped architecture that can later deploy via Terraform (ECS/RDS/S3/DynamoDB).

---

## 1) What this greenfield project accomplishes (high-level outcomes)

By completing TutorGraph, you’ll have a repo that proves you can:

- **Build GenAI features that ship** (not just prompts): RAG + reranking + citations + evaluations.
- **Design production-shaped services:** multi-tenant, RBAC, audit logging, rate limiting, structured logs.
- **Do full-stack delivery:** React UI + FastAPI backend + Postgres + NoSQL-style events.
- **Operate like a platform engineer:** configuration, local parity, CI, deterministic tests, deploy-ready infra module.
- **Meet real-world requirements:** accessibility (WCAG 2.2 AA intent), performance posture, security posture.

This is exactly what “prototype → production” roles want as proof.

---

## 2) The job this is meant for (and why it’s aligned)

**Target job profile:** “AI Software Engineer” / “AI Platform Engineering” in a company building **adaptive learning** and **LLM-enabled digital platforms**.

**Why TutorGraph is a match:** It demonstrates the core capabilities from the job posting:

- AI-powered apps/services that are reliable, scalable, secure
- RAG with orchestration (LangGraph/LangChain)
- Collaboration-ready productionization: evals, CI/CD, provider abstraction
- Full-stack engineering (React + backend + relational + NoSQL patterns)
- Cloud/IaC posture (AWS-shaped local + optional Terraform deploy)
- Accessibility, performance, security concerns

---

## 3) System scope (MVP v1) — what “done” means

### Must-have features (MVP v1)
1. **Multi-tenant orgs**
   - Create tenant (org)
   - Invite users
   - Accept invite (link-based for local demo)
   - Tenant switcher in UI
   - Roles: Owner/Admin/Learner
2. **Courses + content ingest**
   - Create course
   - Upload text/markdown as “documents”
   - Chunking + embedding + indexing into Postgres (pgvector)
   - Document list w/ status (processing/indexed/failed)
3. **RAG Q&A**
   - Ask question within course
   - Retrieve top-N via vector search
   - Rerank to top-K
   - Answer using only retrieved context
   - Return citations (doc + chunk + snippet)
   - Feedback: helpful + grounded
4. **Quizzes + adaptive difficulty**
   - Generate quiz for a course
   - Store attempts
   - Grade attempt (with cited explanation)
   - Track mastery score per user/course
   - Adjust difficulty tier based on performance (last N attempts)
5. **Ops/quality**
   - Structured logs + request_id
   - Rate limiting (query + quiz generation)
   - Input limits + validation
   - Health endpoint
   - Evaluation harness (recall@k, citation coverage)
   - CI runs unit + integration tests (mock LLM)

### Explicit non-goals for v1
- PDF parsing/OCR (v1 accepts text/markdown; “paste extracted text” is okay)
- “Real SAML SSO execution” (we implement architecture + config UI; local demo uses dev auth)
- Payments/subscriptions
- Mobile app

---

## 4) UX / UI (yes, this has UI/UX)

### UI shape (simple but impressive)
- **Top bar:** Tenant switcher, Course switcher, Profile
- **Left nav:** Dashboard, Ask, Quiz, Content, Admin (role gated)

### Pages (MVP)
1. **Login (local demo)**
   - “Sign in as Demo Admin” / “Sign in as Demo Learner” (dev mode)
   - JWT still used so API auth path matches production
2. **Dashboard**
   - Current course summary
   - Mastery meter (0–100)
   - Recent Q&A + recent quiz score
3. **Content**
   - Upload docs (text/markdown)
   - Documents list + indexing status
   - Reindex action (admin)
4. **Ask**
   - Question input
   - Answer output + citations panel
   - Feedback controls
5. **Quiz**
   - Start quiz (shows difficulty tier)
   - Submit answers
   - Results: score + cited explanations + weak topics
6. **Admin**
   - Members + roles
   - Invite form + pending invites
   - SSO config screen (domain mapping)
   - Audit log viewer

### Accessibility checklist (WCAG 2.2 AA intent)
- Keyboard nav end-to-end
- Labels on all inputs
- Focus management after actions (answer loads, modal opens)
- Citations expandable via keyboard with ARIA attributes
- Visible focus ring and proper heading hierarchy

---

## 5) Architecture (Local-first, AWS-shaped)

### Core design principle
**All “cloud integrations” are behind adapters** so local and AWS share the same code paths. Only endpoints/config change.

### Target A (Local, free)
- API: FastAPI container
- Web: React container
- DB: Postgres container with pgvector
- AWS-like services: LocalStack
  - S3 for document storage (optional in v1; can store docs in Postgres too)
  - DynamoDB for events/audit stream (or Postgres audit_log + optional Dynamo stream)
- LLM: local model or mocked (for CI + deterministic tests)
- Auth: dev JWT issuer (or simple local auth) but API verifies JWT like production

### Target B (AWS optional module)
- Web: S3 + CloudFront
- API: ECS Fargate behind ALB
- DB: RDS Postgres (pgvector)
- Events: DynamoDB
- Logs: CloudWatch
- Secrets: SSM / Secrets Manager
- IaC: Terraform modules + CI deploy pipeline

### Mermaid architecture diagram
TutorGraph Architecture (Local-first, AWS-shaped)
================================================

Core principle:
  - All cloud integrations go through adapters.
  - Local + AWS share the same code paths.
  - Only endpoints/config change.

+-------------------+          +------------------+         +------------------------------+
|       User        |  HTTPS   |    React Web     |  HTTPS  |         FastAPI API          |
|  (Browser Client) +--------->|  (Frontend UI)   +-------->|  (Backend Service Layer)     |
+-------------------+          +------------------+         +---------------+--------------+
                                                                          |
                                                                          | SQL (pgvector)
                                                                          v
                                                              +---------------------------+
                                                              |   Postgres + pgvector     |
                                                              |  - tenants/courses/docs   |
                                                              |  - chunks + embeddings    |
                                                              |  - Q/A + citations        |
                                                              +---------------------------+
                                                                          |
                                                                          | (via Adapter Interfaces)
                               +------------------------------------------+------------------------------------------+
                               |                                          |                                          |
                               |                                          |                                          |
                               v                                          v                                          v
                   +---------------------------+             +---------------------------+             +---------------------------+
                   |  S3-Compatible Storage    |             |   DynamoDB-Style Events   |             |     LLM / Embeddings      |
                   |  - document blobs         |             |  - event stream / audit   |             |  - local or mocked        |
                   |  - optional in v1         |             |  - optional in v1         |             |  - deterministic for CI   |
                   +---------------------------+             +---------------------------+             +---------------------------+

Auth + Tenant Context
--------------------
  - Web sends JWT (Authorization: Bearer <token>)
  - Web sends selected tenant via header: X-Tenant-Id
  - API verifies JWT + checks membership in tenant


Target A: Local (Free) Implementation
------------------------------------
  React Web       -> local container
  FastAPI API     -> local container
  Postgres        -> local container + pgvector
  S3 Storage      -> LocalStack S3 (optional) OR store doc text in Postgres
  Events (NoSQL)  -> LocalStack DynamoDB (optional) OR use Postgres audit_log only
  LLM             -> local model OR mocked provider (best for CI)
  Auth            -> dev JWT issuer (but API verifies JWT like production)


Target B: AWS (Optional Deployment Module)
-----------------------------------------
  React Web       -> S3 + CloudFront
  FastAPI API     -> ECS Fargate behind ALB
  Postgres        -> RDS Postgres + pgvector
  S3 Storage      -> AWS S3
  Events (NoSQL)  -> DynamoDB
  Logs            -> CloudWatch
  Secrets         -> SSM / Secrets Manager
  IaC             -> Terraform modules + CI deploy pipeline


Adapter Map (Same Code Paths)
-----------------------------
  StorageProvider : LocalStack S3  <->  AWS S3
  EventStore      : LocalStack DDB <->  AWS DynamoDB
  VectorStore     : Local Postgres <->  RDS Postgres
  LLMProvider     : Mock/Local     <->  Bedrock/Azure/OpenAI
  AuthProvider    : Dev JWT        <->  OIDC (Cognito/Auth0/Okta/etc.)


---

## 6) Provider adapters (so you can “turn on AWS” later)

Implement these interfaces early:

- `AuthProvider`
  - Local dev issuer now
  - Later: OIDC provider (Cognito/Auth0/Okta/Entra)
- `LLMProvider`
  - Local model or mock now
  - Later: Bedrock / Azure OpenAI / OpenAI
- `StorageProvider`
  - LocalStack S3 now
  - Later: real S3
- `EventStore`
  - LocalStack DynamoDB now
  - Later: real DynamoDB
- `VectorStore`
  - Postgres pgvector local now
  - Later: RDS pgvector

**Why this matters:** your “AWS work” becomes configuration + Terraform, not rewrites.

---

## 7) Data model (MVP tables)

### TenantKit tables
- `users(id, external_auth_id, email, name, created_at)`
- `tenants(id, name, created_at)`
- `memberships(id, tenant_id, user_id, role, created_at)`
- `invites(id, tenant_id, email, role, token, expires_at, accepted_at)`
- `tenant_sso_connections(id, tenant_id, type, domains[], config_json, enabled)`
- `audit_log(id, tenant_id, actor_user_id, action, target_type, target_id, metadata_json, ts)`

### TutorGraph domain tables
- `courses(id, tenant_id, title, description, created_at)`
- `documents(id, tenant_id, course_id, title, source_type, status, created_at)`
- `chunks(id, tenant_id, course_id, document_id, chunk_index, content, embedding, metadata_json)`
- `questions(id, tenant_id, course_id, user_id, text, created_at)`
- `answers(id, tenant_id, course_id, question_id, text, model_provider, created_at)`
- `citations(id, tenant_id, answer_id, document_id, chunk_id, snippet, score)`
- `quizzes(id, tenant_id, course_id, difficulty_tier, created_at)`
- `quiz_questions(id, tenant_id, quiz_id, prompt, choices_json, correct_answer, citations_json)`
- `quiz_attempts(id, tenant_id, quiz_id, user_id, score, submitted_at, feedback_json)`
- `mastery(id, tenant_id, course_id, user_id, mastery_score, last_updated)`

### NoSQL (LocalStack DynamoDB) — optional but “AWS-shaped”
- `EventStream` (PK: tenant_id, SK: timestamp)
  - event_type, actor, metadata_json

---

## 8) API contract (MVP endpoints)

### Auth/Tenant context
- Bearer JWT required
- Tenant context: `X-Tenant-Id`
- API enforces membership + role

### Tenants
- `POST /tenants`
- `GET /tenants`
- `POST /tenants/{tenant_id}/invites`
- `POST /invites/{token}/accept`
- `GET /tenants/{tenant_id}/members`
- `PATCH /tenants/{tenant_id}/members/{user_id}`

### Courses/content
- `POST /courses`
- `GET /courses`
- `POST /courses/{course_id}/documents`
- `GET /courses/{course_id}/documents`
- `POST /documents/{doc_id}/reindex`

### RAG
- `POST /courses/{course_id}/query`
  - request: question, top_n, top_k, debug
  - response: answer, citations[], retrieval_stats, answer_id
- `POST /answers/{answer_id}/feedback`
  - helpful, grounded, comment?

### Quizzes
- `POST /courses/{course_id}/quizzes/generate`
- `POST /quizzes/{quiz_id}/attempts`
- `GET /courses/{course_id}/performance`

### Ops
- `GET /health`
- `GET /version`

---

## 9) RAG pipeline details (how it works)

### Ingestion
1. Receive doc (text/markdown)
2. Normalize text
3. Chunking strategy (MVP):
   - chunk size: ~600–1000 chars
   - overlap: ~100–150 chars
4. Embed chunks
5. Store chunks in Postgres (with pgvector embedding)
6. Mark document as indexed

### Query
1. Embed question
2. Vector search top N (30–50) within course scope
3. Rerank to top K (8–12) via hybrid scoring
4. Build context bundle (chunks + metadata)
5. Generate answer constrained to context
6. Return answer + citations (snippet, doc title, chunk id)

### Reranking (MVP local-friendly)
- Score = (vector similarity) + (keyword overlap / BM25-ish)
- Debug mode returns rerank scores + selected chunk IDs

### Grounding rule
- If no relevant chunks above threshold: return “Not found in course content.”

---

## 10) Adaptive difficulty details (how it works)

### Difficulty tiers
- Tier 1: recall
- Tier 2: explain/define
- Tier 3: apply in scenario
- Tier 4: synthesize/tricky distractors

### Mastery update (MVP)
- Keep last N attempts (e.g., 5)
- mastery_score = weighted average of recent scores
- If mastery_score > 85: increase tier
- If mastery_score < 60: decrease tier
- Else keep tier

---

## 11) Quality, security, observability (what makes it “production-grade”)

### Security/guardrails
- Rate limit query + quiz generation
- Input max sizes (doc length, question length)
- Timeouts/retries for model calls
- Strict CORS allowlist
- Security headers on web

### Observability
- Structured logs: request_id, tenant_id, user_id, latency_ms, status_code
- Log retrieval stats: top_n, top_k, rerank_method, selected_chunk_ids

### Evaluation harness (big differentiator)
- `data/eval_set/questions.jsonl` includes:
  - question
  - expected doc_id (or doc title)
  - expected chunk keywords
- Script computes:
  - recall@k (retrieved includes expected doc/chunk)
  - citation coverage rate
- CI runs retrieval eval with mocked LLM (cheap + deterministic)

---

## 12) Local demo packaging (how this becomes sharable)

### “One command” run
- `docker compose up` starts API, Web, Postgres, LocalStack
- `make demo` (or `npm run demo`) does:
  - migrate DB
  - seed demo tenant + users + course + docs
  - prints:
    - URLs
    - demo login shortcuts
    - tenant/course IDs

### 3-minute demo script (the story you show)
1. Login as Demo Admin
2. Create tenant + invite learner (copy invite link)
3. Create course “Intro to X”
4. Upload a doc (or click “Load Demo Data”)
5. Switch to Ask page: ask 2 questions and open citations
6. Go to Quiz: take quiz; show score + explanations
7. Show mastery meter and second quiz tier changed
8. Open Admin → Audit log: show events

### How to share (without AWS)
- Public GitHub repo + README demo steps
- Optional: record a 2–3 minute video walkthrough (fast proof)
- Optional: temporary tunnel demo (only with demo data)

---

## 13) Optional AWS deployment module (later)

**Goal:** zero rewrite, only config + Terraform.

### Terraform modules (recommended structure)
- `infra/modules/network`
- `infra/modules/postgres`
- `infra/modules/api-ecs`
- `infra/modules/web-s3-cloudfront`
- `infra/modules/dynamodb`
- `infra/envs/dev`

### AWS deployment story (later)
- Build images → push to registry
- `terraform apply` provisions services
- CI deploy updates ECS task definition
- Smoke tests hit `/health`

**Note:** AWS services cost money if left up. Treat AWS as “spin up for demo → destroy.”

---

## 14) Skills you’ll learn by building this (what you can claim confidently)

### GenAI / RAG
- Chunking + embeddings
- Vector search with pgvector
- Reranking strategies
- Grounded answering + citations
- Evaluation harness + retrieval metrics

### Platform engineering
- Multi-tenant data isolation patterns
- RBAC + invite flows
- Audit logging + event streams
- Provider abstractions (swap model/backend)

### Full-stack delivery
- React UX flows for admin + learning
- API design with contracts and role gating
- Database modeling + migrations

### Cloud readiness (even while local)
- LocalStack + AWS SDK patterns (S3/Dynamo style)
- Terraform module decomposition (when you enable Target B)
- CI/CD, deterministic testing, smoke checks

### “Senior-level” signals
- Spec-first design (endpoints, schema, milestones)
- Guardrails (rate limiting, limits, retries)
- Accessibility checklist (rare among candidates)

---

## 15) What you can demo to a hiring team (talk track)

- “This is a multi-tenant learning platform with RAG + reranking and citations.”
- “It adapts quiz difficulty based on per-user mastery.”
- “I designed it like a platform service: roles, invites, audit logs, rate limits, structured logs.”
- “I included an evaluation harness so retrieval quality is measurable and tracked in CI.”
- “It runs locally with AWS-shaped services (S3/Dynamo) and can be deployed to AWS with Terraform later.”

---

## 16) Execution milestones (build order)

### M1 — Demoable local core
- tenants + memberships + invites + tenant switch UI
- courses + upload + chunk+embed+index
- query endpoint + citations (no rerank yet)

**Acceptance:** ask a question and see citations in UI.

### M2 — Reranking + better grounding
- implement reranking stage
- debug mode shows retrieval stats

**Acceptance:** reranked citations visibly improve relevance.

### M3 — Quizzes + grading + mastery
- generate quiz + grade attempt + store results
- mastery score updates
- difficulty tier selection logic

**Acceptance:** second quiz difficulty changes based on score.

### M4 — Admin polish + audit/events
- members page + role change + audit log viewer
- EventStream writing (optional)

**Acceptance:** audit log shows actions across flows.

### M5 — Eval harness + CI + guardrails
- eval script and dataset
- CI runs tests + eval (mock LLM)
- rate limiting + input limits

**Acceptance:** CI passes and prints eval results.

### M6 (optional) — AWS module
- Terraform + ECS/RDS/S3/Dynamo + deploy pipeline

**Acceptance:** `/health` works on AWS, then destroy infra.

---

## 17) Notes on model providers & cost (Bedrock reality)
- “Bedrock is pay-as-you-go.”
- Local demo should default to mocked/local models so running TutorGraph is $0.
- Provider adapter lets you switch to Bedrock/Azure/OpenAI later without rework.

---

## 18) Repo skeleton (greenfield structure)

Suggested structure (monorepo):
```
tutorgraph/
  apps/
    api/                  # FastAPI service
    web/                  # React app
  packages/
    tenantkit/            # reusable tenancy/authz library
    ragcore/              # retrieval, rerank, citation, eval
  infra/                  # optional Terraform
  data/
    eval_set/
  docker-compose.yml
  Makefile
  README.md
```

---

## 19) “Next time I come back” checklist
When you’re ready to restart this project, do these in order:

1. Scaffold repo with the skeleton above
2. Stand up local stack (compose + Postgres + LocalStack)
3. Implement TenantKit (tenants/memberships/invites/RBAC)
4. Implement course/doc ingest + indexing
5. Implement query + citations
6. Add reranking
7. Add quizzes + mastery
8. Add eval harness + CI
9. (Optional) add Terraform module + AWS deploy

---

**End of spec.**
