# Application Architecture

**Product:** Search Visibility Optimization Platform  
**Source:** PRD §§49–54 (security, NFRs, technical direction, service boundaries, data model)  
**Style:** Modular monolith first. Split services only when scale or team boundaries require it.

This document defines **what we will build with**, plus **free / low-cost / industry-standard** options for each layer. The **recommended production path for this project is the Low-cost column**, because it stays cheap, uses open standards, and can graduate to the Industry-standard column without a rewrite.

---

## 1. Architecture principles (non-negotiable)

1. **One modular monolith + one worker pool** for MVP. Not microservices.
2. **Async-first:** crawls, GSC sync, scoring, AI, reports run on queues.
3. **Workspace isolation:** every tenant table has `workspace_id`; all queries scoped.
4. **AI is not the source of truth:** Raw data → validation → rules → stats → LLM → human approval.
5. **Adapters for every vendor:** GSC, GA, GBP, LLM, object storage, email. Swap without product rewrites.
6. **No silent website writes.** Recommendations only until an explicit approval layer exists.
7. **Observed vs interpreted** labeled in data and UI.
8. Prefer **open source + S3-compatible + Postgres**. Avoid proprietary lock-in in the core domain.

---

## 2. What this project should use (decision lock)

This is the stack to implement. It is **industry-standard technology**, run on **low-cost / free-tier infrastructure**.

| Concern | Choice | Cost posture |
|---|---|---|
| Language | TypeScript (app + workers) | Free |
| Monorepo | pnpm + Turborepo | Free |
| Frontend framework | **Next.js** (App Router) | Free (OSS) |
| UI / design | **Tailwind CSS + shadcn/ui** implementing `design.html` tokens (IBM Plex, forest `#0F6E56`) | Free |
| Charts | Recharts | Free |
| Backend API | **NestJS** (Fastify adapter) — modular monolith | Free |
| Internal RPC (optional) | REST + OpenAPI as the contract; tRPC only inside the web if needed | Free |
| Worker | Same NestJS codebase, `worker` process + **BullMQ** | Free |
| Database | **PostgreSQL 16** via **Drizzle ORM** + migrations | Free OSS; hosted cheap |
| Cache / jobs | **Redis 7** + BullMQ | Free OSS; hosted cheap |
| Auth (users) | **Better Auth** (self-hosted, sessions in Postgres) | Free |
| Auth (Google products) | Google OAuth 2.0 for Search Console / Analytics / GBP | Free |
| File storage | **S3-compatible API** — **Cloudflare R2** (low cost) or MinIO (free self-host) | Free / low |
| Search (later) | Postgres full-text first; OpenSearch only at scale | Free until needed |
| Crawler | HTTP crawler (undici) + **Playwright** for JS-heavy pages | Free OSS |
| AI | Provider adapter: **Google Gemini** (cheap) / OpenAI / Anthropic; local Ollama for dev | Low |
| Email | Resend (low) or SMTP | Low |
| Observability | Pino logs + OpenTelemetry; **Sentry** free tier; Uptime Kuma if self-host | Free / low |
| Hosting (recommended) | **Web:** Vercel or Cloudflare; **API + workers + Redis + crawl:** Hetzner/Fly; **DB:** Neon or Supabase | Low |
| CI | GitHub Actions | Free |
| Infra as code | Docker Compose (dev + cheap VPS); Terraform later for industry tier | Free |

**Do not start with:** Kubernetes, Kafka, Elasticsearch cluster, separate microservice per PRD module, Auth0 enterprise, AWS everything.

---

## 3. Three cost tiers (same architecture, different operators)

The **code architecture does not change**. Only where it runs and which managed product fills each interface.

| Layer | Free (dev / first tenants) | Low cost (recommended launch) | Industry standard (scale / enterprise) |
|---|---|---|---|
| **Frontend** | Next.js self-host on one VPS | Next.js on Vercel or Cloudflare Pages | Next.js on AWS Amplify / CloudFront + ECS |
| **Backend** | NestJS on same VPS | NestJS on Fly.io / Hetzner / Render | NestJS on ECS/EKS or Cloud Run + GKE |
| **Workers** | Same VPS, Docker Compose | Separate small VM or Fly Machines | Autoscaled worker ASG / Cloud Run jobs |
| **Database** | Postgres in Docker; or Neon/Supabase **free tier** | **Neon** or **Supabase Pro** (backups, branching) | RDS PostgreSQL Multi-AZ / Cloud SQL HA |
| **Auth** | Better Auth + Google OAuth | Same (still self-hosted) | Better Auth **or** Cognito / Clerk / WorkOS SSO |
| **File storage** | **MinIO** on VPS | **Cloudflare R2** (near-zero egress) | **Amazon S3** + CloudFront, KMS, Object Lock |
| **Cache / queue** | Redis in Docker | **Upstash Redis** or Redis on Hetzner | ElastiCache + SQS / Cloud Tasks |
| **Secrets** | `.env` + Docker secrets | Doppler / Infisical free-low | AWS Secrets Manager / GCP Secret Manager |
| **CDN** | Cloudflare free | Cloudflare Pro if needed | CloudFront / Fastly |
| **Observability** | Logs to file + Uptime Kuma | Sentry + Axiom/Better Stack | Datadog / Grafana Cloud / CloudWatch |
| **LLM** | Gemini free tier + Ollama locally | Gemini Flash / cheap OpenAI | Multi-provider + VPC, evals, budget controls |
| **Typical monthly** | ~$0–15 (VPS or free tiers) | ~$25–80 at MVP volume | $500+ plus usage |

**Rule:** write to **interfaces** (`ObjectStorage`, `Queue`, `Sql`, `AuthSession`, `LlmProvider`). Swap MinIO → R2 → S3 without touching product code.

---

## 4. System context

```
Browser (Next.js)
        |
        | HTTPS (session cookie)
        v
   NestJS API  ──────────── PostgreSQL
        |                         ^
        | enqueue                 | read/write domain
        v                         |
   Redis + BullMQ ── workers ─────┘
                        |
                        +-- Crawler (HTTP / Playwright)
                        +-- GSC / GA / GBP adapters (OAuth)
                        +-- Scoring + diagnostics (deterministic)
                        +-- LLM adapter (interpretation only)
                        +-- Report renderer
                        |
                        v
                 Object storage (R2 / MinIO / S3)
                 crawl HTML, screenshots, report PDFs
```

Matching PRD §52, as **modules inside one API**, not separate deployables:

```
Web App → API Gateway (Nest)
  Auth | Projects | Integrations | SEO | AI | Tasks | Reporting
                         |
                   Data pipeline jobs
              Crawler | Search ingest | Analytics ingest
```

---

## 5. Frontend architecture

### Framework

- **Next.js App Router + TypeScript**
- Server Components for shells and first paint of dashboards
- Client Components for tables, filters, job polling, charts
- Route groups: `(auth)`, `(app)` with workspace/project in the URL  
  `/[workspaceSlug]/[projectSlug]/...`

### Design system

- Tokens from `design.html` (CSS variables)
- **Tailwind + shadcn/ui** (Radix primitives) for a11y
- IBM Plex Sans / IBM Plex Mono
- Desktop-first dense data; mobile for NBA + task status only
- Charts: Recharts
- Tables: TanStack Table
- Server/client data: **TanStack Query** for jobs, lists, polling

### Frontend structure

```
apps/web
  app/                    # routes
  components/             # DS + product
  features/               # dashboard, audit, nba, tasks...
  lib/api.ts              # typed client from OpenAPI
```

### Frontend quality bar (PRD)

- Six dashboard questions always in the same order
- Every recommendation: Evidence, Reason, Confidence, Action, Measurement
- Partial-data, crawl-fail, rate-limit empty states
- Never headline “247 issues”

---

## 6. Backend architecture

### Shape

- **Modular monolith** in NestJS
- One HTTP process (`api`)
- One or more **worker** processes sharing modules
- Bounded contexts as Nest **modules** (folders), not networks:

| Module | Responsibility |
|---|---|
| Identity | Users, sessions, Better Auth, RBAC |
| Tenancy | Org, workspace, membership, isolation |
| Projects | Website, markets, competitors, settings |
| Integrations | OAuth tokens (encrypted), GSC/GA/GBP clients |
| Pipeline | Job orchestration, completeness flags |
| Crawler | Fetch, parse, snapshots to object storage |
| Audit | Deterministic technical/on-page issues |
| Visibility | GSC rollups, buckets, trends |
| Opportunity | Candidate generation + scoring formula |
| Optimize | Page analysis + grounded LLM copy |
| Workflow | Tasks, owners, status machine |
| Changes | Change log, snapshots |
| Verify | Baselines, windows, experiments, no causality claims |
| Reporting | Monthly narrative from same metrics |
| Assistant | RAG over project data (post-MVP) |

### API style

- REST `/v1/workspaces/:workspaceId/...`
- OpenAPI generated from Nest
- Long work: `202 Accepted` + `jobId`; client polls or websockets later
- Cursor pagination, consistent error envelope
- Idempotency keys on job-creating POSTs

### Domain rules in code, not only in the LLM

- Scoring: `priority = business_value × visibility_opportunity × confidence × feasibility` (coefficients in config)
- Diagnosis: rules engine (JSON/TS rule set)
- LLM receives a **structured evidence payload**; cannot invent GSC rows or crawl issues

---

## 7. Database architecture

**Engine:** PostgreSQL 16  
**Access:** Drizzle ORM (SQL-visible, cheap, good for complex reporting queries)  
**Migrations:** Drizzle Kit, required in CI  

### Tenancy

- `organizations` → `workspaces` → `projects`
- `workspace_id` on every tenant-owned table
- RLS **optional later**; application scoping + tests first; enable Postgres RLS when agency isolation is a sales requirement

### Core entities (PRD §53 + extras)

User, Organization, Workspace, Membership, Project, Website, Page, Keyword, SearchQuery, Competitor, Audit, Issue, Opportunity, Recommendation, Task, Optimization, Change, Experiment, MetricSnapshot, Integration, Report, AIQuery, AIMention, AICitation, Location  

Plus: Evidence, JobRun, AuditLog, ScoreBreakdown (JSONB)

### Data stores by purpose

| Data | Store |
|---|---|
| Relational domain | Postgres |
| Job state / cache / rate-limit counters | Redis |
| HTML snapshots, screenshots, PDFs | Object storage (keys in Postgres) |
| Full crawl corpora at huge scale | Still object storage; do not put HTML blobs in Postgres |

**Backups:** daily (free/low: provider PITR on Neon/Supabase; industry: RDS automated + PITR).

---

## 8. Authentication and authorization

### User authentication (app)

- **Better Auth** (open source, self-hosted)
- Email/password + Google login
- HttpOnly secure cookies, CSRF, session table in Postgres
- Password hashing: Argon2 (library default)

### Integration authentication (Google)

- OAuth 2.0 with **least privilege** scopes
- Tokens encrypted at rest (envelope encryption: data key in DB, master key in env/KMS)
- Never log tokens
- Disconnect + token revoke

### Authorization

MVP roles: **Owner, Admin, Strategist, Collaborator, Client Viewer**  
Permissions checked in a Nest guard: `workspace` membership + role.

### Security (PRD §49)

- TLS everywhere
- Encrypted secrets
- Audit log: login, role change, integration connect, export, delete
- Workspace isolation tests
- Rate limit per workspace (Redis)
- Account/workspace deletion (right to erase)

**Free:** Better Auth + Google Cloud OAuth client  
**Low cost:** same  
**Industry:** add SAML/SSO (WorkOS or Keycloak), SCIM, short-lived AWS IAM for workers

---

## 9. File storage architecture

**Interface:** S3 API (`PutObject`, `GetObject`, presigned URLs).

| Object | Path pattern | Retention |
|---|---|---|
| Crawl HTML | `ws/{id}/proj/{id}/crawls/{run}/pages/{hash}.html.gz` | Policy (e.g. 90 days) |
| Screenshots | `.../screenshots/{pageId}.webp` | 90 days |
| Report PDFs | `.../reports/{reportId}.pdf` | User-driven |
| Change diffs | `.../changes/{changeId}.json` | Long |

**Free:** MinIO  
**Low cost:** Cloudflare R2  
**Industry:** AWS S3 (SSE-KMS, versioning, lifecycle)

Public access: **none**. App issues short-lived signed URLs.

---

## 10. Background processing

**Queue:** BullMQ on Redis  

Job types: `crawl.site`, `gsc.backfill`, `gsc.incremental`, `audit.run`, `opportunities.score`, `page.analyze`, `report.generate`, `verify.snapshot`, `nba.refresh`

- Idempotent job IDs (`projectId + type + date`)
- Retries with backoff; dead-letter + UI error
- Crawl budget and GSC quota tracked per workspace
- Playwright isolated in worker (not in the HTTP API process)

---

## 11. Integrations architecture

Adapter per provider, behind interfaces:

```
SearchConsoleClient
AnalyticsClient
GbpClient
LlmProvider
Mailer
ObjectStorage
```

**MVP:** Google Search Console + first-party crawler  
**Next:** Google Analytics  
**When feasible:** Google Business Profile  
**Later:** CMS (WP/Shopify/Webflow) — approval-gated writes only

All third-party failures: graceful degradation, completeness flags, no fake metrics.

---

## 12. AI architecture

```
Evidence objects (DB)
    → prompt contract (JSON schema)
    → LlmProvider.complete()
    → persist interpretation with confidence + "interpreted" flag
    → never persist as observed fact
```

- Default cheap model for copy; stronger model only for page optimization
- Prompt/version stored on each recommendation
- Cost cap per workspace per day (Redis counter)

---

## 13. Hosting topologies

### Free / almost free

Single **Hetzner CX22** or Oracle free ARM (if available):

- Docker Compose: `web`, `api`, `worker`, `postgres`, `redis`, `minio`
- Cloudflare DNS + TLS (free)
- Limitation: crawls and DB on one box; fine for prototype, not for agencies at scale

### Low cost (recommended launch)

- **Neon** Postgres
- **Upstash** or small Redis on Hetzner
- **R2** files
- **Vercel** for Next.js
- **Hetzner/Fly** for `api` + `worker` (Playwright needs a real VM; Vercel serverless is a poor crawler host)
- Cloudflare in front

This split is the important industry lesson: **edge/serverless for the UI, long-running VM for crawl/workers.**

### Industry standard

- VPC, private Postgres, NAT for crawlers
- S3, ElastiCache, SQS or managed Redis
- WAF, Shield, KMS, CloudTrail
- Multi-az workers, blue/green deploys
- Still the **same Nest modules** — only ops changes

---

## 14. Observability, CI, quality

- Structured JSON logs (Pino) with `workspaceId`, `jobId`, `traceId`
- Metrics: crawl fail, GSC fail, queue lag, rec generation fail, dashboard p95
- Sentry for API + web
- GitHub Actions: typecheck, unit tests, Drizzle migrate dry-run, build
- Contract tests for adapters (recorded GSC fixtures; no live secrets in CI)

---

## 15. Repo layout

```
apps/web          # Next.js
apps/api          # NestJS HTTP
apps/worker       # NestJS/BullMQ (same modules as api)
packages/db       # Drizzle schema
packages/domain   # scoring, rules, types
packages/integrations
packages/ui       # design system
packages/ai       # LLM adapter
packages/storage  # S3/R2/MinIO client
```

---

## 16. Explicitly rejected for MVP

- Microservices per module
- Kubernetes
- Building our own backlink index
- Storing OAuth tokens in plaintext
- Running Playwright on Vercel serverless
- Using the LLM as the audit engine
- MongoDB as primary DB (relational reporting + tenancy fit Postgres)
- Firebase as the core (weak for this domain model)

---

## 17. One-page summary

| Question | Answer |
|---|---|
| Frontend | Next.js + Tailwind + shadcn, tokens from `design.html` |
| Backend | NestJS modular monolith, REST/OpenAPI |
| Workers | BullMQ + Redis, Playwright/HTTP crawler |
| Database | PostgreSQL + Drizzle |
| Auth | Better Auth + Google OAuth; RBAC |
| Files | S3 API (MinIO → R2 → S3) |
| AI | Pluggable LLM behind evidence pipeline |
| Style | Free OSS code, low-cost hosted launch, industry-standard path to AWS/GCP |

Build once. Change hosting as money and volume appear. Do not change the domain architecture.
