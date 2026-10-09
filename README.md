# e2ecg

**Evidence-Grounded AI Video Generation and Fact-Checking Platform**

> e2ecg (End-to-End Content Generation) turns a user's idea into a short video draft while adding a claim-level fact-checking layer that checks factual statements against retrieved evidence before they move into production.

## Overview

e2ecg is a web-first platform for planning, generating, reviewing, and publishing short-form video content. A user submits a topic or idea, and the system coordinates an asynchronous workflow that can:

1. Create a content plan and draft script with an AI model.
2. Extract checkable factual claims from the script.
3. Retrieve relevant external sources and evaluate the evidence for each claim.
4. Flag claims that are contradicted, unsupported, or uncertain for revision or human review.
5. Gather suitable media assets, generate narration, and compose a video from the reviewed script.
6. Render a platform-ready video and, when configured, publish it to supported social platforms.
7. Show progress, evidence, decisions, artifacts, and job history in the web app.

The key differentiator is **evidence-grounded fact-checking within the content-generation workflow**. The goal is not to promise that every generated statement is always correct. Instead, the system makes factual claims visible, links them to evidence, distinguishes different evidence outcomes, and provides a review path before a video is finalized.

The application is designed for transparency: users can inspect each workflow stage, review claim-level evidence and decisions, view generated artifacts, and manage their own jobs and account history.

## Product Goals

- Reduce the manual work required to create social-ready short videos.
- Integrate claim-level fact-checking into generation rather than treating it as a disconnected final proofreading step.
- Make evidence and uncertainty inspectable, with source links and clear claim statuses.
- Route contradicted, unsupported, or ambiguous claims to revision or human review before production continues.
- Provide end-to-end automation with human-visible checkpoints and user control.
- Support resumable workflows, per-user data isolation, and observable job execution.
- Keep the MVP cost-aware through provider abstraction and free/low-cost options where practical.

## Core User Experience

### 1. Submit an Idea

Users provide:
- Content idea or topic.
- Optional target audience and tone.
- Target platform (YouTube, TikTok, or both).
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

These labels describe the system's evidence assessment, not a guarantee of truth. Source quality, recency, context, and retrieval coverage can affect the result. A lack of retrieved evidence must not automatically be treated as proof that a claim is false.

### 4. Continue Through Video Production

Once the script passes the configured review gate—or the user explicitly resolves flagged claims—the workflow can continue through:
- Media discovery and curation.
- Voice-over generation.
- Timeline, caption, and audio composition.
- Video rendering.
- Optional manual review and publishing.

The system should preserve traceability from a reviewed claim to its script passage and, where feasible, to the segment in which it appears in the rendered video.

### 5. Track Jobs and Review History

Users can:
- View overall and per-stage progress.
- Inspect logs, outputs, evidence, and errors.
- Watch generated videos and review scripts, media, and audio artifacts.
- See which claims were flagged and how they were resolved.
- Retry a failed stage or revise and rerun a previous job.
- Browse job history with filtering and search.
- View publishing links and timestamps when publishing is enabled.

## End-to-End System Flow

1. User submits an idea through the web app.
2. API validates the request and creates a generation job.
3. The orchestration engine records workflow state and schedules asynchronous work.
4. Planning service generates a content plan and draft script.
5. Claim extraction service identifies factual claims and associates them with script locations.
6. Evidence retrieval service searches permitted external sources and stores source metadata and relevant excerpts.
7. Claim assessment service evaluates each claim against retrieved evidence and assigns one of the defined outcomes.
8. Review gate determines which claims can proceed and which require revision or human review.
9. The script is revised, claims are reassessed when necessary, and review decisions are recorded.
10. Media service gathers licensed or policy-compliant images and video clips.
11. Voice service generates narration from the approved or reviewed script.
12. Composition service assembles the timeline, captions, transitions, and audio.
13. Render service exports video in platform-ready formats.
14. User reviews the final draft; publishing service uploads to supported platforms when requested and configured.
15. Notification and progress services update the UI and send completion alerts.

**Important workflow rule:** fact-checking should happen after a draft script exists and before downstream production is treated as final. The exact gate can be configurable, but flagged claims should remain visible and must not be silently represented as verified.

## Recommended Architecture

Think of e2ecg as a production kitchen with a quality-control station. The planning station drafts the recipe, the fact-checking station checks factual ingredients against evidence, and only then do media, voice, and rendering stations produce the final dish. A workflow coordinator tracks every station, preserves state, and helps a job recover when a station fails.

### Frontend

- **Recommended:** Next.js with TypeScript and App Router.
- Handles idea submission, job progress, claim/evidence review, artifact previews, and job history.
- Server Components can load initial dashboard data; client components can consume live progress updates.

### Backend API

- **Recommended:** NestJS or Fastify on Node.js, deployed separately from the frontend.
- Owns authentication integration, job creation, permissions, API validation, and access to job/artifact data.
- A separate API and worker architecture lets the pipeline scale and deploy independently of the frontend.

### Durable Orchestration

- **Recommended:** Temporal; evaluate Inngest as a lighter alternative.
- Coordinates multi-stage work, persists workflow state, retries transient failures, and resumes eligible jobs after worker or process failures.
- BullMQ with Redis can handle simpler one-step background tasks where durable multi-stage orchestration is unnecessary.

### Worker Services

Use independently deployable worker responsibilities where separation provides practical value:

- Planning and script generation.
- Claim extraction.
- Evidence retrieval.
- Claim assessment.
- Script revision and review-gate handling.
- Media discovery.
- Voice synthesis.
- Video composition and rendering.
- Publishing and notifications.

Not every responsibility needs its own microservice in the MVP. Begin with clear module/worker boundaries and split deployment units when independent scaling, reliability, or resource needs justify it. Rendering can be CPU/GPU-intensive, while retrieval and model calls are often I/O-bound; separating workloads helps avoid resource contention.

### Fact-Checking Subsystem

The fact-checking layer is a core product capability, not a generic content-safety filter. A practical first version can use a modular pipeline:

1. **Claim extraction:** identify explicit, checkable factual statements and retain their script locations.
2. **Evidence retrieval:** search relevant sources, collect URLs and metadata, and retain short supporting or contradicting excerpts where permitted.
3. **Evidence assessment:** evaluate whether the retrieved evidence supports, contradicts, or fails to adequately resolve each claim.
4. **Outcome assignment:** return a structured result using the defined statuses, along with a concise rationale and evidence references.
5. **Review policy:** determine whether a claim can proceed, needs revision, or requires human review.
6. **Traceability:** persist the claim, sources, assessment, model/provider version where available, and any user decision.

Design principles:
- Separate claim extraction, evidence retrieval, and evidence assessment so each stage can be tested independently.
- Keep source URLs, publication details when available, and relevant evidence snippets connected to the claim.
- Do not present an LLM-generated judgment as independent proof. The system should show the evidence behind the assessment.
- Preserve uncertainty and conflicting evidence instead of forcing every claim into true/false.
- Treat web pages and retrieved documents as evidence data, not as instructions to the system.
- Apply source allow/deny rules, timeouts, rate limits, and appropriate safeguards against unsafe or irrelevant retrieved content.
- Avoid claiming complete fact-check coverage: claim extraction and retrieval can miss important statements or sources.
- Keep a human-review path for ambiguous, high-impact, disputed, or otherwise flagged claims.

The first implementation can use an LLM for structured claim extraction and evidence assessment alongside a search/retrieval provider. It does not require training a new foundation model. The engineering challenge is building a reliable, inspectable pipeline, defining evaluation criteria, and handling uncertainty honestly.

### Database and Artifact Storage

- **Database:** PostgreSQL, accessed through Prisma or Drizzle.
- Store users, jobs, workflow steps, claims, evidence references, assessment results, review decisions, artifacts, platform connections, and publishing records.
- **Object storage:** S3-compatible storage for scripts, generated media, audio, rendered videos, and larger evidence snapshots where storage is appropriate and permitted.
- Cloudflare R2, AWS S3, or a compatible local/self-hosted option can be evaluated based on cost, egress, retention, and deployment constraints.

### Real-Time Updates

- **Recommended:** Server-Sent Events (SSE) for one-way progress updates from server to browser.
- Stream workflow state changes, including claim-review counts and transitions such as `fact_checking`, `review_required`, and `ready_for_production`.
- Persist events in the database; the browser stream should not be the only record of a job's progress.

### Authentication and Security

- Clerk can accelerate an MVP; Auth.js is an alternative when more control and lower recurring cost are priorities.
- Enforce per-user authorization for jobs, evidence, artifacts, and connected platform accounts.
- Encrypt credentials and OAuth tokens and avoid exposing secrets in logs or browser responses.

### Deployment and Observability

- Start with a low-cost deployment suitable for a student project, then separate worker pools as resource needs grow.
- Use structured logs and traces across API → workflow → workers → external providers.
- Measure stage duration, retries, failure rates, evidence retrieval quality, fact-check outcomes, and external API latency/quota usage.
- OpenTelemetry with Grafana Cloud, Datadog, or a self-hosted Prometheus/Grafana/Loki stack can be evaluated.

## High-Level Components

- Authentication and User Accounts
- Idea Intake and Prompt Builder
- AI Planning and Script Generation
- Claim Extraction
- Evidence Retrieval and Source Metadata
- Claim Assessment and Outcome Assignment
- Claim Review and Script Revision
- Media Discovery and Curation
- AI Voice Synthesis
- Video Composition and Rendering
- Platform Publishing Integrations
- Progress Timeline and History UI
- Notifications and Activity Feed
- Orchestration and Workflow State
- Provider Abstraction and Configuration

## Recommended Tech Stack

The stack should support interchangeable providers so the project can be developed and demonstrated with free or low-cost services, while allowing paid providers to be configured later. Actual free-tier limits and provider terms should be checked during implementation; “free” does not necessarily mean unlimited or production-reliable.

| Layer | Paid / Managed Option | Free / Low-Cost Option | Purpose |
|---|---|---|---|
| Frontend | Next.js hosted on a managed platform | Same code on a free tier or self-hosted | Submission, progress, evidence review, and history UI |
| Backend API | Managed container hosting | Self-hosted container or low-cost VM | Job API, authorization, and application logic |
| Orchestration | Temporal Cloud or managed workflow provider | Self-hosted Temporal, or a simpler queue for a reduced MVP | Durable multi-step workflows and retries |
| Lightweight queue | Managed Redis with BullMQ | Local/self-hosted Redis | Simple background tasks |
| Database | Managed PostgreSQL | Free-tier PostgreSQL provider or self-hosted PostgreSQL | Jobs, claims, evidence metadata, review decisions, and users |
| Object storage | Cloudflare R2 or AWS S3 | Eligible free tier or self-hosted MinIO | Scripts, media, audio, and video artifacts |
| Real-time updates | SSE | SSE | Live workflow and review progress |
| Auth | Clerk or another hosted provider | Auth.js with application-managed sessions | Account and session management |
| Planning / script LLM | Hosted LLM API | Provider free tier or local open-weight model where practical | Structured content planning and script generation |
| Claim extraction / assessment | Hosted LLM API | Reuse an available model/provider or run a suitable local model | Claim identification and evidence-based classification |
| Evidence retrieval | Search API or licensed data provider | Search providers with usable free tiers, or limited manual/source-specific retrieval for the MVP | Find sources relevant to each claim |
| Media discovery | Pexels/Pixabay or other permitted stock APIs; optional image generation | Stock APIs and user-provided assets | Source visuals with license/provenance metadata |
| Voice synthesis | Hosted TTS API | Local TTS such as Piper or another suitable free option | Narration |
| Video composition/render | Remotion and FFmpeg | Remotion and FFmpeg, subject to applicable license terms | Compose and export video |
| Publishing | Official YouTube/TikTok APIs, subject to access requirements | Same official APIs, subject to quota and review requirements | Optional publishing |
| Notifications | Managed email provider | Eligible free tier or SMTP provider | Completion and failure alerts |
| Observability | Managed monitoring platform | Self-hosted Prometheus/Grafana/Loki | Debugging and reliability metrics |

Provider choices are implementation decisions rather than fixed product requirements. Keep external services behind small interfaces where switching providers is realistic. Avoid building a broad abstraction framework before the first end-to-end workflow works.

## Deployment Modes: Free vs. Paid

The system should support a cost-conscious development mode and a managed mode without duplicating the business logic.

- **Free / low-cost mode:** use free tiers, local tools, or self-hosted open-source components where practical. Trade-offs include resource limits, extra operations work, weaker model quality in some cases, and possible single points of failure.
- **Paid / managed mode:** use managed databases, storage, orchestration, model APIs, and hosting where their reliability or quality justifies the cost.

Keep provider-specific behavior behind interfaces for services that are likely to change, such as LLM, evidence search, TTS, storage, and orchestration. Configure providers through environment variables or a local configuration command. Never put API keys in source control or expose them to the client.

The target is a demonstrable student MVP with tightly controlled costs, not a large production fleet. Validate actual provider pricing, rate limits, licensing, and terms before enabling a service.

## Conceptual Data Model

### User
- `id`
- `email`
- Authentication provider ID or password hash, depending on auth design
- Profile metadata
- `created_at`

### Job
- `id`
- `user_id`
- Original idea and generation constraints
- Target platforms
- Overall status
- Created/updated timestamps

### Job Step / Workflow Event
- `id`
- `job_id`
- Step name and status
- Start/finish timestamps
- Structured logs, error details, retry metadata
- Event timestamp and correlation information

### Script Version
- `id`
- `job_id`
- Version number
- Script/content-plan payload
- Revision reason and review state
- Created timestamp

### Claim
- `id`
- `job_id`
- `script_version_id`
- Claim text
- Script location (for example, paragraph, sentence, or segment index)
- Claim type and extraction metadata when available
- Fact-check status
- Assessment rationale
- Created/updated timestamps

### Evidence Source
- `id`
- `claim_id`
- Source URL
- Source title, publisher, publication date, and retrieval timestamp when available
- Relevant excerpt or evidence summary, subject to source terms
- Evidence direction/relevance and quality notes
- Content hash or other deduplication metadata when useful

### Claim Review / Decision
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
- Type (script, evidence report, image, clip, voice, rendered video)
- URI and metadata

### Platform Account
- `id`
- `user_id`
- Provider (YouTube or TikTok)
- Encrypted OAuth credentials
- Token expiry and connection metadata

### Publish Record
- `id`
- `job_id`
- Provider
- Remote content ID and URL
- Published timestamp
- Response payload summary

The schema is conceptual. Normalize it based on actual access patterns, audit requirements, and the MVP's chosen persistence strategy.

## API Surface (Draft)

### Jobs and Workflow
- `POST /auth/register`
- `POST /auth/login`
- `POST /jobs`
- `GET /jobs`
- `GET /jobs/:id`
- `GET /jobs/:id/steps`
- `GET /jobs/:id/artifacts`
- `GET /jobs/:id/stream` — SSE stream for live progress
- `POST /jobs/:id/retry`

### Claims and Evidence
- `GET /jobs/:id/claims`
- `GET /claims/:id`
- `GET /claims/:id/evidence`
- `POST /claims/:id/review` — record a review decision
- `POST /jobs/:id/recheck` — rerun checks after a script revision, if supported

### Publishing and Integrations
- `POST /jobs/:id/publish`
- `GET /integrations`
- `POST /integrations/:provider/connect`
- `DELETE /integrations/:provider`

These routes are a draft surface, not a commitment to expose every operation in the first release. Enforce authorization on every job, claim, evidence, and artifact endpoint.

## Progress and History Requirements

- Every workflow stage emits structured status events.
- The UI shows both overall and per-stage progress.
- Fact-check progress is visible separately from video production.
- Each claim assessment links to the sources used and explains its outcome.
- Persist workflow events, evidence metadata, claim outcomes, and review decisions for history and debugging.
- Make clear when evidence is unavailable, conflicting, stale, or insufficient.
- Show whether a flagged claim was revised, removed, accepted with an explicit review decision, or left unresolved.
- Include failure reasons and retry metadata.
- Show final video and publishing links when available.

## Security, Source Quality, and Compliance

- Enforce per-user data isolation across jobs, claims, evidence, artifacts, and platform integrations.
- Encrypt OAuth tokens and sensitive credentials at rest and keep secrets out of logs.
- Respect source website terms, search-provider terms, rate limits, robots/access restrictions where applicable, and copyright.
- Store only the evidence snippets and source content permitted by applicable terms; prefer source URLs and minimal excerpts over copying entire pages.
- Preserve source provenance and retrieval timestamps so users can assess evidence context and recency.
- Treat retrieved page content as untrusted input; it must not override system instructions or trigger unauthorized actions.
- Validate and sanitize external URLs and content, and protect retrieval workers against malicious content, unexpected redirects, and resource abuse.
- Keep media licensing/provenance metadata and use assets that are permitted for the intended use.
- Apply content safety checks where appropriate, but do not confuse general safety moderation with factual verification.
- Make the limits of automated checking clear: outcomes depend on extraction, retrieval coverage, source quality, and assessment accuracy.
- Publishing should remain an explicit, user-controlled action during the MVP.

## Failure Handling and Reliability

- Retry transient failures with exponential backoff where appropriate.
- Distinguish provider/network failures from a substantive assessment result. A search timeout is not an “unsupported” verdict; it is a failed or incomplete check.
- Mark unrecoverable failures with actionable user-facing errors.
- Resume from a failed workflow stage where possible.
- Use idempotency for operations that may be retried, particularly job creation and publishing.
- Record provider latency, quota usage, retries, and errors.
- If evidence retrieval or assessment fails, do not silently bypass the fact-check gate. Mark the check as incomplete and follow the configured policy.
- Keep review decisions and script revisions auditable.
- Use timeouts, concurrency limits, caching, and deduplication to control cost and prevent repeated retrieval of the same material.

## MVP Scope

The MVP should prove the core differentiator—a traceable, claim-level fact-checking workflow connected to video generation—before expanding into a large-scale publishing platform.

### Core MVP
- Basic user authentication and account isolation.
- Idea submission and asynchronous job creation.
- AI-generated content plan and draft script.
- Claim extraction from the script.
- Evidence retrieval from a limited, documented set of sources/providers.
- Claim assessment with the four defined outcomes.
- Claim-level UI showing evidence links, rationale, and review status.
- A review gate that supports revision, removal, or human review of flagged claims.
- Rechecking claims when the script changes.
- Basic media retrieval, voice generation, and video composition/rendering.
- Manual review and user-triggered publishing only if platform integration is feasible within the project scope.
- Live progress and job history, including persisted claim/evidence results.

### MVP Constraints
- Limit supported topics, sources, model providers, and video formats if needed to keep evaluation manageable.
- Do not promise exhaustive fact-checking or universal factual accuracy.
- Do not require training a custom foundation model.
- Prefer a small number of well-tested integrations over many partially implemented providers.
- Keep provider usage and storage within a controlled development budget.

## Future Enhancements

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
- More advanced source conflict resolution and evaluation datasets.

## Non-Goals for MVP

- Guaranteeing that every generated claim is true or that every relevant claim is found.
- Replacing expert review for high-impact or specialized topics.
- Building a new foundation model or training a model on a large viral-video corpus.
- Competing with professional timeline editors or attempting a full professional editing suite.
- Deep social analytics and attribution.
- Enterprise organization management.
- Fully autonomous publishing without a user-controlled review/approval path.

## Development Roadmap

1. **Foundation:** repository setup, configuration, auth, PostgreSQL schema, and job lifecycle.
2. **Workflow and progress:** orchestration, structured events, retries, and SSE updates.
3. **Planning:** structured AI content plan and script generation.
4. **Claim extraction:** identify factual claims and map them to script locations.
5. **Evidence retrieval:** integrate a limited set of sources and persist source metadata.
6. **Claim assessment:** implement the four outcomes, rationale, and evidence-linked result format.
7. **Review gate:** build the claim review UI, script revision loop, and reassessment workflow.
8. **Evaluation:** create a small test set of claims with known evidence/outcomes; evaluate extraction coverage, assessment quality, evidence relevance, and failure handling.
9. **Media pipeline:** integrate media retrieval and provenance.
10. **Voice and rendering:** generate narration, compose, and render a short video.
11. **End-to-end integration:** preserve claim-to-script traceability through the generated artifact where feasible.
12. **Hardening:** permissions, rate limits, retries, observability, cost tracking, and error handling.
13. **Optional publishing:** integrate official platform APIs after the review and generation path is stable.

## Evaluation and Success Metrics

Measure both product workflow performance and the quality of the fact-checking subsystem.

### Workflow Metrics
- Time from idea submission to first reviewable script/video draft.
- Job completion rate.
- Failure rate and retry count per workflow stage.
- Video render success rate.
- Publishing success rate when publishing is enabled.
- External API latency, quota use, and estimated cost per job.

### Fact-Checking Metrics
- **Claim extraction coverage:** proportion of checkable claims in an annotated sample that the system identifies.
- **Evidence relevance:** whether retrieved sources actually address the corresponding claim.
- **Assessment quality:** agreement between system outcomes and a documented human-reviewed reference set.
- **Unsupported vs. contradicted distinction:** whether the system avoids treating lack of evidence as proof of falsity.
- **Uncertainty handling:** whether conflicting or insufficient evidence is surfaced rather than forced into a confident conclusion.
- **Traceability:** proportion of assessments with accessible source references and a rationale.
- **Review-gate behavior:** whether flagged or incomplete checks are handled according to the configured policy.
- **Recheck consistency:** whether revised claims are reassessed and prior decisions remain auditable.

For an academic project, document the test-set construction, annotation procedure, scoring method, and limitations. Avoid claiming the system is “accurate” based only on a few hand-picked demonstrations.

## Risks and Mitigations

| Risk | Why it matters | Mitigation |
|---|---|---|
| Claim extraction misses a factual statement | A claim that is never extracted cannot be checked by the pipeline. | Evaluate against annotated scripts; show users the extracted claims and state coverage limitations. |
| Retrieval finds weak or irrelevant sources | The assessment may appear evidence-based while relying on poor evidence. | Show source links, measure relevance, use source-quality guidance, and support further review. |
| Conflicting or outdated evidence | Claims may be time-sensitive or legitimately disputed. | Preserve retrieval dates, show conflicts, and use an insufficient/conflicting outcome. |
| LLM assessment overstates certainty | Users may mistake generated reasoning for proof. | Require evidence-linked results, constrained structured outputs, calibrated language, and human review. |
| Search/API costs or quotas grow | Fact-checking may require multiple retrieval calls per script. | Limit claims and retrieval depth, cache results, deduplicate sources, add rate limits, and track per-job cost. |
| External providers fail | A check may be incomplete even when the rest of the workflow works. | Retry transient errors and mark failed checks as incomplete instead of silently bypassing the review gate. |
| Pipeline becomes too complex for the project timeline | Many workers and integrations can delay a working demonstration. | Build a vertical slice first; keep module boundaries clear without prematurely splitting everything into separate services. |
| Copyright or terms restrict evidence storage | Saving full pages or excerpts may violate source/provider rules. | Store URLs and metadata, keep excerpts minimal, and verify terms before implementation. |
| Rendering and model calls exceed the budget | Video generation can consume more resources than expected. | Set per-job limits, choose short output formats, use local/free options selectively, and measure costs early. |

## Project Name

**e2ecg** means **End-to-End Content Generation**. The expanded project positioning is **Evidence-Grounded AI Video Generation and Fact-Checking Platform**.

## Current Status

Planning and specification stage. The next implementation milestone should be a small end-to-end vertical slice:

**idea → draft script → extracted claims → retrieved evidence → claim assessments → review/revision → approved script → simple rendered video**

This slice should demonstrate the project's main contribution before optional platform publishing and broader provider support are added.
