# Implementation Plan

**Product:** Search Visibility Optimization Platform  
**Source:** PRD For Search Visibility Optimization Platform (draft, MVP definition)  
**ICP:** SEO/marketing agencies first, then in-house growth teams  
**Hypothesis to prove:** Users will use a system that identifies the most important search visibility opportunities and turns them into prioritized, measurable work.

This plan covers the **entire PRD**, ordered so design and architecture lock before feature work. Later phases stay in scope but do not block MVP.

---

## How to use this plan

- **Do not skip Phase 0–2.** UI, data contracts, and job architecture constrain every module.
- **MVP is Phases 0–8.** That is PRD §42 Must Have plus security/NFR foundations.
- **Phases 9–13** map to PRD Phase 2–4 (Search Visibility OS, Execution, Autonomous).
- Each phase has: goal, decisions to lock, deliverables, exit criteria.
- Signature experience to protect: **Next Best Action** plus **evidence → reason → confidence → action → measurement**.

---

## Recommended stack (lock in Phase 1)

| Layer | Choice | Why |
|---|---|---|
| Frontend | Next.js (App Router) + TypeScript | Matches PRD; SSR/RSC for dashboards; one app for web |
| UI kit | Tokens + shadcn/ui + Tailwind | Fast, consistent, accessible; not a custom DS from scratch |
| Charts | Recharts or Visx | Visibility trends, ranking distribution |
| Backend | Modular monolith: NestJS **or** Next.js Route Handlers + a dedicated worker process | PRD says do not start as microservices |
| API | REST + OpenAPI; tRPC optional inside the monolith | External integrations stay REST |
| DB | PostgreSQL + Prisma or Drizzle | Core entities in §53 |
| Cache / queues | Redis + BullMQ | Crawls, GSC sync, analysis, reports |
| Files | S3-compatible (crawl HTML, screenshots, report PDFs) | |
| Auth | Auth.js / Clerk / custom + OAuth for GSC/GA/GBP | Workspace RBAC required |
| AI | Provider-agnostic adapter (OpenAI, Anthropic, Gemini) | PRD §51, §37 |
| Crawl | Playwright/Chromium worker + HTTP crawler (robots, sitemap, links) | Technical + on-page |
| Observability | OpenTelemetry + structured logs + Sentry | §50 |
| Hosting | Vercel (web) + Fly/Render/AWS (API + workers) **or** single AWS/GCP | Split web vs workers early |

**Hard rule:** one deployable app + one worker pool until a real scale or team boundary appears.

---

# Phase 0 — Research, UX validation, information architecture

**Goal:** Validate H1–H4 with prototypes before building engines. Aligns with PRD Phase 0 and §63.

### Work

1. Primary interviews: agency SEO strategist, in-house marketer (5–8 each).
2. Competitive teardown: Ahrefs, Semrush, Surfer, Clearscope, Profound/Peec, Local Falcon — map gaps to workflow, not feature count.
3. Clickable prototypes (Figma) for four experiments:
   - Next Best Action
   - Page Optimization recommendation card
   - Change → Outcome
   - Agency command center (even if Agency Mode is post-MVP)
4. Map current agency workflow: tools, handoffs, spreadsheets.
5. Define **information architecture** and navigation for the product loop: Discover → Diagnose → Prioritize → Act → Verify → Monitor → Learn.

### Design artifacts (pre–design-system)

- User journeys for US-001 through US-007
- Screen inventory (auth, onboarding, project dashboard, audit, visibility, opportunities, page optimizer, action center, change log, verification, reports)
- Content/tone rules: no ranking guarantees; Observed vs Interpreted labels

### Exit criteria

- NBA prototype scores high on comprehension, usefulness, trust, willingness to act
- Copy rules for evidence/confidence frozen
- Screen list frozen enough to design a component inventory

---

# Phase 1 — Design system and product UI language

**Goal:** A design system that can express **data-heavy SEO workflows**, not a generic SaaS kit. This phase starts **before** feature implementation.

## 1.1 Brand and product language

- Name, logo, dark/light (agencies live in dashboards all day — default **light with dense data**, optional dark)
- Voice: specific, evidence-first, never hype (“will rank #1”)
- Iconography for: search, AI search, local, technical, content, task, experiment

## 1.2 Design tokens

- Color: semantic statuses (Critical / High / Medium / Low), health (Healthy / Attention / Critical), ranking movement (up/down/flat), confidence, branded vs non-branded
- Type scale: dashboard density vs report/export readability
- Spacing, radius, elevation, motion (subtle; data tables over animation)
- Breakpoints: desktop-first (primary ICP), tablet; mobile for **status + NBA + approve task**, not full crawl tables

## 1.3 Core components (build these first)

| Component | Used by |
|---|---|
| App shell (workspace switcher, project switcher, nav) | Everywhere |
| Data table + filters + saved views | Keywords, issues, tasks, pages |
| Metric card + sparkline + delta | Dashboards |
| Evidence panel (source, date, sample size) | All recommendations |
| Recommendation card (problem, why, action, effort, confidence) | Optimizer, NBA |
| Next Best Action hero | Home, project home |
| Severity badge + issue row | Audit |
| Scoring breakdown (Business × Opportunity × Confidence × Feasibility) | Opportunities |
| Task kanban / list (Backlog → Completed) | Action Center |
| Change log timeline | Changes, verification |
| Before/after metric comparison | Verification, experiments |
| Empty, loading, partial-data, error, rate-limit states | NFRs |
| Integration connection card (OAuth status) | Setup |
| Trust labels: Observed evidence vs Model interpretation | AI + diagnostics |

## 1.4 Patterns (product UX, not just UI)

- **Never dead-end on “247 issues.”** Always: category → priority → next action → create task.
- Dashboard answers six questions (§32) in that order.
- Every rec shows: Evidence, Reason, Confidence, Action, Measurement (§38).
- Partial data: show what you have; do not invent GSC/crawl completeness.

## 1.5 Accessibility and viz

- WCAG AA, keyboard tables, color not sole signal for rank movement
- Chart defaults: impressions/clicks/position with annotation for change events

## 1.6 Design-to-code

- Figma variables ↔ CSS tokens
- Storybook (or Ladle) for DS components
- Page templates: Onboarding, Project Home, Audit, Visibility, Page Detail, Action Center, Report

### Exit criteria

- Token set and component library in Figma **and** code stubs in the app
- Templates for NBA, recommendation, verification, and dashboard signed off
- Copy system for confidence and non-causality

---

# Phase 2 — Architectural decisions, tenancy, and platform skeleton

**Goal:** Lock the modular monolith, tenancy, jobs, and AI layer so every module plugs in the same way.

## 2.1 Decisions to freeze

1. **Modular monolith** with bounded contexts (not microservices):
   - Identity & Access
   - Workspaces & Projects
   - Integrations
   - Data Pipeline (ingest)
   - Crawler
   - Visibility Intelligence
   - Diagnostic Engine
   - Opportunity & Priority Engine
   - Optimization Engine
   - Action & Workflow
   - Verification & Experiments
   - Reporting
   - AI Assistant (RAG over project data)
2. **Multi-tenancy:** `organization` → `workspace` → `project`. All queries scoped by workspace. Row-level isolation in Postgres (`workspace_id` on every tenant table).
3. **RBAC (MVP):** Owner, Admin, Strategist, Collaborator, Client Viewer. Agency-grade roles can extend later.
4. **Async-first:** anything > few hundred ms is a job (crawl, GSC sync, page analysis, scoring, reports).
5. **Event log:** domain events (`PageCrawled`, `GscSynced`, `OpportunityScored`, `ChangeRecorded`) feed scoring, NBA, verification.
6. **AI pipeline (mandatory):** Raw data → Validation → Deterministic rules → Statistical signals → LLM interpretation → Recommendation → Human approval. LLM never writes scores without evidence objects.
7. **Integration adapters:** interface `SearchConsoleClient`, `AnalyticsClient`, `GbpClient`, `LlmProvider`. Swap without product rewrites.
8. **No silent site writes.** Implementation layer is recommendation-only in V1.
9. **Idempotent jobs** + provider rate-limit budgets.
10. **Data retention & deletion:** workspace wipe, GSC cache TTL, crawl snapshot retention policy.

## 2.2 Repo and app structure

```
apps/web          # Next.js UI
apps/worker       # BullMQ processors
packages/db       # schema, migrations
packages/domain   # scoring, diagnosis rules, types
packages/integrations
packages/ui       # design system
packages/ai       # LLM adapter + grounded prompt contracts
```

Monorepo (pnpm + Turborepo) is the default.

## 2.3 Data model (implement schema now, fill later)

Implement tables for §53 even if some modules are empty:

User, Organization, Workspace, Membership, Project, Website, Page, Keyword, SearchQuery, Competitor, Audit, Issue, Opportunity, Recommendation, Task, Optimization, Change, Experiment, MetricSnapshot, Integration, Report, AIQuery, AIMention, AICitation, Location.

Plus:

- `Evidence` (polymorphic: gsc_row, crawl_field, competitor_url, llm_note)
- `JobRun` (type, status, error, metrics)
- `AuditLog` (security)
- `ScoreBreakdown` JSON on Opportunity/Recommendation

Recommendation record must match §54.

## 2.4 API conventions

- `/v1/workspaces/:id/...`
- Cursor pagination, consistent errors
- Job resources: `POST /audits` → `202` + `job_id`
- OpenAPI generated from code

## 2.5 Security baseline (from day one, §49)

- TLS, encryption at rest for tokens (KMS or app-level envelope)
- OAuth tokens in vault table, never logs
- Workspace isolation tests
- Rate limits per workspace
- Audit logs for auth, integrations, permission changes, data export/delete

## 2.6 Observability skeleton

- Trace IDs on requests and jobs
- Dashboards: crawl fail, GSC fail, queue lag, recommendation gen fail, p95 dashboard API

### Exit criteria

- Auth + empty workspace/project CRUD
- Worker hello-world job
- Schema migrated
- Storybook DS in the same repo
- Security checklist documented

---

# Phase 3 — Account, workspace, project onboarding

**Maps to:** Module 1, US-001, activation steps 1–2.

### Features

- Sign up / login
- Create workspace
- Create project: URL (validated), target country, locations, search engines, business category, products/services, audience, competitors (names + domains)
- Project settings editable
- Connection wizard: Website → GSC (required for MVP value) → optional later GA

### UX

- Onboarding checklist matching the MVP journey (§44)
- Block “full dashboard” until first crawl **or** first GSC sync has *some* data; show progress, not a blank vanity dashboard

### Exit criteria

- US-001 acceptance criteria
- Invalid URL, unreachable host, and permission-denied GSC states handled

---

# Phase 4 — Integrations and data pipeline (GSC first)

**Maps to:** §33 MVP integrations; Search data Must Have.

## 4.1 Google Search Console (Must)

- OAuth, property picker, least-privilege
- Ingest: queries, pages, countries, devices, dates
- Metrics: impressions, clicks, CTR, position
- Store daily grain + rollups (7/28/90)
- Sync jobs: initial backfill (16 months if available) + daily incremental
- Handle quota, 403, disconnected property

## 4.2 Website crawler (Must)

- Respect robots.txt, sitemap discovery, crawl budget per plan
- Capture: status, canonical, indexability, title, meta, H1–H3, links in/out, redirects, hreflang, basic structured data, image alts, word count, content extract
- Page speed: **indicators only** in MVP (TTFB, LCP from lab crawl or CrUX later — do not block on full Lighthouse fleet)
- Store HTML snapshot hash in object storage
- Classify page type (heuristic): home, service, blog, location, other

## 4.3 Google Analytics (Should, can slip to Phase 9 if time-boxed)

- Sessions, landing pages, conversions, engagement
- Needed for “business outcomes” but **not** required to prove NBA + change tracking if GSC exists

## 4.4 Pipeline quality

- Data validation layer: nulls, date gaps, property mismatch
- `dataset_completeness` on project (crawl %, GSC days)

### Exit criteria

- Project has pages + GSC query/page rows
- Failures are visible in UI and ops
- Re-sync and re-crawl are user-triggerable with rate limits

---

# Phase 5 — Website audit (Diagnostic Engine, technical slice)

**Maps to:** Module 2, US-002.

### Deterministic checks (rules engine, not LLM)

HTTPS, broken links, crawlability, indexability, robots, sitemap, canonicals, redirects, duplicates, missing/duplicate titles, missing meta, heading structure, image alts, mobile viewport, structured data presence, internal linking, orphans.

### Output

- Issues with severity: Critical / High / Medium / Low
- Issue → page(s) → evidence → suggested action
- **Not** a raw count as the headline; Critical first, then “convert to tasks”

### AI role (thin)

- Summarize *clusters* of issues for humans
- Must not invent issues the crawler did not find

### Exit criteria

- US-002
- Audit job status visible
- Each issue openable with evidence

---

# Phase 6 — Search visibility intelligence

**Maps to:** Module 3.

### Features

- Keyword/page performance from GSC
- Ranking groups: Top 3 / 10 / 20 / 50 / Not ranking (position buckets from GSC avg position — **label as modeled from GSC, not rank-tracker SERP**)
- Trends, branded vs non-branded (user-defined brand terms)
- Intent classification (rules + optional LLM, labeled as interpretation)
- Visibility by topic and page type
- SERP features: **defer true SERP scrape** unless you add a rank-tracker provider; MVP uses GSC only and is honest about that

### Product decision (lock here)

- **MVP visibility = GSC-based.** Paid rank tracking is a Phase 9 adapter so you do not pretend GSC position is a SERP rank tracker.

### Exit criteria

- Visibility dashboard answers “where are we visible / invisible”
- Filters by query, page, country, date range
- Movement vs previous period

---

# Phase 7 — Opportunity, scoring, Next Best Action, page optimization

**This is the product.** Maps to Modules 5, §§19–21, US-003, US-004, signature UX §45.

## 7.1 Opportunity generation (deterministic + statistical)

Candidate types for MVP:

- High impressions, low CTR (title/meta)
- Position 8–20 with business-relevant queries
- Critical indexability on money pages
- Cannibalization (multiple pages ranking for same query)
- Content gaps vs user-listed competitors (limited: compare crawled competitor homepages/key URLs **or** GSC if they add competitor later — keep MVP competitor **basic**)
- Declining queries/pages (WoW/MoM)

## 7.2 Scoring

```
priority = business_value * visibility_opportunity * confidence * feasibility
```

- **Business value:** project products/services match, user-tagged money pages, conversion URL list
- **Visibility opportunity:** impressions × distance from top 10 × CTR gap vs expected CTR curve
- **Confidence:** data volume, crawl freshness, GSC completeness
- **Feasibility:** estimated effort enum, technical vs content

Formula coefficients in config, not hardcoded magic. Show **breakdown UI**.

## 7.3 Recommendation object

Always: problem, why it matters, recommended action, suggested implementation, expected objective, evidence, confidence, measurement plan.

LLM writes **implementation guidance** only after a structured evidence payload is built in code.

## 7.4 Page Optimization Engine

User selects page (or NBA deep-links here):

- Target queries for that URL
- Intent vs content
- Title, headings, metadata, internal links, semantic coverage, images, structured data
- Competitive context: user-provided competitor URLs (fetch + extract) — not a full SERP corpus in MVP
- User value + AI discoverability as **interpreted** sections

CTA: **Start Optimization** → creates recommendation set + optional task.

## 7.5 Next Best Action

- Rank opportunities, pick #1 (with ties broken by feasibility then recency)
- Hero on project dashboard
- Explain reasoning in human language grounded in the score breakdown
- User can dismiss, snooze, or convert to task

### Exit criteria

- US-003 and US-004
- Users can go from NBA → page optimizer → task without leaving the loop
- No recommendation without evidence IDs

---

# Phase 8 — Action Center, change log, verification, project dashboard

**Maps to:** Modules 9–11, US-005, US-006, Dashboard §32, MVP success criteria.

## 8.1 Action Center

Task fields per §26. Status pipeline:

Backlog → Planned → In Progress → Implemented → Verified → Monitoring → Completed

Assign owner, due date, effort. MVP: in-app users only (no Jira).

## 8.2 Change log

Record: URL, elements changed, reason, owner, date, before/after snapshots (manual notes + optional crawled title/H1 diff).

Link task ↔ change ↔ recommendation.

## 8.3 Verification (basic)

- Snapshot metrics at implementation date (baseline window vs monitoring window, default 28 days)
- Show ranking/impressions/clicks/CTR (and AI/local later)
- Copy: **observed change, not causation**
- Simple experiment entity (objective, change, baseline, period, result, interpretation)

## 8.4 Project dashboard

Six questions: How visible? What changed? What’s going wrong? What should we do? What have we implemented? Did it work?

## 8.5 Reporting (MVP slice)

- In-app “monthly” view using the same six questions
- Export PDF/PNG later; CSV of opportunities/tasks in MVP is enough
- White-label deferred

### Exit criteria

- Full MVP journey §44 completable
- Success signal possible: “I implemented a rec and came back to see what happened”
- Activation funnel events instrumented (§40)

**MVP freeze.** Ship to design partners (agencies). Do not start Phase 9 until activation and “at least one implementation” are observed.

---

# Phase 9 — Search Visibility OS (PRD Phase 2)

Ship as slices; order below is recommended.

## 9.1 Competitor intelligence (Module 4, basic → full)

- Competitor domains, keyword overlap if you add rank data or third-party API
- Competitor Gap: “they rank for N high-value queries you do not”
- Content/topical coverage, SERP presence
- Gap scoring: which gaps are worth pursuing (reuse priority formula)

## 9.2 Content Opportunity Engine (Module 8)

- Missing topics, declining/outdated/weak pages, cannibalization, unanswered questions, high-impression/low-CTR
- Actions: update, create, consolidate, interlink, restructure — **not** “write 2,000 words”

## 9.3 Google Analytics integration

- Sessions, landings, conversions on dashboards, scoring (business value), verification, reports

## 9.4 Reporting v2

- Scheduled email reports
- “What changed?” narrative section
- Stakeholder vs strategist views

## 9.5 Rank tracking adapter (optional)

- Third-party or in-house SERP collection for true positions and SERP features
- Keep GSC and rank tracker visually distinct

---

# Phase 10 — AI search visibility (Modules 6–7 in PRD numbering: AI + citation gap)

**Maps to:** JTBD 7, H6, non-goal of “extensive provider coverage” for MVP.

### Architecture

- Prompt set + providers behind adapters
- Store raw model outputs, citations, mentions
- Strict **Observed vs Interpreted**
- Cost controls and sampling of query sets (user-defined prompts)

### Track

Brand/product mentions, recommendations, citations, sources, competitor mentions, query-level visibility, sentiment only if reliable

### AI Citation Gap

Share of voice vs competitors → sources cited → entity/content/schema gaps → actions into Opportunity engine

### Scope control

Start with **one** environment (e.g. a single API-accessible answer engine), then add ChatGPT/Gemini/Perplexity/Google AI as adapters.

---

# Phase 11 — Local visibility (Module 7)

- Local keywords, geo-grid or location-level rankings (provider or manual locations)
- GBP monitoring where feasible (OAuth)
- Local competitors, landing pages, reviews, NAP consistency, local content opportunities, visibility by location (Lagos Island vs Lekki example)
- Feed local issues into the same Opportunity + NBA pipeline

Defer “complex local SEO management” until this slice is validated.

---

# Phase 12 — Agency Mode (Module 31, US-007)

- Multiple workspaces/clients, client permissions, team
- Agency command center: health, priority action counts, alerts
- Client dashboard (limited)
- White-label reports, scheduled reports, cross-client opportunities
- Project health rollup

Keep Action Center as the work system; do **not** become Jira.

---

# Phase 13 — Execution platform and autonomous loop (PRD Phase 3–4)

## Implementation layer

1. Recommendation only (done)
2. Exact implementation instructions (CMS-agnostic diffs, schema JSON, copy)
3. CMS integrations (WordPress, Shopify, Webflow, Wix) with **explicit approval**
4. Approval-based automation; never silent production edits

## Workflow integrations

Slack, Teams, Jira, Linear, Trello, Asana, Sheets — push tasks, don’t replace Action Center

## AI Assistant (§36)

Grounded Q&A over project data: visibility loss, this week’s pages, competitor outranking, content gaps, high-impression/low-CTR, post-optimization deltas. Same evidence rules.

## Autonomous visibility agent (V4)

Continuous monitor → detect → diagnose → recommend → request approval → implement → verify → learn  
**Moat work:** store action → page change → search change → AI change → traffic → conversion. Outcome Intelligence dataset.

---

# Cross-cutting work (runs in every phase)

| Track | What |
|---|---|
| Trust | Evidence, confidence, no guarantees, observed vs interpreted |
| Scoring | Tunable formula; log all inputs for later learning |
| Jobs | Idempotency, retries, dead letters, user-visible status |
| Security | RBAC, OAuth, isolation, deletion, audit |
| Analytics | Activation, engagement, outcome metrics (§39–40) |
| QA | Contract tests for adapters; golden fixtures for scoring and diagnosis |
| Legal | GSC/GA ToS, scraping policy, AI training/data use, robots.txt |

---

# Suggested team sequencing (if small team)

| Weeks (indicative) | Focus |
|---|---|
| 1–2 | Phase 0 prototypes + Phase 1 DS in Figma |
| 3–5 | Phase 1 in code + Phase 2 skeleton |
| 6–7 | Phase 3 onboarding |
| 8–10 | Phase 4 GSC + crawler |
| 11–12 | Phase 5 audit |
| 13–14 | Phase 6 visibility |
| 15–18 | Phase 7 NBA + optimizer (protect quality) |
| 19–21 | Phase 8 actions, changes, verify, dashboard |
| 22 | Design-partner MVP, instrument, freeze |

Adjust calendar; **do not** compress Phase 7 by dumping unprioritized issue lists.

---

# Explicitly out of MVP (do not pull forward)

- Backlink index, Ahrefs/Semrush as core
- Full CRM / marketing automation / PM suite
- Unlimited AI content generation
- Autonomous or silent CMS writes
- Dozens of integrations
- Complex local + full white-label agency
- Broad AI search provider coverage
- Replacing GSC/GA
- Ranking guarantees

---

# Definition of done for “the platform exists”

A design-partner agency can:

1. Create a workspace and project  
2. Connect a site and Search Console  
3. See an audit and visibility picture  
4. Trust a **Next Best Action** with evidence  
5. Run page optimization  
6. Create and assign a task  
7. Log a change  
8. Return later and see **what changed** without the product claiming causality  

That is the PRD’s strongest validation signal. Everything after that is expansion of the same loop.
