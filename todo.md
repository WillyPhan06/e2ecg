# e2ecg — Development To-Do

Version 1.2 · Derived from `README.md` (v1.2) and `ARCHITECTURE.md` (v1.2). Scope: **publicly deployed on free-tier cloud platforms**, zero recurring cost, with a local Docker Compose stack for development.

Phases are ordered so each one produces something runnable before the next begins, like building a house from foundation to roof rather than framing every room at once. **Phase N here is step N of the README's Development Roadmap**, so the three documents share one numbering. Every phase ends in a demoable state, even if the demo is rough.

Checkbox legend: `[ ]` not started · `[~]` in progress · `[x]` done
Relative size (solo developer): **S** about 1–2 days · **M** about 3–5 days · **L** a week or more. Sizes are rough and exist to help ordering, not to promise dates.

## How to Use This File

- References like *(ARCH §3.3)* point to sections of `ARCHITECTURE.md`; *(README: …)* points to a README heading.
- **Cross-cutting rules for every phase** (a task is not done unless these hold):
  - Every limit is configuration (ARCH §10.4), never a hard-coded constant.
  - Every call to a paid-or-quota-limited provider writes a `usage_records` row (ARCH §5, §14).
  - No secrets in the repo, logs, or browser responses.
  - New behavior has a focused test; new failure modes have an explicit user-facing message.
  - If a free-tier limit or term turns out to differ from `docs/free-tier-limits.md`, update that file and the README's Free-Tier Reality Check the same day.

## MVP Traceability (README MVP Scope → Phase)

| README Core MVP item | Phase(s) |
|---|---|
| Authentication and account isolation | 2 |
| Idea submission and asynchronous job creation | 3 |
| AI content plan and draft script | 4 |
| Claim extraction | 5 |
| Evidence retrieval (limited, documented sources) | 6 |
| Claim assessment with four outcomes + check state | 7 |
| Documented evaluation set and scoring | 8 |
| Claim-level UI, review gate, recheck | 9 |
| Script-level content safety check | 10 |
| Media, voice | 10 |
| Composition, rendering, preview, download | 11 |
| Live progress and job history | 3, 9, 11, 12 |
| Quota controls, retention, at-capacity messaging | 3, 4, 12 |
| Public deployment on the free-tier stack + local dev stack | 1, 3, 12, 13 |

---

## Phase 1 — Foundation & Accounts  ·  Size M

*The slab and the permits: repo layout, local stack, CI, and, most importantly, getting the free accounts and the VM requested on day one, because free Arm capacity can take days or weeks to obtain.*

**Do first (week 1):**
- [ ] Request the Oracle Always Free Arm VM (ARCH §11.2). Retry across availability domains or use capacity-retry automation; note the home-region constraint. Record the outcome date.
- [ ] Create Vercel (Hobby), Neon, Cloudflare R2, GitHub Container Registry, and an external uptime-monitor account
- [ ] Create `docs/free-tier-limits.md`: the verified allowances and terms (UI host, VM, database, object storage, LLM, search, stock media), each with a "checked on" date (README: Free-Tier Reality Check)

**Repository and tooling:**
- [ ] Initialize monorepo (pnpm workspaces or Turborepo): `apps/frontend`, `apps/api`, `apps/workers`, `packages/shared`, `eval/`
- [ ] Shared TypeScript config, ESLint, Prettier across packages
- [ ] Define the full environment variable schema in `.env.example` (ARCH §10.4), including all limit variables and the reserved `DEPLOYMENT_MODE=free`
- [ ] Build `scripts/configure.ts` / `npm run configure` (ARCH §10.3)
  - [ ] Choose target: local dev or deployed free-tier stack
  - [ ] Write `.env.local` (or an `.env.production` template) without overwriting unrelated values
  - [ ] Lightly validate each provider key, the Neon connection, and R2 reachability
  - [ ] Derive a suggested `MAX_JOBS_PER_DAY` from the entered daily LLM limit and ~30 calls per job (ARCH §14)
- [ ] Write `docker-compose.dev.yml`: app Postgres, MinIO, Temporal + its Postgres, API, worker (ARCH §11.6)
- [ ] Write the `docker-compose.free.yml` skeleton for the VM: Caddy, API, worker, Temporal, Temporal Postgres, Temporal UI under a profile (ARCH §11.5)
- [ ] CI (GitHub Actions): lint, typecheck, unit tests on every PR
- [ ] CI: build multi-arch (`linux/amd64`, `linux/arm64`) images and push to GHCR; confirm every base image has an `arm64` build
- [ ] Dev-setup doc (`CONTRIBUTING.md`) pointing to `npm run configure`

**Exit criteria:** a fresh clone can run `npm run configure` (local), `docker compose -f docker-compose.dev.yml up`, and reach a "Hello World" page on `localhost`; accounts exist; `docs/free-tier-limits.md` is filled in; and the VM is either provisioned or has a dated, retrying request plus a decision date for the local-worker fallback (ARCH §16).

---

## Phase 2 — Auth & Data Foundations  ·  Size M

*Before anything can be a "job," the app needs to know who is asking and somewhere durable to write things down.*

- [ ] Auth.js in the Next.js app issuing signed session tokens (ARCH §3.1); provider choice (email/password vs OAuth) recorded in the repo
- [ ] API `AuthModule`: verify token signature and expiry using `AUTH_SECRET`; attach `userId` to every request context; `GET /health` is the only unauthenticated route (ARCH §3.2)
- [ ] Postgres schema + migrations (Prisma or Drizzle) for: `users` (+ Auth.js adapter tables), `jobs`, `job_steps`, `job_events`, `script_versions`, `claims`, `claim_assessments`, `evidence_sources`, `assessment_evidence`, `claim_reviews`, `artifacts`, `usage_records` (ARCH §5)
  - [ ] `claim_assessments` check constraint: an outcome exists if and only if `check_state = 'completed'`
  - [ ] Unique constraints on `job_steps(job_id, step_name, attempt)` and `script_versions(job_id, version_number)`
- [ ] Indexes from ARCH §5 (history, events, claims, assessments, evidence, artifacts and expiry, usage)
- [ ] Configure pooled (app) vs direct (migrations) Neon connections; add a short retry for connection errors to absorb cold starts (ARCH §5, §7.4)
- [ ] Next.js app shell: layout, navigation, auth-aware routing (`/dashboard`, `/jobs/new`, `/jobs/[id]`, `/settings`)
- [ ] Registration/login flow wired to Auth.js
- [ ] Enforce per-user isolation at the query layer for every table that carries job-scoped data (ARCH §12; README: Security, Source Quality, and Compliance)
- [ ] Test: user A can never read or modify user B's jobs, claims, evidence, or artifacts

**Exit criteria:** a user can sign up, log in, and see an empty dashboard scoped to their account, locally and against the real Neon database.

---

## Phase 3 — Walking Skeleton, Deployed  ·  Size L

*The expo station goes up, and the food truck is driven onto the street, before any real cooking happens. Deploying a stubbed pipeline now means capacity, TLS, and CORS surprises show up while they are cheap to fix.*

**Orchestration core (ARCH §3.3, §3.4):**
- [ ] Stand up self-hosted Temporal with its own Postgres via Compose
- [ ] Define `VideoGenerationWorkflow` with **stubbed activities**, the `stage()` wrapper, and the review loop skeleton, using canonical job states (README: Canonical Job States)
- [ ] `stage()` behavior: record start/finish; on exhausted retries set `failed` and wait for `retryStage` or `cancelJob` signals
- [ ] Define task queues (`e2ecg-planning`, `e2ecg-fact-checking`, `e2ecg-media`, `e2ecg-voice`, `e2ecg-composition`, `e2ecg-render`) polled by **one worker process**; set render concurrency to 1
- [ ] Helper activities `recordStep` and `setJobState` that write `job_steps`, `job_events`, and `jobs.status` in one transaction (no DB writes in workflow code)
- [ ] Pass references (IDs, storage keys), never payload bodies, between activities
- [ ] Start options with explicit workflow ID conflict and reuse policies; integration test for duplicate starts (ARCH §7.1)

**API and quota (ARCH §3.2, §6, §14):**
- [ ] `JobsModule`: `POST /jobs` (validate, check kill switch and daily quotas, insert `Job`, start workflow `job:{jobId}`); 429 `at_capacity` with `resetsAt` and 503 `intake_paused`
- [ ] `GET /jobs`, `GET /jobs/:id`, `GET /jobs/:id/steps`
- [ ] `QuotaModule`: `GET /quota` (remaining per-user and global capacity, next reset using `QUOTA_RESET_TZ`)
- [ ] `HealthModule`: `GET /health` with database, Temporal, and storage checks
- [ ] `POST /jobs/:id/cancel`, `POST /jobs/:id/retry` (both paths), `DELETE /jobs/:id` (ARCH §6, §7.3)
- [ ] `StreamModule`: SSE from persisted `job_events` with `Last-Event-ID` replay, keep-alive comments, polling only while connected, **no Redis and no `LISTEN/NOTIFY`** (ARCH §9)
- [ ] `S3StorageProvider` against MinIO (local) and R2 (deployed); signed read URLs (ARCH §10.1)
- [ ] `ReconciliationWorkflow`: mark jobs with no live workflow as `failed: workflow_lost` (ARCH §3.5, §7.3)

**Frontend:**
- [ ] Idea submission form (`/jobs/new`) with the README Submit Idea fields (idea, audience, tone, output format, constraints) and the remaining daily quota
- [ ] Live progress view (`/jobs/[id]`) using `EventSource` against the API origin, falling back to `GET /jobs/:id/steps` on failure
- [ ] "At capacity, resets at …" and "intake paused" states instead of generic errors

**Deploy it for real (ARCH §11):**
- [ ] Provision the VM from a script (Docker, swap file, firewall, root-only `.env`); expose only ports 80/443; Temporal UI on localhost via SSH tunnel
- [ ] Caddy with TLS for the API hostname (free subdomain, or Cloudflare Tunnel if a domain is owned); CORS limited to the UI origin
- [ ] Deploy the UI on Vercel Hobby; the API and worker on the VM; connect Neon and R2
- [ ] CI/CD deploy workflow: pull the arm64 images from GHCR and restart over SSH
- [ ] Uptime monitor on `GET /health`
- [ ] Production smoke test: submit a job on the public URL and watch stubbed stages complete

**Exit criteria:** submitting an idea on the **public** deployment creates a job that visibly walks through stubbed stages in real time, survives a browser refresh (replay via `Last-Event-ID`), and shows a clear at-capacity state when the daily limit is reached. No real AI calls yet.

---

## Phase 4 — AI Planning & Script Generation  ·  Size M

*The first real station comes online: turning a one-line idea into a structured script the rest of the pipeline can build on, with every call counted against the free quota.*

- [ ] Define `LLMProvider` (generic `generateStructured`) and the stage services `ScriptPlanner`, `ScriptReviser` (ARCH §10.1)
- [ ] Implement the primary free-tier hosted provider (final choice per Open Decisions, ARCH §16)
- [ ] Implement `QuotaGuardedLLM`: daily-ceiling check, `usage_records` row per call, stage-to-model routing, fallback on rate or daily limits, non-retryable `QuotaExhaustedError` (ARCH §10.1, §14)
- [ ] Implement an optional second free provider as the fallback (the one sanctioned exception to one-implementation-per-interface)
- [ ] Prompt templates incorporating audience, tone, duration, CTA, and language constraints; script segments carry stable `segmentId`s
- [ ] Implement `planContent` and `draftScript` activities on `e2ecg-planning`
- [ ] Validate output against a typed `Script` schema; retry-with-repair on malformed output
- [ ] Persist each script as JSON in object storage (`script/v{n}.json`) and as a `script_versions` row with `content_key`, plus an `artifacts` row with `expires_at` (ARCH §5, §8)
- [ ] Unit tests: prompt → schema-valid output; quota-ceiling behavior; fallback routing
- [ ] Frontend: display the plan and script in the job detail view once available
- [ ] Cap script length and estimated duration against `MAX_VIDEO_SECONDS`

**Exit criteria:** a submitted idea produces a real, structured script visible in the UI on the deployed stack, usage is recorded, and a simulated exhausted quota produces a clear job failure or at-capacity message rather than a crash.

---

## Phase 5 — Claim Extraction  ·  Size M

*Decompose the script into the specific statements that can be checked, and be honest about the ones that cannot.*

- [ ] Define typed contracts: `ExtractedClaim`, claim taxonomy (`statistic`, `date_event`, `attribution`, `causal_scientific`, `other`), non-checkable reasons (`opinion`, `prediction`, `subjective`, `other`)
- [ ] Implement the `ClaimExtractor` stage service and the `extractClaims` activity on `e2ecg-fact-checking`: identify explicit checkable statements, preserve `segmentId` and in-segment location, label non-checkable statements instead of dropping them
- [ ] Persist `claims` rows with extraction metadata and coverage limitations
- [ ] Enforce `MAX_CLAIMS_PER_JOB`: extras are recorded as `skipped_by_limit` and shown to the user
- [ ] Claim-level read endpoints: `GET /jobs/:id/claims`, `GET /claims/:id` (ARCH §6), scoped by owner
- [ ] Tests: extraction on a small fixture set (statistics, dates, attributions, opinions, predictions); location mapping; cap behavior

**Exit criteria:** a draft script produces an inspectable list of checkable and non-checkable claims tied to script segments.

---

## Phase 6 — Evidence Retrieval  ·  Size M

*Find sources worth trusting, within a free allowance, without ever letting a retrieved web page give the system orders.*

- [ ] **Decide and document the first evidence sources** (candidates: Wikipedia/Wikidata APIs, a fact-check claim-search API, one free-tier web search API). For each, verify the current free allowance and terms; record them in `docs/free-tier-limits.md` (ARCH §16)
- [ ] Define `EvidenceProvider` and per-source allow/deny rules (ARCH §10.1)
- [ ] Implement the provider adapters and a result-quality filter
- [ ] Build the **SSRF-safe fetcher**: `http`/`https` only; resolve and pin IPs; block private, loopback, link-local, instance-metadata, and Docker network ranges, including after redirects; cap redirects, size, and time (ARCH §12)
- [ ] Implement `retrieveEvidence`: store URL, title, publisher, publication and retrieval dates, minimal permitted excerpts, direction, quality notes, and content hash; process page text **transiently**
- [ ] Cache by normalized query and content hash; enforce `MAX_SOURCES_PER_CLAIM`, per-request timeouts, and `SEARCH_DAILY_LIMIT` with `usage_records`
- [ ] Record retrieval failures as incomplete checks with a `failure_reason`; retry transient failures; never silently continue
- [ ] `GET /claims/:id/evidence` endpoint, scoped by owner
- [ ] Tests: SSRF cases (private IP, redirect to private IP, metadata address, oversized body), source allow/deny rules, cache hits, timeout handling

**Exit criteria:** claims come back with source metadata and permitted excerpts; SSRF tests pass; retrieval failures are recorded as incomplete checks rather than empty results.

---

## Phase 7 — Claim Assessment  ·  Size M

*The core differentiator: assess each claim only against the evidence in hand, and say so plainly when the check could not run.*

- [ ] Define typed contracts for `ClaimAssessment`, the four outcomes (`supported`, `contradicted`, `unsupported`, `insufficient_or_conflicting`), and the separate check state (`completed`, `incomplete`, `skipped_by_limit`)
- [ ] Implement the `ClaimAssessor` stage service: strict output schema, **no tools**, retrieved text delimited and labeled as data, off-schema output rejected (ARCH §12)
- [ ] Implement `assessClaims`: structured outcome plus concise rationale; link the sources actually used through `assessment_evidence`
- [ ] Distinguish `unsupported` from `contradicted`; never treat missing evidence as proof of falsity
- [ ] Record provider, model, prompt version, retrieval timestamps, failure reasons, and uncertainty metadata
- [ ] Quota-aware behavior: on `QuotaExhaustedError` mark the affected checks `incomplete` (`failure_reason = quota_exhausted`); try the fallback provider first
- [ ] Reuse prior assessments for unchanged claims across rounds (`reused_from_assessment_id`) (ARCH §3.3)
- [ ] Append-only assessments: a recheck inserts new rows and never edits old ones
- [ ] Implement `applyReviewGate`: supported claims proceed under policy; flagged, ambiguous, `incomplete`, and `skipped_by_limit` claims route to revision or human review
- [ ] Tests: outcome classification on fixtures, unsupported-vs-contradicted distinction, uncertainty and conflict handling, prompt-injection fixtures (a page that tries to instruct the model), incomplete-check handling, gate behavior

**Exit criteria:** a draft script yields assessed claims with rationale and evidence links; failures show as "Check incomplete"; the gate flags everything it should.

---

## Phase 8 — Evaluation (early, not last)  ·  Size M

*Measure the differentiator before the pipeline around it grows. A smoke detector should be tested with smoke.*

- [ ] Write the annotation guide: what counts as a checkable claim, how to label each outcome, how to label acceptable evidence (ARCH §15)
- [ ] Build `eval/dataset/*.jsonl`: roughly 50–100 annotated claims across short scripts; two annotators where possible; record agreement
- [ ] Implement the record/replay cache for retrieval and LLM responses so repeated runs do not burn quota
- [ ] Implement `eval/run.ts`: runs extraction, retrieval, and assessment directly (not through Temporal)
- [ ] Metrics: extraction coverage and precision; evidence relevance; assessment accuracy and per-class precision/recall; unsupported-vs-contradicted confusion; uncertainty handling; traceability (README: Evaluation and Success Metrics)
- [ ] Fault-injection runs: timeouts, quota errors, malformed model output; confirm failures become `incomplete`, never outcomes
- [ ] Produce the baseline report in `eval/results/` and a short write-up of method and limitations
- [ ] Use the results to settle the LLM primary/fallback choice and the claim cap (ARCH §16)

**Exit criteria:** a documented, repeatable evaluation exists with a baseline score, and the choice of provider and models is justified by it.

---

## Phase 9 — Review Gate & Revision Loop  ·  Size L

*Flagged claims stop the line until a person decides, and every decision leaves a permanent record.*

- [ ] `POST /claims/:id/review` (`accept`, `revise`, `remove`, `request_review`): immutable `claim_reviews` rows; send `reviewSubmitted` once every flagged claim in the round has a decision (ARCH §6)
- [ ] `POST /jobs/:id/recheck` with optional user script edits; reject past `MAX_REVISION_ROUNDS` with a clear message
- [ ] Implement `reviseScript` (decisions and optional edits → new `script_versions` row); persist the revision reason
- [ ] Complete the workflow review loop: revise → re-extract → re-retrieve → re-assess, until clear, cancelled, or at the round cap; at the cap only "accept remaining" or cancel is offered (ARCH §3.3)
- [ ] Claim-level review UI: claim text and script location, sources and excerpts, rationale, limitations, outcome or "Check incomplete", suggested next action
- [ ] Show revision history and the resolution of each flagged claim (revised, removed, explicitly accepted, unresolved)
- [ ] Emit fact-check progress, claim, flagged, and incomplete counts, and `review_required` transitions through `job_events` (ARCH §9)
- [ ] Tests: loop termination, round cap, decision immutability, recheck consistency, reuse of unchanged assessments, signal handling after a worker restart

**Exit criteria:** flagged or incomplete checks stop production; a user can resolve or revise them; the revised script is reassessed with prior results preserved; nothing proceeds silently.

---

## Phase 10 — Content Check, Media & Voice  ·  Size M

*Check the approved script before spending media, voice, and render resources; then run casting (media) and narration (voice) in parallel.*

- [ ] Implement `contentCheck` on `e2ecg-planning`: strict-schema moderation of the approved script through the quota-guarded LLM (or a free moderation endpoint if available); runs after `ready_for_production` and before any production stage (ARCH §12)
- [ ] Define `MediaProvider`; implement Pexels and Pixabay adapters with license/provenance metadata per asset
- [ ] Optional: user-provided assets via `POST /uploads` with type and size limits and signed upload URLs
- [ ] Implement `collectMedia`: stock search first, user assets where provided, license tags recorded; no generative images in this version
- [ ] Define `TTSProvider`; implement `PiperTTSProvider` (local on the VM); confirm the Piper build and voice model on `arm64`
- [ ] Implement `generateVoice`: synthesize **per script segment**, respect the `language` constraint, record per-segment durations
- [ ] Confirm the workflow runs `collectMedia` and `generateVoice` concurrently, and only after the content check passes (ARCH §3.3)
- [ ] Persist media and voice artifacts with metadata (duration, resolution, provider, license) and `expires_at`
- [ ] Frontend: media thumbnails and an audio player in the job detail view as artifacts land
- [ ] Tests: content-check pass/fail routing, license metadata present on every asset, parallel execution

**Exit criteria:** an approved script passes the content check and produces real narrated audio and a matched set of correctly licensed media, on the deployed stack.

---

## Phase 11 — Composition & Rendering  ·  Size L

*The most resource-hungry station on the line, running on two cores shared with everything else. Start deliberately simple and measure on the real VM.*

- [ ] Define the `RenderSpec` schema: ordered segments (`segmentId`, media ref, audio-derived duration), caption cues, audio refs, output preset
- [ ] Implement `composeTimeline` on `e2ecg-composition`: build the `RenderSpec` from per-segment audio durations; enforce `MAX_VIDEO_SECONDS` (ARCH §3.4)
- [ ] Implement `renderVideo` on `e2ecg-render` with FFmpeg: scale/crop to the preset, simple motion for stills, concatenation, burned-in captions, narration mix, H.264 + AAC with fast-start; cap threads and lower scheduling priority
- [ ] Heartbeat to Temporal and emit coarse progress by parsing FFmpeg progress output; set start-to-close and heartbeat timeouts
- [ ] Export captions (`.srt`) and the **claim-to-segment map** alongside the final video (ARCH §8)
- [ ] Store `final.mp4`, captions, and the map under `render/` with `expires_at`
- [ ] Mark the output as AI-generated in the UI and the file metadata (README: Security, Source Quality, and Compliance)
- [ ] Frontend: video preview player, download via signed R2 URL, claim-to-segment view, and the expiry date
- [ ] **Measure on the real VM:** render time and peak memory per preset, with and without a concurrent API load; fill in the table in ARCH §11.3 and set the default resolution and duration cap from the data
- [ ] Kill the worker mid-render and confirm Temporal resumes the activity without data loss

**Exit criteria:** a job produces a finished, watchable, captioned video that can be previewed and downloaded from the public deployment, with measured render time and memory recorded.

---

## Phase 12 — Hardening  ·  Size L

*Turn "it works" into "it keeps working on free tiers": quotas that match reality, data that expires, backups that restore, and failures that explain themselves.*

**Quotas, retention, backups (ARCH §3.5, §8, §14):**
- [ ] Re-derive `MAX_JOBS_PER_DAY` and per-user limits from the current provider quotas; document the arithmetic
- [ ] `RetentionSweepWorkflow`: delete expired artifacts, mark jobs `expired`; add a bucket lifecycle backstop set slightly longer than the app TTL
- [ ] `DatabaseBackupWorkflow`: weekly `pg_dump` to R2, keep the last few
- [ ] `QuotaWatchWorkflow`: `quota_warning` log events and an admin-page banner at about 80%
- [ ] Delete flow: `DELETE /jobs/:id` removes artifacts, evidence, and rows; account deletion in `/settings`

**Reliability (README: Failure Handling and Reliability):**
- [ ] Tune per-activity retry policies (aggressive for LLM rate limits, conservative for search, rare for render); confirm `QuotaExhaustedError` is non-retryable
- [ ] Surface actionable failure reasons in the UI (no raw stack traces)
- [ ] Verify the database cold-start retry path under load

**Observability (ARCH §13):**
- [ ] Structured JSON logs correlated by `jobId` and `workflowId`/`runId`, with size-limited rotation
- [ ] Admin page: job funnel, per-stage failure rate, incomplete-check rate, render time, quota use against ceilings
- [ ] Confirm the uptime monitor alerts correctly (stop the API and check)

**Security review (ARCH §12):**
- [ ] SSRF re-test, CORS check, exposed-ports audit, secret-handling audit, signed-URL expiry, per-user isolation audit, and prompt-injection fixtures
- [ ] Free-tier terms checklist: for each provider, confirm commercial-use, region, and data-reuse terms; disclose in the UI which providers process user input

**Recovery drills (ARCH §11.7):**
- [ ] Drill 1: kill the worker mid-render and confirm the job resumes
- [ ] Drill 2: destroy and rebuild the VM from the repository; confirm reconciliation marks in-flight jobs `workflow_lost` and retries resume from persisted artifacts
- [ ] Drill 3: restore the app database from a backup dump into a fresh Neon project

**Load:**
- [ ] Basic load test to find the single-VM ceiling (concurrent jobs, render queue depth); record the result in ARCH §11.3

**Exit criteria:** you can answer "what is our failure rate on the render stage this week" and "how close are we to each quota" from the admin page; all three recovery drills have been run and documented; artifacts expire on schedule.

---

## Phase 13 — Documentation & Acceptance  ·  Size S–M

*Prove the MVP works, unassisted, on the stack you actually deploy.*

- [ ] Write user-facing setup and deployment docs (local dev, VM provisioning, Vercel, Neon, R2) referencing `README.md` and `ARCHITECTURE.md`
- [ ] Publish the free-tier terms checklist and the "what happens when a free tier changes" runbook
- [ ] End-to-end acceptance run against the README's **MVP Scope** checklist on the **deployed** stack: idea → script → claims → evidence → assessments → review/revision → approved script → rendered video → download
- [ ] Re-run the evaluation and include final scores and limitations in the project write-up
- [ ] Check that `README.md`, `ARCHITECTURE.md`, and `todo.md` agree (job states, endpoints, limits, phase numbers)

**Exit criteria:** the MVP scope from the README works unassisted on the public free-tier deployment, with a documented local dev stack and a tested recovery procedure.

---

## Phase 14 — Optional, Post-MVP Backlog

*Not scheduled. Pull forward only when the above is stable. Items match README "Future Enhancements" and ARCH §17.*

**Infrastructure and providers**
- [ ] Paid / managed deployment mode and a documented Free → Paid migration (database dump/restore, object copy, `DEPLOYMENT_MODE` flip)
- [ ] Full observability stack: OpenTelemetry, Grafana, Prometheus, Loki, queue-depth alerting
- [ ] Remotion-based templates and richer motion graphics (after verifying resource use and license terms)
- [ ] Generative image provider as a media fallback behind an `ImageGenProvider` interface
- [ ] Email notifications behind an `EmailProvider` interface
- [ ] GPU-accelerated encoding, only if render time becomes the bottleneck on paid infrastructure

**Publishing (explicit, user-initiated only)**
- [ ] Check YouTube and TikTok app-review, audit, and quota requirements before any build work
- [ ] OAuth connect/disconnect, encrypted token storage (`platform_accounts`), `publishContent` with a deterministic `idempotency_key` per `(jobId, provider)` written before the network call, `POST /jobs/:id/publish`, and `publish_records`

**Product**
- [ ] Better source-quality, recency, and corroboration signals; more source providers
- [ ] Claim dependency mapping; claim-level timestamps mapped directly to video segments
- [ ] Human review queues and collaborative approval workflows (role-based access)
- [ ] Multi-style templates and brand kits
- [ ] Multiple script/video variants for the same idea
- [ ] Multi-language generation and evidence retrieval
- [ ] Analytics sync for post performance
- [ ] User-configurable evidence policies and source preferences
- [ ] Larger evaluation datasets and more advanced source-conflict resolution

**Explicit non-goals for the MVP** (revisit only if user demand is clear): a full professional timeline editor, a deep analytics/attribution suite, enterprise organization management, and guaranteed factual accuracy.