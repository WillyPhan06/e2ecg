# e2ecg

**Evidence-Grounded AI Video Generation and Fact-Checking Platform**

> e2ecg (End-to-End Content Generation) turns a user's idea into a short video draft while adding a claim-level fact-checking layer that checks factual statements against retrieved evidence before they move into production.

Version 1.2 · Scope: **Publicly deployed on free-tier cloud platforms** · Status: Planning / Pre-implementation

## Overview

e2ecg is a web-first platform for planning, generating, reviewing, and exporting short-form video content. A user submits a topic or idea, and the system coordinates an asynchronous workflow that can:

1. Create a content plan and draft script with an AI model.
2. Extract checkable factual claims from the script.
3. Retrieve relevant external sources and evaluate the evidence for each claim.
4. Flag claims that are contradicted, unsupported, or uncertain for revision or human review.
5. Gather suitable media assets, generate narration, and compose a video from the reviewed script.
6. Render a platform-ready video the user can preview and download.
7. Show progress, evidence, decisions, artifacts, and job history in the web app.

The key differentiator is **evidence-grounded fact-checking within the content-generation workflow**. The goal is not to promise that every generated statement is always correct. Instead, the system makes factual claims visible, links them to evidence, distinguishes different evidence outcomes, and provides a review path before a video is finalized.

The application is designed for transparency: users can inspect each workflow stage, review claim-level evidence and decisions, view generated artifacts, and manage their own jobs and account history.

## Scope of This Version

This version targets **one deployment option only: a publicly reachable deployment on free-tier cloud platforms, with zero recurring cost.** "Free" here means *free within quotas*: every platform below has limits, some of which changed during 2026, so the design treats quotas as a first-class constraint instead of an afterthought. A fully local development stack (Docker Compose) mirrors the deployed one so the project can be built and demoed without any cloud account.

Think of it as a food truck with rented storage. The truck (a free VM) does the cooking. The pantry (managed database) and the cold storage (object storage) live *off* the truck, so if the truck is towed, the inventory survives and a new truck can be driven in.

| Area | In scope (this version) | Deferred (post-MVP) |
|---|---|---|
| Deployment | Hybrid free-tier cloud: Vercel Hobby (UI) + Oracle Always Free VM (API, workers, Temporal) + Neon (app database) + Cloudflare R2 (artifacts). Local Compose stack for development. | Paid / managed mode, `DEPLOYMENT_MODE` switch, Free → Paid migration |
| Auth | Auth.js (self-hosted logic, sessions or tokens backed by the app database) | Clerk or other hosted auth |
| Orchestration | Self-hosted Temporal on the VM | Temporal Cloud (it has trial credits but no permanent free tier) |
| LLM | One free-tier hosted API behind one interface, plus an optional second free provider as a quota fallback | Paid providers |
| Evidence | A small, documented set of free or open sources | Licensed data providers |
| Voice | Local TTS (Piper) on the VM | Hosted TTS |
| Media | Stock APIs (Pexels/Pixabay) + user-provided assets | Generative image providers |
| Storage | Cloudflare R2 (MinIO only in the local dev stack) | Other S3-compatible stores |
| Composition | FFmpeg-first composition sized for a small VM | Remotion templates if resources and license terms allow |
| Output | Preview + **download** of the rendered video and captions | One-click publishing to YouTube/TikTok |
| Notifications | In-app status (SSE + job history) | Email notifications |
| Observability | Structured JSON logs, private Temporal UI, DB-backed stage metrics, a free uptime monitor | Grafana/Prometheus/Loki, OpenTelemetry tracing |

Provider interfaces are still kept for the services most likely to change (LLM, evidence, TTS, storage), but each gets one implementation in this version. The single sanctioned exception is the LLM interface, which may have a second free provider as a fallback, because exhausted free quotas are the most likely way for this system to fail.

## Product Goals

- Reduce the manual work required to create social-ready short videos.
- Integrate claim-level fact-checking into generation rather than treating it as a disconnected final proofreading step.
- Make evidence and uncertainty inspectable, with source links and clear claim statuses.
- Route contradicted, unsupported, or ambiguous claims to revision or human review before production continues.
- Provide end-to-end automation with human-visible checkpoints and user control.
- Support resumable workflows, per-user data isolation, and observable job execution.
- Stay at zero recurring cost by enforcing explicit per-job and per-user limits derived from free-tier quotas, so usage can never silently exceed them.

## Core User Experience

### 1. Submit an Idea

Users provide:
- Content idea or topic.
- Optional target audience and tone.
- Output format preset (e.g., vertical 9:16 short, landscape 16:9). Platform names are used only to choose format and duration presets; they do not trigger publishing.
- Optional constraints such as duration, style, call-to-action, and language.

### 2. Generate a Plan and Draft Script

The planning service turns the idea into a structured content plan and draft script. The script should be structured enough for later stages to identify factual claims, map claims to script passages, and connect reviewed text to the final video.

### 3. Review Claim-Level Fact-Checks

The fact-checking stage identifies factual claims that can be evaluated against external evidence. For each claim, the interface should show:
- The claim text and where it appears in the script.
- Retrieved sources and links to the relevant evidence.
- The assessment outcome and a short explanation.
- Any limitations, such as weak evidence, conflicting sources, or insufficient information.
- Whether the claim can proceed, should be revised, or needs human review.

Use these outcome labels consistently:

| Status | Meaning | Expected handling |
|---|---|---|
| **Supported** | Retrieved evidence supports the claim to the extent assessed. | May proceed, subject to the configured review policy. |
| **Contradicted** | Retrieved evidence conflicts with the claim. | Flag for correction or human review; do not silently pass it through. |
| **Unsupported** | The search did not find adequate evidence supporting the claim. This does not by itself prove the claim is false. | Request a source, revise, remove, or send for review. |
| **Insufficient / conflicting evidence** | Available evidence is too limited, ambiguous, or inconsistent to make a reliable assessment. | Preserve uncertainty and request human review or further research. |

**Check state (separate from the outcome).** A claim's *outcome* describes what the evidence says. A claim's *check state* describes whether the check actually ran: `completed`, `incomplete` (e.g., search timeout, quota exhausted, provider error), or `skipped_by_limit` (the per-job claim cap was reached). An incomplete check has **no outcome** and is shown to the user as "Check incomplete", never as Unsupported. Think of a lab report: "negative" and "inconclusive" are results, while "sample lost in transit" is a different kind of fact and needs its own box on the form. On free tiers this matters even more, because quota exhaustion will be a routine cause of incomplete checks.

These labels describe the system's evidence assessment, not a guarantee of truth. Source quality, recency, context, and retrieval coverage can affect the result. A lack of retrieved evidence must not automatically be treated as proof that a claim is false.

### 4. Continue Through Video Production

Once the script passes the configured review gate (or the user explicitly resolves flagged claims), the workflow can continue through:
- A script-level content safety check.
- Media discovery and curation.
- Voice-over generation.
- Timeline, caption, and audio composition.
- Video rendering.
- Manual review and download.

The system should preserve traceability from a reviewed claim to its script passage and, where feasible, to the segment in which it appears in the rendered video. To make this feasible, each script segment carries a stable ID that flows through composition into the render spec and caption file.

### 5. Track Jobs and Review History

Users can:
- View overall and per-stage progress.
- Inspect logs, outputs, evidence, and errors.
- Watch generated videos and review scripts, media, and audio artifacts.
- See which claims were flagged and how they were resolved.
- Retry a failed stage or revise and rerun a previous job.
- Browse job history with filtering and search.
- Delete a job and its stored artifacts and evidence.
- See when a job or artifact will expire under the retention policy.

## End-to-End System Flow

1. User submits an idea through the web app.
2. API validates the request, checks per-user and global quotas, and creates a generation job.
3. The orchestration engine records workflow state and schedules asynchronous work.
4. Planning service generates a content plan and draft script.
5. Claim extraction service identifies factual claims and associates them with script locations.
6. Evidence retrieval service searches permitted external sources and stores source metadata and relevant excerpts.
7. Claim assessment service evaluates each claim against retrieved evidence and assigns one of the defined outcomes (or marks the check incomplete).
8. Review gate determines which claims can proceed and which require revision or human review.
9. **Review loop:** the script is revised, claims are reassessed, and review decisions are recorded, repeating until no flagged or incomplete claims remain unresolved, or until the configured maximum number of revision rounds is reached (after which the job waits for an explicit user decision).
10. Script-level content safety check runs on the approved script.
11. Media service gathers licensed or policy-compliant images and video clips.
12. Voice service generates narration from the approved script.
13. Composition service assembles the timeline, captions, transitions, and audio.
14. Render service exports video in the selected format.
15. User reviews the final draft and downloads the video and captions. Progress and completion are reflected live in the web app.

**Important workflow rule:** fact-checking should happen after a draft script exists and before downstream production is treated as final. The exact gate can be configurable, but flagged claims should remain visible and must not be silently represented as verified.

### Canonical Job States

One shared vocabulary is used by the database, the SSE stream, and the UI, so the three never drift apart:

`queued` → `planning` → `script_drafting` → `claim_extraction` → `evidence_retrieval` → `claim_assessment` → `review_required` ⇄ `revising` → `ready_for_production` → `content_check` → `media_collection` / `voice_generation` (parallel) → `video_composition` → `rendering` → `completed`

Terminal and side states: `failed` (with a retry path), `cancelled`, and `expired` (artifacts removed under the retention policy).

## Recommended Architecture

Think of e2ecg as a production kitchen with a quality-control station. The planning station drafts the recipe, the fact-checking station checks factual ingredients against evidence, and only then do media, voice, and rendering stations produce the final dish. A workflow coordinator tracks every station, preserves state, and helps a job recover when a station fails.

### Deployment Topology (Free-Tier Cloud)

```
 Browser
   │
   ├── HTTPS ──► Vercel Hobby ── Next.js UI (pages, auth screens)
   │
   └── HTTPS (REST + SSE) ──► Oracle Always Free VM (Arm, 2 OCPU / 12 GB)
                                 ├─ Caddy         TLS reverse proxy
                                 ├─ API           NestJS/Fastify, stateless
                                 ├─ Worker        activities: LLM calls, retrieval,
                                 │                Piper TTS, FFmpeg composition/render
                                 ├─ Temporal      server + its own Postgres (on the VM)
                                 └─ Temporal UI   private, reached via SSH tunnel only
                                       │                         │
                                       ▼                         ▼
                              Neon Postgres              Cloudflare R2
                              (app data: users,          (scripts, media, audio,
                               jobs, claims, events)      videos, captions)

 Outbound calls from the worker: free-tier LLM API(s), evidence sources,
 Pexels/Pixabay.
```

Where each component runs and why:

| Component | Runs on | Why |
|---|---|---|
| Web UI | Vercel Hobby | Free, zero-ops hosting for Next.js. Personal/non-commercial use only. |
| API, workers, Temporal, Piper, FFmpeg | One Oracle Always Free Arm VM | The only free option with enough CPU and memory for Temporal and video rendering. Typical free container PaaS tiers are far too small (a few hundred MB of RAM, sleeping after inactivity). |
| App database | Neon free Postgres | Lives off the VM, so job history survives a VM loss. Scales to zero when idle and resumes automatically. |
| Temporal's database | Postgres container on the VM | Temporal needs a low-latency, always-on store. Its history is operational state, and the app database remains the source of truth for job outcomes. |
| Artifacts | Cloudflare R2 | Free storage allowance with free egress, so video downloads cost nothing. S3-compatible. |
| TLS and public endpoint | Caddy on the VM with a free subdomain, or a Cloudflare Tunnel if you own a domain | The browser connects to the API directly for SSE (Vercel functions are not suited to long-lived streams). |
| CI/CD | GitHub Actions + GitHub Container Registry (verify current free limits) | Build images, push, and deploy to the VM over SSH. |

**Design rule: durable truth lives off the VM.** The VM can be reclaimed, resized, or rebuilt, so it must be reproducible from the repository (Compose file plus a cloud-init or provisioning script) in under an hour. On startup, the API runs a **reconciliation pass** that finds jobs in non-terminal states with no live workflow and marks them failed-and-retryable, so a rebuilt VM never leaves orphaned jobs.

### Free-Tier Reality Check

Figures below were checked in October 2026 and **will drift**. Re-verify before each milestone and before any public demo.

| Service | Used for | What you get free | The catch |
|---|---|---|---|
| Vercel Hobby | Web UI | About 100 GB data transfer, ~1M edge requests, ~1M function invocations, ~4 hours of active CPU per month | Personal, non-commercial use only; one seat; hitting a cap pauses the project instead of billing. |
| Oracle Always Free (Arm) | API, workers, Temporal, rendering | 1,500 OCPU-hours and 9,000 GB-hours per month, equivalent to **2 OCPU and 12 GB** always-on (cut from 4 OCPU / 24 GB in 2026) | Arm capacity is often unavailable ("out of capacity") in popular regions and can take days or weeks to obtain; idle instances can be reclaimed; free tier has no support. |
| Neon (free) | App database | Roughly 0.5 GB per project, scale-to-zero after about 5 minutes idle, a monthly compute-hour allowance | Small storage; cold-start latency on first query; held-open connections keep compute awake and burn the allowance. |
| Supabase (alternative) | App database | 500 MB database, free projects limited to two | Projects auto-pause after a week of inactivity and need manual unpausing. |
| Cloudflare R2 | Artifacts | 10 GB storage, 1M Class A and 10M Class B operations per month, free egress | Storage and operation counts still cap you; rendered videos fill 10 GB faster than text. |
| Temporal Cloud | (not used) | Trial credits only | No permanent free tier, so Temporal is self-hosted. |
| Free-tier LLM APIs (e.g., Gemini API, Groq) | Planning, extraction, assessment | Rate- and day-limited; caps vary per model and change without notice | Some free tiers restrict serving end users in certain regions or allow the provider to use submitted data to improve products. **Read the current terms before launch.** |
| Render / Fly.io / Koyeb | (not used) | Render's free web service sleeps after inactivity and is tiny; Fly.io and Koyeb no longer offer free compute for new accounts | Cannot host Temporal, workers, or rendering. |

### Frontend

- Next.js with TypeScript and App Router, deployed on Vercel Hobby.
- Handles idea submission, job progress, claim/evidence review, artifact previews, and job history.
- Server Components can load initial dashboard data; client components consume live progress updates directly from the API's SSE endpoint.
- Hobby is restricted to personal, non-commercial use. That fits an academic project; if the project is ever monetized, move the UI to a plan or host whose terms allow it.

### Backend API

- NestJS (or Fastify) on Node.js, running in a container on the VM behind Caddy.
- Owns authentication integration, job creation, permissions, API validation, quota checks, and access to job/artifact data.
- Stateless: no job state lives in API process memory.
- CORS is restricted to the deployed UI origin; all endpoints require an authenticated session except the health check.

### Durable Orchestration

- **Self-hosted Temporal** on the VM, with its own Postgres container.
- Coordinates multi-stage work, persists workflow state, retries transient failures, and resumes eligible jobs after worker or process failures.
- A lighter queue-based approach remains a documented fallback if Temporal proves too heavy for the VM (see Open Decisions).

Orchestration design rules (to avoid common pitfalls):
- **Pass references, not payloads.** Activities exchange artifact IDs and storage keys, never whole scripts, media files, or videos. It is the difference between handing someone a delivery receipt and mailing them the sofa. Orchestrator payloads have size limits.
- **Keep workflow code deterministic.** Database writes and event publishing happen inside activities, not in workflow code itself.
- **Heartbeat long activities** (especially rendering) and set explicit timeouts.
- **Retry semantics are explicit.** Transient failures retry automatically with backoff. If an activity exhausts its retries, the workflow enters a waiting state and accepts a user-triggered retry signal, instead of closing in a state that can no longer be signaled.
- **Quota exhaustion is not a transient error.** A daily-quota failure should pause the job and mark the affected checks incomplete, rather than burning retries against a limit that will not reset for hours.
- **Duplicate starts are an explicit design choice.** Workflow ID is derived from the job ID, and starting a duplicate must be configured to return the existing run or reject it, never to create a second pipeline.
- **The review loop is a real loop.** Revise → re-extract → re-retrieve → re-assess repeats until resolved or until the round cap, and the user's review decisions are passed into the revision step as inputs.

### Worker Services

Start with **one worker process** that registers clear module boundaries and separate task queues, and split into separate deployables only when measurement justifies it. Planned responsibilities:

- Planning and script generation.
- Claim extraction.
- Evidence retrieval.
- Claim assessment.
- Script revision and review-gate handling.
- Content safety check.
- Media discovery.
- Voice synthesis.
- Video composition and rendering (the first candidate to run as its own process with a concurrency cap of 1, because it is CPU- and memory-heavy and can starve everything else on a 2-OCPU VM).

### Fact-Checking Subsystem

The fact-checking layer is a core product capability, not a generic content-safety filter. A practical first version can use a modular pipeline:

1. **Claim extraction:** identify explicit, checkable factual statements and retain their script locations. The first version targets a defined claim taxonomy: statistics and numbers, dates and events, attributions ("X said/discovered/founded Y"), and simple causal or scientific statements. Opinions, predictions, and subjective judgments are labeled as non-checkable instead of silently dropped.
2. **Evidence retrieval:** search a small documented set of sources, collect URLs and metadata, and retain short supporting or contradicting excerpts where permitted. Page text may be processed transiently for assessment; only minimal excerpts and metadata are stored.
3. **Evidence assessment:** evaluate whether the retrieved evidence supports, contradicts, or fails to adequately resolve each claim, using only the supplied evidence.
4. **Outcome assignment:** return a structured result using the defined statuses, along with a concise rationale and evidence references.
5. **Review policy:** determine whether a claim can proceed, needs revision, or requires human review.
6. **Traceability:** persist the claim, sources, assessment, model/provider version where available, and any user decision.

Design principles:
- Separate claim extraction, evidence retrieval, and evidence assessment so each stage can be tested independently and, if needed, use a different model.
- Keep source URLs, publication details when available, and relevant evidence snippets connected to the claim.
- Do not present an LLM-generated judgment as independent proof. The system should show the evidence behind the assessment.
- Preserve uncertainty and conflicting evidence instead of forcing every claim into true/false.
- Treat web pages and retrieved documents as evidence data, not as instructions to the system. The assessment model gets no tools and no ability to take actions, and retrieved text is clearly delimited in the prompt.
- Apply source allow/deny rules, timeouts, rate limits, and safeguards against unsafe or irrelevant retrieved content.
- Avoid claiming complete fact-check coverage: claim extraction and retrieval can miss important statements or sources.
- Keep a human-review path for ambiguous, high-impact, disputed, or otherwise flagged claims.

**Model-quality note for the free option.** Claim assessment is the stage where model quality matters most. A 12 GB Arm VM cannot host a capable local model at usable speed, so deployed instances call a free-tier hosted API for extraction and assessment (a small local model remains a development-only option). Keep the assessment prompt and output schema strictly structured, and treat assessment accuracy as something to measure (see Evaluation), not assume.

### Database and Artifact Storage

- **App database:** Neon Postgres (free plan), accessed through Prisma or Drizzle with a pooled connection string. Stores users, jobs, workflow steps and events, claims, claim assessments, evidence references, review decisions, artifact records, and usage records.
- **Keep the database small.** The free plan has a small storage cap, so the database holds metadata and short excerpts only. Scripts, media, audio, videos, and larger evidence snapshots go to object storage.
- **Object storage:** Cloudflare R2 (S3-compatible, free egress). MinIO is used only in the local dev stack; because both speak the S3 API, no code changes are needed between them.
- **Retention is mandatory, not optional.** Artifacts expire automatically (default 14 days, configurable) using bucket lifecycle rules plus a cleanup job that marks jobs `expired`. Without this, rendered videos will exhaust the free storage allowance quickly.
- Access to stored objects is through short-lived signed URLs issued by the API; the browser never receives storage credentials.
- Object keys are versioned by script version where relevant (e.g., `/{userId}/{jobId}/script/v{n}.json`).
- **Backups:** a scheduled `pg_dump` of the app database is written to a separate R2 prefix (weekly, keep the last few), so a database mishap or account problem is recoverable.

### Real-Time Updates

- Server-Sent Events (SSE) for one-way progress updates, streamed directly from the API on the VM to the browser.
- Stream workflow state changes using the canonical job states above, plus claim-review counts and transitions.
- **Every event is persisted** in a `job_events` table with a monotonically increasing sequence number. The SSE handler reads new rows for the connected job (polling every one to two seconds only while a client is connected) and supports `Last-Event-ID` replay on reconnect. The stream is a convenience layer, never the only record of progress.
- No Redis and no `LISTEN/NOTIFY` are used. A held-open listener connection would keep Neon's compute awake and consume its free allowance, and there is only one API instance, so simple polling of an indexed table is enough.

### Authentication and Security

- Auth.js in the Next.js app, with sessions or signed tokens backed by the app database. The API validates the token on every request. Registration and login are handled by Auth.js rather than custom endpoints.
- Enforce per-user authorization for jobs, claims, evidence, script versions, review decisions, and artifacts. Ownership is always derived from the authenticated session, never from client-supplied IDs.
- Secrets live in Vercel environment variables, GitHub Actions secrets, and a root-only `.env` on the VM. They are never committed and never exposed to the client or logs.
- Expose only ports 80/443 on the VM. SSH uses keys only; Temporal's UI and database ports are bound to localhost and reached through an SSH tunnel.

### Deployment and Observability

- **Deployed stack:** Vercel (UI), the Oracle VM running `docker-compose.free.yml` (Caddy, API, worker, Temporal, Temporal Postgres, optional Temporal UI), Neon, and R2.
- **Local stack:** `docker-compose.dev.yml` adds local Postgres and MinIO so the entire system runs on a laptop with no cloud accounts.
- **Start early:** request the Oracle VM in the first week. Arm capacity can be slow to obtain, and the whole project should not be blocked on it. A fallback is to run the worker on a spare local machine that connects to the Temporal server, accepting that rendering then depends on that machine being online.
- **Sizing (rough estimates, to be measured):** with 12 GB RAM, budget roughly 2 GB for Temporal and its database, 0.5 GB for the API, 1 GB for the worker, 0.5 GB for Piper, 1–2 GB peak for an FFmpeg render, and about 1 GB for OS and Docker overhead. That leaves headroom, but only with render concurrency fixed at 1. Confirm that every container image has an `arm64` build.
- **Availability reality:** this is a demo-grade deployment. The VM is a single point of failure, free instances can be reclaimed if idle, and platforms can change free-tier terms. The reconciliation pass, off-VM data, backups, and reproducible provisioning exist to make recovery quick, not to promise uptime.
- **Account safety:** if the Oracle account is upgraded to Pay-As-You-Go to improve capacity, usage within the Always Free limits is not billed, but a payment card is attached. Set budget alerts and a hard cap on any resource that can bill.
- **MVP observability:** structured JSON logs correlated by `jobId` and workflow ID/run ID, the private Temporal UI, per-stage duration/failure/usage data in the app database viewable on a simple admin page, and a free uptime monitor pinging the API health endpoint.
- A Grafana/Prometheus/Loki stack and OpenTelemetry tracing are valuable but deferred, because they compete for the same limited VM resources.

## High-Level Components

- Authentication and User Accounts
- Idea Intake and Prompt Builder
- AI Planning and Script Generation
- Claim Extraction
- Evidence Retrieval and Source Metadata
- Claim Assessment and Outcome Assignment
- Claim Review and Script Revision
- Script Content Safety Check
- Media Discovery and Curation
- AI Voice Synthesis
- Video Composition and Rendering
- Progress Timeline and History UI
- Orchestration and Workflow State
- Quota, Usage Tracking, and Retention
- Provider Abstraction and Configuration

## Recommended Tech Stack (Free-Tier Cloud)

"Free" does not mean unlimited or production-reliable. Candidates below are starting points, not commitments, and limits must be re-checked during implementation.

| Layer | Choice | Purpose |
|---|---|---|
| Frontend | Next.js (TypeScript, App Router) on Vercel Hobby | Submission, progress, evidence review, and history UI |
| Backend API | NestJS or Fastify on Node.js, containerized on the Oracle VM | Job API, authorization, quotas, and application logic |
| Orchestration | Self-hosted Temporal (Docker, on the VM) | Durable multi-step workflows and retries |
| App database | Neon free Postgres | Jobs, claims, assessments, evidence metadata, review decisions, events, users |
| Temporal persistence | Postgres container on the VM | Workflow history |
| Object storage | Cloudflare R2 (MinIO locally) | Scripts, media, audio, and video artifacts |
| Real-time updates | SSE reading the persisted `job_events` table | Live workflow and review progress |
| Auth | Auth.js | Account and session management |
| Planning / script / extraction / assessment | One free-tier hosted LLM API, with an optional second free provider as quota fallback | Structured planning, claim identification, and evidence-based classification |
| Evidence retrieval | A small documented set to verify before Phase 6: e.g., Wikipedia/Wikidata APIs, a fact-check claim-search API, and one free-tier web search API | Find sources relevant to each claim |
| Media discovery | Pexels/Pixabay stock APIs and user-provided assets; no generative images | Source visuals with license/provenance metadata |
| Voice synthesis | Piper (local, open source) | Narration. Unofficial or reverse-engineered TTS endpoints are avoided because of terms-of-service risk. |
| Video composition/render | FFmpeg (image/clip sequencing, burned-in captions, audio mix) | Compose and export video on a small VM. Remotion is deferred: headless-browser rendering is memory-heavy and carries license terms to check. |
| Reverse proxy / TLS | Caddy | HTTPS for the API and SSE |
| CI/CD | GitHub Actions + GHCR | Build, push, and deploy |
| Observability | JSON logs, Temporal UI, DB-backed stage metrics, free uptime monitor | Debugging and reliability metrics |

Keep external services behind small interfaces where switching providers is realistic, one implementation per interface for now (plus the LLM fallback exception). Do not build a broad abstraction framework before the first end-to-end workflow works.

## Configuration

- A single setup command (e.g., `npm run configure`) writes `.env.local`, validates that required keys are present, and points the developer at the right Compose file (`dev` or `free`).
- Provider keys, quota ceilings (daily LLM requests, daily jobs), retention days, and per-job limits are all environment variables, so tuning to a changed free tier requires no code changes.
- `DEPLOYMENT_MODE=free` is reserved so a future paid mode can be added without renaming anything, but no code path should depend on a second value yet.
- Never put API keys in source control or expose them to the client.

## Conceptual Data Model

### User
- `id`
- `email`
- Authentication provider ID or password hash, depending on Auth.js configuration
- Profile metadata
- `created_at`

### Job
- `id`
- `user_id`
- Original idea and generation constraints
- Output format preset
- Overall status (canonical job state)
- Created/updated timestamps and artifact expiry timestamp

### Job Step
- `id`
- `job_id`
- Step name, attempt number, and status
- Start/finish timestamps
- Error details and retry metadata

### Job Event
- `id` and a per-job monotonically increasing sequence number
- `job_id`
- Event type and payload
- Timestamp and correlation information (workflow ID/run ID)

### Script Version
- `id`
- `job_id`
- Version number
- Script/content-plan payload, with stable segment IDs
- Revision reason and review state
- Created timestamp

### Claim
- `id`
- `job_id`
- `script_version_id`
- Claim text
- Script location (paragraph, sentence, or segment ID)
- Claim type and extraction metadata
- Checkability flag (checkable / non-checkable, with reason)
- Created timestamp

### Claim Assessment
A separate, **append-only** record for each assessment run. Rechecking adds a new row instead of editing an old one: a ledger gets a new line, not an eraser.
- `id`
- `claim_id`
- Check state (`completed`, `incomplete`, `skipped_by_limit`)
- Outcome (supported / contradicted / unsupported / insufficient_or_conflicting), empty when the check is not completed
- Rationale and uncertainty notes
- Model/provider version and assessment timestamp
- Failure reason when incomplete (including quota exhaustion)

### Evidence Source
- `id`
- `assessment_id` (and therefore the claim)
- Source URL
- Source title, publisher, publication date, and retrieval timestamp when available
- Relevant excerpt or evidence summary, subject to source terms
- Evidence direction/relevance and quality notes
- Content hash or other deduplication metadata

### Claim Review / Decision
Immutable once written.
- `id`
- `claim_id`
- Decision (accept, revise, remove, or request further review)
- Reviewer type (user/system)
- Rationale and timestamp
- Link to the resulting script version when applicable

### Artifact
- `id`
- `job_id`
- Optional script-version or workflow-step reference
- Type (script, evidence report, image, clip, voice, rendered video, captions)
- Object-storage key, size, and metadata (including license/provenance for media)
- Expiry timestamp

### Usage Record
- `id`
- `job_id` and `user_id`
- Provider, operation, units consumed (requests, tokens, searches, characters, render seconds)
- Timestamp

The schema is conceptual. Normalize it based on actual access patterns, audit requirements, and the free database's size cap. Add indexes for job history (`user_id, created_at`), job events (`job_id, sequence`), claims by job and script version, assessments by claim, evidence by assessment, usage by day and provider, and artifacts by job and expiry.

## API Surface (Draft)

### Jobs and Workflow
- `POST /jobs`
- `GET /jobs`
- `GET /jobs/:id`
- `DELETE /jobs/:id` — delete a job with its artifacts and evidence
- `GET /jobs/:id/steps`
- `GET /jobs/:id/artifacts`
- `GET /jobs/:id/usage` — per-job provider usage against limits
- `GET /jobs/:id/stream` — SSE stream with `Last-Event-ID` replay
- `POST /jobs/:id/retry` — signals the waiting workflow to retry the failed stage, preserving completed artifacts
- `GET /quota` — the caller's remaining daily jobs and the current global capacity state

### Claims and Evidence
- `GET /jobs/:id/claims`
- `GET /claims/:id`
- `GET /claims/:id/evidence`
- `POST /claims/:id/review` — record a review decision
- `POST /jobs/:id/recheck` — create a new assessment set after a script revision

### Operations
- `GET /health` — liveness and dependency checks, used by the uptime monitor

Sign-up and login are provided by Auth.js routes, not by custom API endpoints. These routes are a draft surface, not a commitment to expose every operation in the first release. Enforce authorization on every job, claim, evidence, and artifact endpoint.

## Progress and History Requirements

- Every workflow stage emits structured status events using the canonical job states.
- The UI shows both overall and per-stage progress.
- Fact-check progress is visible separately from video production.
- Each claim assessment links to the sources used and explains its outcome.
- Persist workflow events, evidence metadata, claim outcomes, and review decisions for history and debugging.
- Make clear when evidence is unavailable, conflicting, stale, or insufficient, and when a check is incomplete (including because a daily quota was reached).
- Show whether a flagged claim was revised, removed, accepted with an explicit review decision, or left unresolved.
- Include failure reasons and retry metadata, written in actionable language rather than raw stack traces.
- Show the final video, captions, download links, and expiry date when available.
- Show a clear "service is at capacity, try again after the daily reset" state instead of a generic failure when global quotas are exhausted.

## Security, Source Quality, and Compliance

- Enforce per-user data isolation across jobs, claims, evidence, and artifacts.
- Keep secrets out of logs and out of browser responses.
- Respect source website terms, search-provider terms, rate limits, robots/access restrictions where applicable, and copyright.
- Store only the evidence snippets and source content permitted by applicable terms; prefer source URLs and minimal excerpts over copying entire pages.
- Preserve source provenance and retrieval timestamps so users can assess evidence context and recency.
- **Treat retrieved page content as untrusted input.** It must not override system instructions or trigger unauthorized actions. The assessment model has no tool access, and retrieved text is delimited and labeled as data.
- **Protect retrieval workers against SSRF and abuse:** allow only `http`/`https`, block private, loopback, and link-local address ranges (including after redirects), cap redirects, response size, and request time, and apply per-domain allow/deny rules. This matters more on a public deployment, where the worker shares a network with Temporal and its database.
- Keep media licensing/provenance metadata and use assets that are permitted for the intended use.
- **Check free-tier terms for your use case.** Free hosting and free API tiers often restrict commercial use, serving end users in certain regions, or reuse of submitted data. Before opening the deployment to the public, confirm that each provider's current terms permit it, and tell users which providers see their submitted ideas.
- **Run the content safety check on the approved script, before spending media, voice, and render resources**, and not after production is already finished. Do not confuse general safety moderation with factual verification.
- **Label AI-generated content.** Platforms increasingly expect disclosure of synthetic or AI-generated media, so the exported video's metadata and the UI should make the AI origin clear.
- **Data lifecycle:** users can delete jobs, and deletion removes stored artifacts and evidence. Artifacts also expire automatically under the retention policy.
- Make the limits of automated checking clear: outcomes depend on extraction, retrieval coverage, source quality, and assessment accuracy.

## Cost and Resource Controls

A zero-cost deployment only stays zero-cost if usage is capped. Limits are enforced by configuration and derived from the free-tier quotas actually in force:

- **Quota-derived job cap.** Estimate the number of LLM calls per job (plan, script, extraction, one assessment per claim, plus revision rounds, so roughly 20–40 calls) and divide the smallest relevant daily provider limit by it. A free tier allowing a few hundred requests per day supports on the order of ten jobs per day. Set `MAX_JOBS_PER_DAY` accordingly, and re-derive it whenever a provider changes its limits.
- Per-user daily job limit, with a global daily limit behind it, and a clear "at capacity" response when either is reached.
- Maximum number of checkable claims per job (default around 12; extras are marked `skipped_by_limit` and shown as such).
- Maximum sources retrieved per claim, and a retrieval timeout.
- Maximum number of revision rounds per job.
- Caching and deduplication of retrieval results (by query and content hash).
- Render concurrency of 1, a maximum video duration (about 60 seconds), and a default resolution suited to a small VM (e.g., 720p vertical, with higher resolution as an opt-in).
- Artifact retention with automatic expiry, to stay within object-storage limits.
- Every provider call writes a usage record, so consumption (requests, tokens, searches, render seconds) is visible per job and per day, with alerts at around 80% of any quota.
- A kill switch (configuration flag) that stops accepting new jobs without taking the site down.

## Failure Handling and Reliability

- Retry transient failures with exponential backoff where appropriate, with policies tuned per activity (more aggressive for LLM rate limits, conservative for external APIs with strict quotas).
- Distinguish provider/network failures from a substantive assessment result. A search timeout is not an "unsupported" verdict; it is an incomplete check.
- Treat quota exhaustion separately from transient errors: pause or fail the affected stage with a clear message, and fall back to the secondary LLM provider where one is configured.
- Mark unrecoverable failures with actionable user-facing errors.
- Resume from a failed workflow stage where possible, via the retry signal.
- If evidence retrieval or assessment fails, do not silently bypass the fact-check gate. Mark the check incomplete and follow the configured policy.
- Keep review decisions and script revisions auditable and append-only.
- Handle the sleeping database: expect the first query after idle to be slower, retry connection errors briefly, and never treat a cold start as a failure.
- Use timeouts, concurrency limits, caching, and deduplication to control cost and prevent repeated retrieval of the same material.
- **Recovery drills before launch:** (1) kill the worker mid-render and confirm the job resumes without data loss; (2) destroy and rebuild the VM from the repository and confirm the reconciliation pass cleans up in-flight jobs; (3) restore the app database from a backup.

## MVP Scope

The MVP should prove the core differentiator (a traceable, claim-level fact-checking workflow connected to video generation) before expanding into anything larger.

### Core MVP
- Basic user authentication and account isolation.
- Idea submission and asynchronous job creation.
- AI-generated content plan and draft script.
- Claim extraction from the script.
- Evidence retrieval from a limited, documented set of sources/providers.
- Claim assessment with the four defined outcomes and the separate check state.
- Claim-level UI showing evidence links, rationale, and review status.
- A review gate that supports revision, removal, or human review of flagged claims.
- Rechecking claims when the script changes, with prior results preserved.
- Script-level content safety check.
- Basic media retrieval, voice generation, and FFmpeg-based video composition/rendering.
- Preview and download of the final video and captions.
- Live progress and job history, including persisted claim/evidence results.
- Quota controls, retention, and clear at-capacity messaging.
- A documented evaluation set and scoring results for the fact-checking subsystem.
- A public deployment on the free-tier stack described above, with a reproducible local dev stack.

### MVP Constraints
- Limit supported topics, sources, model providers, and video formats if needed to keep evaluation manageable.
- Do not promise exhaustive fact-checking or universal factual accuracy.
- Do not require training a custom foundation model.
- Prefer a small number of well-tested integrations over many partially implemented providers.
- Keep provider usage and storage within free-tier limits via the cost and resource controls above.
- Keep the video output modest (short duration, moderate resolution) so rendering fits a 2-OCPU VM.

## Future Enhancements

- **Paid / managed deployment mode** (managed database, storage, orchestration, hosted auth, paid LLM and TTS providers) and a documented Free → Paid migration path.
- **Publishing integrations** (YouTube, TikTok) with official APIs, explicit user approval, idempotent uploads, and checks of each platform's app-review and quota requirements first.
- Email notifications and completion alerts.
- Full observability stack (OpenTelemetry, Grafana, Prometheus, Loki) and alerting.
- Remotion-based templates and richer motion graphics.
- Generative image providers as a media fallback.
- More source providers and domain-specific retrieval.
- Better source-quality, recency, and corroboration signals.
- Claim dependency mapping for claims that rely on other claims.
- Claim-level timestamps or direct mapping to video segments.
- Human review queues and collaborative approval workflows.
- Multi-style templates and brand kits.
- Multiple script/video variants for the same idea.
- Multi-language generation and evidence retrieval.
- Analytics sync for post performance.
- User-configurable evidence policies and source preferences.
- More advanced source conflict resolution and larger evaluation datasets.

## Non-Goals for MVP

- Guaranteeing that every generated claim is true or that every relevant claim is found.
- Replacing expert review for high-impact or specialized topics.
- Building a new foundation model or training a model on a large viral-video corpus.
- Competing with professional timeline editors or attempting a full professional editing suite.
- Paid/managed deployment, automatic publishing, and email notifications (deferred, see Future Enhancements).
- Deep social analytics and attribution.
- Enterprise organization management.
- Commercial operation on free tiers whose terms forbid it.
- Production-grade availability. The single VM is a known single point of failure, and free-tier terms can change without notice.

## Development Roadmap

The order puts the differentiator and its evaluation first, deploys early so hosting problems surface while they are cheap to fix, and keeps visuals simple until the fact-checking path works. Each phase should end in a demoable state.

1. **Foundation and accounts:** monorepo setup, `docker-compose.dev.yml`, configuration command, CI. **In the first week, request the Oracle VM** and create the Vercel, Neon, and R2 accounts, so capacity delays do not block later phases. Record the verified free-tier limits in the repo.
2. **Auth and data:** Auth.js, schema and migrations on Neon, per-user isolation at the query layer, app shell.
3. **Walking skeleton, deployed:** workflow with stubbed activities, `job_events` persistence, SSE with replay, live progress UI. **Deploy this to the real free-tier stack now** (Vercel + VM + Neon + R2), including CI/CD, TLS, and health checks. No real AI calls yet.
4. **Planning:** structured AI content plan and script generation with schema validation and retry-with-repair, plus usage records and quota counters.
5. **Claim extraction:** identify claims from the defined taxonomy and map them to script segment IDs.
6. **Evidence retrieval:** integrate a limited set of sources (verifying each one's current free tier and terms first), SSRF protections, caching, and persisted source metadata.
7. **Claim assessment:** implement the four outcomes plus the separate check state, rationale, and evidence-linked result format, with quota-aware fallback.
8. **Evaluation (early, not last):** build a small annotated claim set (a target of roughly 50–100 claims, with a documented annotation procedure and, where possible, two annotators). Score extraction coverage, evidence relevance, assessment agreement, and failure handling. Repeat after prompt or provider changes.
9. **Review gate:** claim review UI, review loop with a round cap, script revision, and append-only reassessment.
10. **Content check, media, and voice:** script-level safety check, stock media with provenance, Piper narration.
11. **Composition and rendering:** start with a deliberately simple FFmpeg template (captions over stock footage with narration), then iterate. Add `.srt` export and claim-to-segment traceability. Measure render time and memory on the real VM.
12. **Hardening:** quota-derived limits, retention and expiry, backups, uptime monitoring, structured logs, retry tuning, and the three recovery drills (worker kill, VM rebuild, database restore).
13. **Documentation and acceptance:** setup and deployment docs, a free-tier terms checklist, and an end-to-end acceptance run against the MVP scope checklist on the deployed stack.
14. **Optional, post-MVP:** publishing, email, observability stack, Remotion templates, paid mode.

## Evaluation and Success Metrics

Measure both product workflow performance and the quality of the fact-checking subsystem.

### Workflow Metrics
- Time from idea submission to first reviewable script/video draft.
- Job completion rate.
- Failure rate and retry count per workflow stage.
- Video render success rate and render time on the deployed VM.
- Provider latency, quota use, and per-job usage against limits.
- Share of jobs delayed or degraded by quota exhaustion.

### Fact-Checking Metrics
- **Claim extraction coverage:** proportion of checkable claims in an annotated sample that the system identifies.
- **Evidence relevance:** whether retrieved sources actually address the corresponding claim.
- **Assessment quality:** agreement between system outcomes and a documented human-reviewed reference set.
- **Unsupported vs. contradicted distinction:** whether the system avoids treating lack of evidence as proof of falsity.
- **Uncertainty handling:** whether conflicting or insufficient evidence is surfaced rather than forced into a confident conclusion.
- **Incomplete-check handling:** whether provider failures are recorded as incomplete checks and never as outcomes.
- **Traceability:** proportion of assessments with accessible source references and a rationale.
- **Review-gate behavior:** whether flagged or incomplete checks are handled according to the configured policy.
- **Recheck consistency:** whether revised claims are reassessed and prior decisions remain auditable.

For an academic project, document the test-set construction, annotation procedure, scoring method, and limitations. Avoid claiming the system is "accurate" based only on a few hand-picked demonstrations.

## Risks and Mitigations

| Risk | Why it matters | Mitigation |
|---|---|---|
| Claim extraction misses a factual statement | A claim that is never extracted cannot be checked by the pipeline. | Evaluate against annotated scripts; show users the extracted claims and state coverage limitations. |
| Retrieval finds weak or irrelevant sources | The assessment may appear evidence-based while relying on poor evidence. | Show source links, measure relevance, use source-quality guidance, and support further review. |
| Conflicting or outdated evidence | Claims may be time-sensitive or legitimately disputed. | Preserve retrieval dates, show conflicts, and use an insufficient/conflicting outcome. |
| LLM assessment overstates certainty | Users may mistake generated reasoning for proof. | Require evidence-linked results, constrained structured outputs, calibrated language, and human review. |
| Free-tier LLM is weaker or rate-limited | The core differentiator degrades, or jobs stall mid-pipeline. | Measure accuracy on the evaluation set, keep human review on flagged claims, derive job caps from quotas, and keep a second free provider as fallback. |
| Free-tier terms or limits change | Oracle halved its free Arm allocation in 2026, and LLM free tiers change without notice. | Keep limits in configuration, re-verify before milestones, keep the stack reproducible, and keep the paid-mode path documented. |
| Oracle VM unavailable, reclaimed, or never provisioned | The workers and Temporal have nowhere to run. | Request it in week 1, keep data off the VM, script provisioning, keep a local-worker fallback, and run a VM-rebuild drill. |
| 2 OCPU / 12 GB is too small | Rendering can starve the API, Temporal, and the database. | Render concurrency of 1, FFmpeg-first composition, a maximum duration and resolution, measurement in Phase 11, and a lighter queue as fallback. |
| Database sleeps or hits its size cap | Cold starts slow requests, and a full database blocks writes. | Retry on connection errors, store only metadata and excerpts, enforce retention, and back up to R2. |
| Object storage fills up | Rendered videos exceed the free allowance. | Automatic artifact expiry, a duration cap, and storage usage alerts. |
| Free-tier terms forbid the intended use | A "free" plan may bar commercial use or restrict serving certain regions. | Review terms per provider before public launch; treat the project as non-commercial; disclose which providers process user input. |
| Prompt injection via retrieved pages | Malicious content could try to steer the assessment. | Tool-less assessment model, delimited data, structured outputs, and domain allow/deny rules. |
| SSRF through retrieval | A crafted URL could reach internal services such as Temporal. | Scheme and IP-range blocking (including after redirects), redirect/size/time caps, and network isolation of the worker where possible. |
| Copyright or terms restrict evidence storage | Saving full pages or excerpts may violate source/provider rules. | Store URLs and metadata, keep excerpts minimal, process page text transiently, and verify terms before implementation. |
| Pipeline becomes too complex for the project timeline | Many workers and integrations can delay a working demonstration. | One worker process with module boundaries, one implementation per interface, deploy the skeleton early, and a vertical slice first. |

## Open Decisions

- **Temporal vs. a lighter queue.** Self-hosted Temporal is the working choice. If it proves too heavy for 12 GB alongside rendering, fall back to a simpler queue and a hand-rolled state machine, accepting the loss of durable-workflow guarantees.
- **One database or two.** The current plan keeps the app database on Neon and Temporal's on the VM. A simpler alternative is a single Postgres on the VM with frequent backups to R2, at the cost of keeping job history on a reclaimable machine.
- **Neon vs. Supabase.** Neon resumes automatically; Supabase bundles auth and storage but pauses after a week of inactivity. Neon is the default.
- **LLM provider(s).** Choose the primary and fallback free providers after measuring assessment quality on the evaluation set and reading each provider's current terms.
- **Evidence sources.** Pick the first documented set (and confirm each one's free allowance and terms) before Phase 6.
- **Where HTTPS comes from.** A free subdomain with Caddy, or a Cloudflare Tunnel with an owned domain. A domain is the one item that may cost a small amount.
- **Composition and rendering as one activity or two.** Currently split so a composition bug does not waste an expensive render; revisit with real timing data.

## Scope Decisions in This Version (1.2)

| Decision | Reason |
|---|---|
| Free-tier cloud deployment only, with a local dev stack | One deployment to build, test, and document, and it is publicly reachable. |
| Hybrid topology (Vercel + Oracle VM + Neon + R2) | No single free platform can host the UI, workflows, rendering, database, and storage. Each piece sits where its free tier fits. |
| Durable data off the VM | The free VM can be reclaimed or rebuilt, so job history and artifacts must not live on it. |
| Self-hosted Temporal, not Temporal Cloud | Temporal Cloud offers trial credits but no permanent free tier. |
| Cloudflare R2 instead of MinIO in production | Managed, S3-compatible, free egress, and off the VM. |
| FFmpeg-first composition | Fits 2 OCPU / 12 GB far better than headless-browser rendering, and avoids extra license questions. |
| SSE polls `job_events`; no Redis or `LISTEN/NOTIFY` | Avoids an extra service and avoids keeping the sleeping database awake. |
| Quota-derived limits and mandatory retention | Zero recurring cost only holds if usage is bounded by the real free-tier quotas. |
| Deploy the walking skeleton early | Capacity, TLS, and CORS problems surface while they are cheap to fix. |
| One implementation per provider interface (LLM fallback excepted) | Keeps the abstraction without paying for two integrations everywhere. |
| Publishing deferred; output is download-only | Platform app reviews and quotas are a large, uncertain cost, and publishing must stay user-controlled. |
| Email, Grafana stack, and Redis deferred | Fewer moving parts on a resource-limited VM. |
| Separate check state, append-only assessments | Lets "check incomplete" be represented honestly and keeps history immutable. |
| Evaluation moved ahead of media and rendering | The differentiator needs evidence of quality before the surrounding pipeline grows. |
| Safety check moved to the approved script | Avoids spending media, voice, and render resources on content that would be rejected. |

## Project Name

**e2ecg** means **End-to-End Content Generation**. The expanded project positioning is **Evidence-Grounded AI Video Generation and Fact-Checking Platform**.

## Current Status

Planning and specification stage, with scope narrowed to a free-tier, publicly deployed system. The next implementation milestone should be a small end-to-end vertical slice, deployed on the real free-tier stack:

**idea → draft script → extracted claims → retrieved evidence → claim assessments → review/revision → approved script → simple rendered video**

This slice should demonstrate the project's main contribution before optional publishing, paid providers, and broader integrations are added.

`ARCHITECTURE.md` and `todo.md` were written for the earlier dual-mode, single-machine scope and should be brought in line with this version (free-tier hybrid topology, R2 and Neon, FFmpeg-first composition, canonical job states, append-only assessments, deferred publishing and observability stack).