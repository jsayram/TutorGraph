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



# USER WORKFLOW

Here’s the **end-to-end workflow** for TutorGraph, in plain terms, from “first time user” to “daily use.”

---

## Workflow (what a user actually does)

### 1) Join or create a space (Tenant / Org)

* **Admin/Teacher** creates a new space like: “AP Biology — Period 3”
* They **invite students** by email (or invite link)
* Students accept and now they’re inside that space
  *(This is how the app keeps different classes/groups separated.)*

---

### 2) Create a course

* Admin creates a course inside the space, like:

  * “Unit 1: Cells”
  * “Chapter 4: Genetics”
* Think of a course like a folder that holds learning material.

---

### 3) Add content (the “brain” of the course)

Admin uploads course content:

* paste notes
* upload a markdown/text file
* (later: PDF)

The app then automatically:

* breaks content into chunks
* stores it
* creates “search fingerprints” (embeddings)
* marks the course as “ready”

---

### 4) Student uses “Ask” mode (Q&A with receipts)

Student goes to **Ask**:

* types a question like: “What’s the difference between mitosis and meiosis?”
* the app searches the course content
* picks the best sections (reranking)
* answers
* shows citations like: “Source: Doc 2, Chunk 7”

Student can then:

* click a citation to see the exact snippet
* thumbs up/down the answer
* mark if it felt grounded or not

---

### 5) Student uses “Quiz” mode (practice + grading)

Student goes to **Quiz**:

* clicks “Start Quiz”
* the app chooses a difficulty level based on their recent scores
* generates questions from the course content (with citations behind the scenes)
* student answers
* app grades + explains

After finishing:

* score is saved
* weak topics are identified
* mastery score updates

---

### 6) Adaptive difficulty kicks in automatically

Next time they take a quiz:

* if they’ve been doing well → harder questions
* if they struggle → easier questions + more review-style questions

So it evolves with the student.

---

### 7) Admin monitoring (optional but “real product”)

Admin can:

* see who joined
* see what content is uploaded and indexed
* view an **audit log** of actions (invites, uploads, role changes)
* view basic performance summaries (optional)

---

## The “daily use” loop (simple version)

1. Upload/refresh notes
2. Ask questions while studying
3. Take quiz
4. Improve mastery
5. Repeat

---
**UI flow diagram** (ASCII) or a **step-by-step demo script** you can follow in a screen recording.
TutorGraph — Detailed UI + Working Architecture Flow (ASCII)
============================================================

Legend:
  UI = screens/components user sees
  API = FastAPI endpoints
  SVC = internal service/module
  DB = Postgres (pgvector)
  OBJ = S3-like object storage (LocalStack S3 now, AWS S3 later)
  EVT = Dynamo-like event stream (LocalStack DynamoDB now, AWS DynamoDB later)
  LLM = model provider (Mock/Local now, Bedrock/Azure/OpenAI later)

Global Request Pattern
----------------------
[React Web] -- Authorization: Bearer <JWT>
           -- X-Tenant-Id: <tenant_id>
           --> [FastAPI API]
           --> (API verifies JWT + tenant membership + role gates)


0) High-level Component Architecture (Local-first, AWS-shaped)
--------------------------------------------------------------
+-------------------+         HTTPS          +---------------------------+
|     User Browser  | <--------------------> |        React Web UI        |
+-------------------+                        +------------+--------------+
                                                         |
                                                         | HTTPS (JWT + X-Tenant-Id)
                                                         v
                                            +------------+--------------+
                                            |         FastAPI API        |
                                            |  - auth middleware         |
                                            |  - tenant enforcement      |
                                            |  - rate limiting           |
                                            |  - structured logging      |
                                            +------+----------+----------+
                                                   |          |
                                   SQL (pgvector)  |          | adapters (same interface)
                                                   |          |
                                                   v          v
                                       +-----------+--+   +---+------------------+
                                       | Postgres +   |   | Providers/Adapters   |
                                       |  pgvector    |   |  StorageProvider     |
                                       | (single source|   |  EventStore         |
                                       |  of truth for |   |  LLMProvider        |
                                       |  v1 core)     |   +---+-----------+-----+
                                       +---------------+       |           |
                                                               |           |
                                                             OBJ         EVT
                                                    (LocalStack S3) (LocalStack DDB)
                                                    (AWS S3 later)  (AWS DDB later)

LLM is also behind adapter:
   LLMProvider -> Mock/Local now  |  Bedrock/Azure/OpenAI later


1) UI Flow (Detailed) + the backend calls behind each click
------------------------------------------------------------

A) Login + Tenant/Course selection
----------------------------------
+-------------------+        +-----------------------+
|  [Landing/Login]  | -----> |  [Tenant Switcher UI] |
+-------------------+        +-----------+-----------+
        |                               |
        | (dev login issues JWT)        | GET /tenants
        v                               v
+-------------------+        +-----------------------+
|  JWT stored client|        | [Course Selector UI]  |
+-------------------+        +-----------+-----------+
                                        |
                                        | GET /courses (scoped by X-Tenant-Id)
                                        v
                               +-----------------------+
                               |     [Dashboard]       |
                               | mastery + recents     |
                               +-----------+-----------+
                                           |
                                           +--> Nav: Ask | Quiz | Content | Admin


B) Content Upload Flow (Admin) — “Ingest pipeline”
--------------------------------------------------
UI:
[Content Page] -> Upload -> See status -> (Optional) Reindex

Under the hood:

(1) Upload document
[React Content UI]
    |
    | POST /courses/{course_id}/documents
    | body: {title, text/markdown}  OR  (file upload)
    v
[FastAPI API]
    |
    | SVC: DocumentService.create_document()
    |   - writes Documents row (status=processing)
    |   - (optional) stores raw doc in OBJ via StorageProvider
    |   - emits "document_uploaded" event to EVT
    v
[DB: documents]

(2) Chunk + embed + index
[FastAPI API] (sync for MVP OR background job later)
    |
    | SVC: IngestService.index_document(doc_id)
    |   - normalize text
    |   - chunk(text, size ~800 chars, overlap ~120)
    |   - embed chunks via LLMProvider.embed()  (mock/local now)
    |   - store chunks + embeddings in DB (pgvector)
    |   - update document status=indexed
    |   - emit "document_indexed" event
    v
[DB: chunks (embedding vector), documents(status=indexed)]

(3) UI refresh
[React Content UI]
    |
    | GET /courses/{course_id}/documents
    v
[FastAPI API] -> [DB]
    |
    v
[Shows list: processing / indexed / failed]

(4) Reindex (Admin)
[React Content UI] -> "Reindex"
    |
    | POST /documents/{doc_id}/reindex
    v
[FastAPI API] -> IngestService re-runs steps above


C) Ask Flow (Learner) — “RAG + reranking + citations”
-----------------------------------------------------
UI:
[Ask Page] -> Ask question -> See answer + citations -> Provide feedback

Under the hood:

(1) Ask question
[React Ask UI]
    |
    | POST /courses/{course_id}/query
    | body: {question, top_n=40, top_k=10, debug?}
    v
[FastAPI API]
    |
    | Middleware:
    |   - verify JWT
    |   - enforce tenant membership (X-Tenant-Id)
    |   - rate limit (per user/IP)
    |   - attach request_id to logs
    |
    | SVC: QueryService.answer_question()
    |   a) store question
    |   b) embed(question) via LLMProvider.embed()
    |   c) vector search in pgvector: top_n chunks within course_id + tenant_id
    |   d) rerank: hybrid score (vector sim + keyword overlap/BM25-ish)
    |   e) select top_k chunks as context
    |   f) generate answer via LLMProvider.generate()
    |      - prompt forces "answer only from context"
    |   g) build citations (doc_id, chunk_id, snippet, score)
    |   h) store answer + citations
    |   i) emit "question_answered" event with retrieval stats
    v
[DB: questions, answers, citations]   +   [EVT: EventStream optional]

(2) Render answer + citations
[React Ask UI]
    |
    | Response:
    |  - answer text
    |  - citations[] (snippets + doc/chunk refs)
    |  - retrieval_stats (top_n/top_k/latency) when debug enabled
    v
[UI shows Answer + expandable Citations Drawer]

(3) Feedback loop
[React Ask UI] -> thumbs up/down + grounded?
    |
    | POST /answers/{answer_id}/feedback
    v
[FastAPI API] -> FeedbackService.store_feedback()
    |
    +--> [DB: feedback]  +--> emit "answer_feedback" event


D) Quiz Flow (Learner) — “Adaptive difficulty loop”
---------------------------------------------------
UI:
[Quiz Page] -> Start -> Answer -> Submit -> Results -> Next quiz auto-adjusts

Under the hood:

(1) Start quiz
[React Quiz UI]
    |
    | POST /courses/{course_id}/quizzes/generate
    v
[FastAPI API]
    |
    | SVC: QuizService.generate_quiz()
    |   a) read mastery(user_id, course_id)
    |   b) choose difficulty tier (1..4) based on mastery thresholds
    |   c) pick source chunks:
    |      - can reuse retrieval to sample relevant chunks
    |   d) generate questions via LLMProvider.generate()
    |      - "generate questions from these chunks"
    |      - store questions + (optional) citations per question
    |   e) store quiz record (difficulty_tier)
    |   f) emit "quiz_generated" event
    v
[DB: quizzes, quiz_questions]

(2) Take quiz
[React Quiz UI] (client-side only until submit)

(3) Submit attempt + grading
[React Quiz UI] -> Submit
    |
    | POST /quizzes/{quiz_id}/attempts
    | body: {answers[]}
    v
[FastAPI API]
    |
    | SVC: QuizService.grade_attempt()
    |   a) compute score (MCQ exact match; short answer rubric later)
    |   b) generate explanations (optional) with citations
    |   c) store attempt + score
    |   d) update mastery:
    |      mastery_score = weighted avg of last N attempts
    |   e) emit "quiz_submitted" + "mastery_updated"
    v
[DB: quiz_attempts, mastery]  +  [EVT optional]

(4) Show results
[React Quiz UI] -> GET /courses/{course_id}/performance
    |
    v
[FastAPI API] -> [DB] -> mastery meter + recent attempts


2) Internal Modules (what the repo is “really” made of)
-------------------------------------------------------
API Layer:
  - auth middleware (JWT verify)
  - tenant middleware (X-Tenant-Id -> membership check)
  - rate limiter
  - request_id + structured logs

Services:
  - TenantService (tenants/memberships/roles/invites)
  - CourseService (courses)
  - DocumentService (documents + status)
  - IngestService (chunking + embeddings + chunk storage)
  - QueryService (RAG + rerank + citations)
  - QuizService (generate + grade + mastery)
  - Audit/EventService (audit_log rows + optional EVT stream)

Shared “providers” (adapters you can swap later):
  - LLMProvider      : Mock/Local  <-> Bedrock/Azure/OpenAI
  - StorageProvider  : LocalStack S3 <-> AWS S3
  - EventStore       : LocalStack DDB <-> AWS DynamoDB
  - VectorStore      : Postgres pgvector <-> RDS pgvector


3) “AWS-shaped” mapping (so the optional deploy is not painful)
---------------------------------------------------------------
Local Target A (Docker Compose):
  React Web  -> container
  FastAPI    -> container
  Postgres   -> container + pgvector
  LocalStack -> S3 + DynamoDB (optional)
  LLM        -> mocked/local

AWS Target B (Terraform module later):
  React Web  -> S3 + CloudFront
  FastAPI    -> ECS Fargate + ALB
  Postgres   -> RDS Postgres + pgvector
  Events     -> DynamoDB
  Logs       -> CloudWatch
  Secrets    -> SSM / Secrets Manager

Key point: same API + same services; only provider endpoints change.


## USER WORKFLOW 
TutorGraph — UI Workflow Flowchart (ASCII)
==========================================

                         +------------------+
                         |     Landing      |
                         +------------------+
                                  |
                                  v
                         +------------------+
                         |      Login       |
                         | (Admin / Learner)|
                         +------------------+
                                  |
                                  v
                         +------------------+
                         |  Tenant Switch   |
                         | (Org / Class)    |
                         +------------------+
                                  |
                                  v
                         +------------------+
                         |  Course Select   |
                         +------------------+
                                  |
                                  v
                         +------------------+
                         |    Dashboard     |
                         | mastery + recents|
                         +----+---------+---+
                              |         |
               Learner (Student)       Admin (Teacher/Admin)
                              |         |
                +-------------+         +------------------+
                |                                |
                v                                v
        +------------------+            +------------------+
        |       Ask        |            |      Content     |
        | Q -> Answer      |            | Upload / Paste   |
        | + Citations      |            | Docs Status      |
        +--------+---------+            +--------+---------+
                 |                               |
                 v                               v
        +------------------+            +------------------+
        |  Open Citations  |            |  Reindex (opt.)  |
        |  View Snippets   |            +--------+---------+
        +--------+---------+                     |
                 |                               v
                 v                       +------------------+
        +------------------+             |     Dashboard     |
        | Feedback         |             +------------------+
        | Helpful/Grounded |
        +--------+---------+
                 |
                 v
        +------------------+
        |     Dashboard    |
        +--------+---------+
                 |
                 v
        +------------------+
        |       Quiz       |
        | Start Quiz       |
        +--------+---------+
                 |
                 v
        +------------------+
        |   Take Quiz      |
        | Answer + Submit  |
        +--------+---------+
                 |
                 v
        +------------------+
        |   Quiz Results   |
        | Score + Weakness |
        | Mastery Updates  |
        +--------+---------+
                 |
                 v
        +------------------+
        | Next Quiz Tier   |
        | auto adjusts     |
        +--------+---------+
                 |
                 v
             (back to Quiz)


Admin extras (from Dashboard)
-----------------------------
                 +------------------+
                 |      Admin       |
                 | Members/Invites  |
                 | Roles/Audit Log  |
                 +--------+---------+
                          |
                          v
                     +----------+
                     |Dashboard |
                     +----------+


------------------------
------------------------
# TutorGraph — FULL UI STATE FLOW (ALL STATES) (ASCII)
====================================================

Legend:
  [STATE]          = screen/page/state
  (global)         = can happen anywhere
  -->              = navigation/action
  ---error/alt---> = alternate path
  (Admin only) / (Learner) / (All)

--------------------------------------------------------------------------------
0) GLOBAL STATES (can occur from ANY screen)
--------------------------------------------------------------------------------
(global) [Loading Spinner]  -> returns to prior state when done
(global) [API Error Banner] -> user can Retry -> prior state
(global) [Offline / Network Error] -> Retry -> prior state
(global) [Rate Limited (429)] -> Wait/Retry -> prior state
(global) [Session Expired] -> redirects to [Login]
(global) [Forbidden (403)] -> [No Access]
(global) [Not Found (404)] -> [Not Found]
(global) [Maintenance Mode] -> [Maintenance Page]

[No Access]
  -> Switch Tenant
  -> Back to [Tenant Switcher]

[Not Found]
  -> Back to [Dashboard] (if possible) OR [Tenant Switcher]

[Maintenance Page]
  -> Retry -> [Landing] or last known state


--------------------------------------------------------------------------------
1) ENTRY / AUTH / ACCOUNT STATES
--------------------------------------------------------------------------------
[Landing]
  -> Sign In
  -> (optional) Sign Up (if enabled)
  -> (optional) Accept Invite Link (if token present)
      |
      v
[Login] (All)
  -> Success -> [Tenant Switcher]
  ---error---> [Login Error] -> Retry -> [Login]

[Login Error]
  -> Retry -> [Login]
  -> Back -> [Landing]

[Invite Link Opened] (token route)
  -> If not logged in -> [Login] -> returns to [Invite Acceptance]
  -> If logged in -> [Invite Acceptance]

[Invite Acceptance]
  -> Accept Invite -> [Invite Accepted] -> [Tenant Switcher]
  ---expired---> [Invite Expired]
  ---invalid---> [Invite Invalid]
  ---already member---> [Already Joined] -> [Tenant Switcher]

[Invite Expired]
  -> Back -> [Landing] / [Login]
  -> (Admin must resend invite)

[Invite Invalid]
  -> Back -> [Landing] / [Login]

[Already Joined]
  -> Continue -> [Tenant Switcher]

[Profile] (All)  (global via top bar)
  -> Sign Out -> [Landing]
  -> (optional) Account Settings -> [Account Settings]

[Account Settings] (All)
  -> Back -> prior page


--------------------------------------------------------------------------------
2) TENANT (ORG) STATES
--------------------------------------------------------------------------------
[Tenant Switcher] (All)
  -> Select Tenant -> [Course Selector]
  -> Create Tenant (Admin/Owner) -> [Create Tenant]
  ---no tenants yet---> [Empty Tenants State]

[Empty Tenants State]
  -> Create Tenant -> [Create Tenant]
  -> Sign Out -> [Landing]

[Create Tenant] (Admin/Owner)
  -> Create -> [Tenant Created] -> [Course Selector]
  ---error---> [Create Tenant Error] -> Retry -> [Create Tenant]

[Create Tenant Error]
  -> Retry -> [Create Tenant]
  -> Back -> [Tenant Switcher]

[Tenant Created]
  -> Continue -> [Course Selector]


--------------------------------------------------------------------------------
3) COURSE STATES
--------------------------------------------------------------------------------
[Course Selector] (All)
  -> Select Course -> [Dashboard]
  -> Create Course (Admin) -> [Create Course]
  ---no courses yet---> [Empty Courses State]

[Empty Courses State]
  -> Create Course (Admin) -> [Create Course]
  -> Switch Tenant -> [Tenant Switcher]

[Create Course] (Admin)
  -> Create -> [Course Created] -> [Dashboard]
  ---error---> [Create Course Error] -> Retry -> [Create Course]

[Create Course Error]
  -> Retry -> [Create Course]
  -> Back -> [Course Selector]

[Course Created]
  -> Continue -> [Dashboard]


--------------------------------------------------------------------------------
4) DASHBOARD STATES (HOME)
--------------------------------------------------------------------------------
[Dashboard] (All)
  -> Nav: Ask (Learner/Admin) -> [Ask (Idle)]
  -> Nav: Quiz (Learner/Admin) -> [Quiz Home]
  -> Nav: Content (Admin) -> [Content Home]
  -> Nav: Admin (Admin/Owner) -> [Admin Home]
  -> Switch Tenant (top bar) -> [Tenant Switcher]
  -> Switch Course (top bar) -> [Course Selector]

  ---if course has NO content--->
      [Empty Course Content State]
         -> (Admin) Go Upload -> [Content Home]
         -> (Learner) View Message -> back to [Dashboard]

[Empty Course Content State]
  -> (Admin) Upload Content -> [Content Home]
  -> Switch Course -> [Course Selector]


--------------------------------------------------------------------------------
5) CONTENT STATES (ADMIN)
--------------------------------------------------------------------------------
[Content Home] (Admin)
  -> Upload/Paste -> [Upload Form]
  -> View Docs List -> [Documents List]
  -> Reindex Doc -> [Reindex Confirm] -> [Reindexing]
  -> Delete/Archive Doc (optional) -> [Delete Confirm] -> [Doc Deleted]
  -> Back -> [Dashboard]

[Upload Form] (Admin)
  -> Submit Upload -> [Uploading]
  -> Cancel -> [Content Home]

[Uploading]
  -> Success -> [Indexing]
  ---error---> [Upload Failed]

[Upload Failed]
  -> Retry -> [Uploading]
  -> Back -> [Upload Form]

[Indexing]
  -> Success -> [Indexed Success]
  ---error---> [Indexing Failed]

[Indexing Failed]
  -> Retry Index -> [Indexing]
  -> Back -> [Documents List]

[Indexed Success]
  -> View Docs -> [Documents List]
  -> Upload Another -> [Upload Form]
  -> Back -> [Content Home]

[Documents List] (Admin)
  -> View Doc Details -> [Doc Detail]
  -> Reindex -> [Reindex Confirm]
  -> Back -> [Content Home]

[Doc Detail] (Admin)
  -> Back -> [Documents List]

[Reindex Confirm]
  -> Confirm -> [Reindexing]
  -> Cancel -> [Documents List]

[Reindexing]
  -> Success -> [Indexed Success]
  ---error---> [Indexing Failed]

[Delete Confirm] (optional)
  -> Confirm -> [Doc Deleted]
  -> Cancel -> [Documents List]

[Doc Deleted] (optional)
  -> Back -> [Documents List]


--------------------------------------------------------------------------------
6) ASK (Q&A) STATES (LEARNER + ADMIN)
--------------------------------------------------------------------------------
[Ask (Idle)] (Learner/Admin)
  -> Type Question -> Submit -> [Answer Generating]
  -> Back -> [Dashboard]

[Answer Generating]
  -> Success -> [Answer Shown]
  ---no relevant content---> [No Answer Found]
  ---timeout/error---> [Answer Failed]

[Answer Shown]
  -> Expand Citations -> [Citations Expanded]
  -> Submit Feedback -> [Feedback Submitted]
  -> Ask Another -> [Ask (Idle)]
  -> Back -> [Dashboard]

[Citations Expanded]
  -> Open Citation -> [Citation Detail Drawer]
  -> Collapse -> [Answer Shown]

[Citation Detail Drawer]
  -> Close -> [Citations Expanded]

[Feedback Submitted]
  -> Back to Answer -> [Answer Shown]

[No Answer Found]
  -> Suggest upload more content (message)
  -> Ask Another -> [Ask (Idle)]
  -> Back -> [Dashboard]

[Answer Failed]
  -> Retry -> [Answer Generating]
  -> Back -> [Ask (Idle)]


--------------------------------------------------------------------------------
7) QUIZ STATES (LEARNER + ADMIN)
--------------------------------------------------------------------------------
[Quiz Home] (Learner/Admin)
  -> Start Quiz -> [Quiz Generating]
  -> View Past Attempts -> [Attempts List]
  -> Back -> [Dashboard]

  ---if no content indexed--->
      [Quiz Unavailable (No Content)]
         -> Back -> [Dashboard]
         -> (Admin) Upload Content -> [Content Home]

[Quiz Unavailable (No Content)]
  -> Back -> [Dashboard]
  -> (Admin) Upload -> [Content Home]

[Quiz Generating]
  -> Success -> [Quiz Taking]
  ---error---> [Quiz Generate Failed]

[Quiz Generate Failed]
  -> Retry -> [Quiz Generating]
  -> Back -> [Quiz Home]

[Quiz Taking]
  -> Submit -> [Submit Confirm] (optional) -> [Grading]
  -> Cancel -> [Quiz Home] (optional)

[Submit Confirm] (optional)
  -> Confirm -> [Grading]
  -> Cancel -> [Quiz Taking]

[Grading]
  -> Success -> [Quiz Results]
  ---error---> [Grading Failed]

[Grading Failed]
  -> Retry -> [Grading]
  -> Back -> [Quiz Home]

[Quiz Results]
  -> Show Score + Explanations + Weak Topics + Updated Mastery
  -> Take Another Quiz -> [Quiz Home] (difficulty auto-adjusted)
  -> Back -> [Dashboard]

[Attempts List]
  -> Open Attempt -> [Attempt Detail]
  -> Back -> [Quiz Home]

[Attempt Detail]
  -> Back -> [Attempts List]


--------------------------------------------------------------------------------
8) ADMIN PANEL STATES (OWNER/ADMIN)
--------------------------------------------------------------------------------
[Admin Home] (Owner/Admin)
  -> Members -> [Members List]
  -> Invites -> [Invites List]
  -> Send Invite -> [Invite Form]
  -> SSO Settings -> [SSO Settings] (config UI in v1)
  -> Audit Log -> [Audit Log Viewer]
  -> Back -> [Dashboard]

[Members List]
  -> Change Role -> [Role Update] -> [Role Updated]
  -> Remove Member (optional) -> [Remove Confirm] -> [Member Removed]
  -> Back -> [Admin Home]

[Role Update]
  -> Save -> [Role Updated]
  ---error---> [Role Update Failed]

[Role Updated]
  -> Back -> [Members List]

[Role Update Failed]
  -> Retry -> [Role Update]
  -> Back -> [Members List]

[Invite Form]
  -> Send -> [Invite Sending] -> [Invite Sent]
  ---error---> [Invite Failed]
  -> Back -> [Admin Home]

[Invite Sending]
  -> Success -> [Invite Sent]
  ---error---> [Invite Failed]

[Invite Sent]
  -> View Invites -> [Invites List]
  -> Copy Invite Link (local demo)
  -> Back -> [Admin Home]

[Invite Failed]
  -> Retry -> [Invite Sending]
  -> Back -> [Invite Form]

[Invites List]
  -> View Invite -> [Invite Detail]
  -> Resend (optional) -> [Invite Sending]
  -> Revoke (optional) -> [Revoke Confirm] -> [Invite Revoked]
  -> Back -> [Admin Home]

[Invite Detail]
  -> Back -> [Invites List]

[Invite Revoked] (optional)
  -> Back -> [Invites List]

[SSO Settings] (v1 config only)
  -> Save Config -> [SSO Saved]
  ---error---> [SSO Save Failed]
  -> Back -> [Admin Home]

[SSO Saved]
  -> Back -> [SSO Settings]

[SSO Save Failed]
  -> Retry -> [SSO Settings]
  -> Back -> [Admin Home]

[Audit Log Viewer]
  -> Filter/Search -> stays here
  -> Back -> [Admin Home]


--------------------------------------------------------------------------------
9) NAVIGATION SUMMARY (what’s always available)
--------------------------------------------------------------------------------
Top Bar (All, except unauth):
  - Tenant Switcher
  - Course Selector
  - Profile (Sign out)

Left Nav (role-gated):
  - Dashboard (All)
  - Ask (All in tenant)
  - Quiz (All in tenant)
  - Content (Admin only)
  - Admin (Owner/Admin only)


-----

------


# TutorGraph — Background Job States (ASCII)
==========================================

Goal:
  Make long-running work (indexing, quiz gen, etc.) resilient + observable.
  UI shows status, jobs can retry, failures are inspectable.

Legend:
  [JOB STATE]      = background job status
  ->              = normal transition
  ---error--->    = failure transition
  (UI)            = what the user sees

Core Background Job Types
-------------------------
  - DocumentIndexJob     (chunk + embed + store vectors)
  - ReindexJob           (rebuild vectors for a doc)
  - QuizGenerateJob      (generate quiz questions)
  - QuizGradeJob         (grade + store results)  (optional async)
  - EvalRunJob           (offline eval harness)   (optional)
  - CleanupJob           (delete doc + cascade)   (optional)

--------------------------------------------------------------------------------
1) Document Indexing Job State Machine
--------------------------------------------------------------------------------

(UI action) Admin uploads doc or clicks "Reindex"
     |
     v
[CREATED]
  - Job record created with (tenant_id, course_id, doc_id)
  - document.status = "processing"
  |
  v
[QUEUED]
  - job is waiting for a worker
  - (UI) "Queued" badge in documents list
  |
  v
[CLAIMED]
  - a worker locks the job (lease/heartbeat begins)
  - (UI) still shows "Processing"
  |
  v
[RUNNING:CHUNKING]
  - split text into chunks
  - (UI) "Indexing: chunking..."
  |
  v
[RUNNING:EMBEDDING]
  - generate embeddings (batched)
  - (UI) "Indexing: embedding..."
  |
  v
[RUNNING:UPDATING_INDEX]
  - write chunks + vectors to pgvector
  - (UI) "Indexing: saving..."
  |
  v
[SUCCEEDED]
  - document.status = "indexed"
  - job.completed_at set
  - (UI) "Indexed" badge + timestamp
  |
  v
[ARCHIVED]
  - optional: job moved out of hot table / compacted


Failure + retry path:
---------------------
[RUNNING:*]
  ---error--->
[FAILED]
  - store: error_code, error_message, stack trace ref, attempt_count
  - document.status = "failed"
  - (UI) "Failed" badge + "Retry" button
  |
  +--> if retryable AND attempt_count < max:
          |
          v
        [RETRY_SCHEDULED]
          - next_run_at set (exponential backoff: 1m, 5m, 15m, 1h...)
          - (UI) "Retry scheduled"
          |
          v
        [QUEUED]  (when next_run_at arrives)
  |
  +--> else:
        [DEAD_LETTERED]
          - job will not retry automatically
          - requires manual retry from UI/admin
          - (UI) "Needs attention" + "View error"


Timeout / worker crash path:
----------------------------
[CLAIMED] or [RUNNING:*]
  ---heartbeat missing / lease expired--->
[STALLED]
  - (UI) "Stalled" badge (optional)
  |
  v
[RETRY_SCHEDULED] -> [QUEUED]  (picked up by another worker)

Cancellation path:
------------------
(UI) Admin clicks "Cancel indexing" (optional)
  -> [CANCEL_REQUESTED]
  -> worker sees flag
  -> [CANCELLED]
     - document.status = "cancelled" (or revert to previous state)
     - (UI) "Cancelled"


--------------------------------------------------------------------------------
2) Quiz Generation Job State Machine (similar, smaller)
--------------------------------------------------------------------------------

(UI) Learner clicks "Start Quiz"
     |
     v
[CREATED] -> [QUEUED] -> [CLAIMED] -> [RUNNING:GENERATING] -> [SUCCEEDED]
   |                                                   |
   ---error--------------------------------------------->
                [FAILED] -> [RETRY_SCHEDULED] -> [QUEUED]
                         -> [DEAD_LETTERED] (manual retry)

(UI states)
  - Queued: "Preparing your quiz..."
  - Running: "Generating questions..."
  - Failed: "Couldn’t generate quiz. Retry."


--------------------------------------------------------------------------------
3) Rerank / Query Jobs (usually NOT async, but can be)
--------------------------------------------------------------------------------
Most RAG queries stay synchronous for UX.
If you async them, states look like:

(UI) Ask question
  -> [REQUEST_ACCEPTED] (202 + job_id)
  -> [QUEUED] -> [RUNNING:RETRIEVE] -> [RUNNING:RERANK] -> [RUNNING:ANSWER]
  -> [SUCCEEDED] (answer available)
  -> [FAILED]/[RETRY_SCHEDULED]/[DEAD_LETTERED]

(UI) polling states
  - "Searching your course..."
  - "Checking best sources..."
  - "Writing answer..."


--------------------------------------------------------------------------------
4) How the UI should reflect job states (simple mapping)
--------------------------------------------------------------------------------

Documents list badge:
  QUEUED            -> "Queued"
  RUNNING:*         -> "Indexing"
  SUCCEEDED         -> "Indexed"
  FAILED            -> "Failed" + Retry button
  RETRY_SCHEDULED   -> "Retry scheduled"
  STALLED           -> "Stalled"
  DEAD_LETTERED     -> "Needs attention"
  CANCELLED         -> "Cancelled"

Doc detail panel shows:
  - status
  - last updated
  - attempt_count
  - last error (if any)
  - next retry time (if scheduled)
  - actions: Retry / Cancel / Reindex


--------------------------------------------------------------------------------
5) Minimal implementation strategy (Local-first)
--------------------------------------------------------------------------------
Option A (simplest): Postgres-backed job queue
  - jobs table with status + next_run_at
  - worker process polls: WHERE status in (QUEUED, RETRY_SCHEDULED and due)
  - claim with UPDATE ... WHERE status=QUEUED RETURNING *
  - heartbeat column updated every N seconds
  - stalled detection: now - heartbeat > threshold

Option B (AWS-shaped later):
  - SQS for queue + ECS worker
  - DynamoDB for job status OR Postgres jobs table still fine
  - CloudWatch for logs/metrics

Both keep the same state model above.

# ----------------------------