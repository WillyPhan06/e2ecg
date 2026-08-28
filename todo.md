# e2ecg — Development To-Do

Derived from `README.md` and `architecture.md`. Phases are ordered so each one produces something runnable before the next begins — like building a house from foundation to roof rather than framing every room at once. Every phase should end in a demoable state, even if the demo is rough.

Checkbox legend: `[ ]` not started · `[~]` in progress · `[x]` done

---

## Phase 0 — Repository & Environment Foundation

*The slab everything else gets poured on: repo layout, config, and the Free/Paid switch itself, before a single pipeline stage exists.*

- [ ] Initialize monorepo (pnpm workspaces or Turborepo): `apps/frontend`, `apps/api`, `apps/workers`, `packages/shared`
- [ ] Set up shared TypeScript config, ESLint, Prettier across packages
- [ ] Define the full environment variable schema (`.env.example`) covering both `DEPLOYMENT_MODE=free` and `DEPLOYMENT_MODE=paid` variables (architecture.md §10.2)
- [ ] Build `scripts/configure.ts` — the interactive local configuration menu (architecture.md §10.3)
  - [ ] Free/Paid mode prompt
  - [ ] Writes `.env.local` with mode-appropriate variables
  - [ ] Paid mode: prompts for and lightly validates each managed-service API key
  - [ ] Free mode: points to `docker-compose.free.yml`
- [ ] Write `docker-compose.free.yml`: Postgres, Redis, MinIO, self-hosted Temporal, Grafana/Prometheus/Loki containers
- [ ] Set up CI (GitHub Actions): lint, typecheck, unit tests on every PR
- [ ] Write top-level `CONTRIBUTING.md` or dev-setup doc pointing to `npm run configure`

**Exit criteria:** a fresh clone can run `npm run configure` (Free), `docker compose up`, and reach a "Hello World" page on `localhost`.

---

## Phase 1 — Auth & Data Foundations

*Before anything can be a "job," the app needs to know who's asking and somewhere durable to write things down.*

- [ ] Define `AuthProvider` interface (architecture.md §10.1)
- [ ] Implement `ClerkAuthProvider` (paid mode)
- [ ] Implement `AuthJsProvider` self-hosted against Postgres (free mode)
- [ ] Provider factory wired to `DEPLOYMENT_MODE`
- [ ] Postgres schema + Prisma migrations for: `users`, `jobs`, `job_steps`, `artifacts`, `platform_accounts`, `publish_records` (architecture.md §5)
- [ ] Add indexes: `jobs(user_id, created_at desc)`, `job_steps(job_id)`, `artifacts(job_id)`
- [ ] Next.js app shell: layout, navigation, auth-aware routing (`/dashboard`, `/jobs/new`, `/jobs/[id]`, `/settings/integrations`)
- [ ] Registration/login flow wired to the active `AuthProvider`
- [ ] API middleware: attach `userId` from verified session to every request context
- [ ] Enforce per-user row-level isolation at the query layer (README Security and Compliance Notes)

**Exit criteria:** a user can sign up, log in, and see an empty dashboard scoped to their account, in both Free and Paid mode.

---

## Phase 2 — Job Orchestration Core

*This is the expo station going up before any food is cooked — the workflow engine and job bookkeeping need to exist before planning, media, or voice logic has anywhere to report to.*

- [ ] Stand up Temporal: self-hosted server via Docker (free) and Temporal Cloud namespace (paid)
- [ ] Define `VideoGenerationWorkflow` skeleton with stubbed activities (architecture.md §3.3)
- [ ] Define one Temporal task queue per stage: `e2ecg-planning`, `e2ecg-media`, `e2ecg-voice`, `e2ecg-composition`, `e2ecg-render`, `e2ecg-publish`
- [ ] `JobsModule`: `POST /jobs` (validate input, insert `Job` row, start workflow with id `job:{jobId}`)
- [ ] `GET /jobs`, `GET /jobs/:id`, `GET /jobs/:id/steps`
- [ ] `recordStep()` helper: writes `job_steps` row + publishes Redis pub/sub event on every activity start/finish
- [ ] `StreamModule`: `GET /jobs/:id/stream` SSE endpoint relaying pub/sub events (architecture.md §9)
- [ ] Frontend: idea submission form (`/jobs/new`) matching the README's Submit Idea fields (idea, audience, tone, platforms, constraints)
- [ ] Frontend: live progress view (`/jobs/[id]`) subscribed to the SSE stream, with fallback to `GET /jobs/:id/steps` on reconnect
- [ ] Idempotency check: re-submitting the same job id returns the existing workflow handle, not a duplicate run

**Exit criteria:** submitting an idea creates a job that visibly walks through stubbed stages in the UI in real time, end to end, with no real AI calls yet.

---

## Phase 3 — AI Planning & Script Generation

*The first real "station" comes online: turning a one-line idea into a structured script an entire pipeline can build on.*

- [ ] Define `LLMProvider` interface (architecture.md §10.1)
- [ ] Implement `AnthropicLLMProvider` using tool use for structured plan/script output
- [ ] Implement `OllamaLLMProvider` (self-hosted Llama 3.1/Mistral) and/or `GroqLLMProvider` (free tier, open models)
- [ ] Design prompt templates that incorporate audience, tone, duration, CTA, and language constraints
- [ ] Implement `planContent` activity
- [ ] Implement `draftScript` activity
- [ ] Validate LLM output against a typed `Script` schema; retry-with-repair on malformed output
- [ ] Persist `script.json` as an `Artifact` via `StorageProvider`
- [ ] Unit tests: prompt → schema-valid output for both providers
- [ ] Frontend: display plan/script artifact in the job detail view once available

**Exit criteria:** a submitted idea produces a real, structured script visible in the UI, using either provider depending on `DEPLOYMENT_MODE`.

---

## Phase 4 — Media & Voice Pipeline

*Two stations that don't depend on each other — casting (media) and narration (voice) — built to run in parallel once the script exists, per the workflow design in architecture.md §3.3.*

- [ ] Integrate stock media APIs (Pexels, Pixabay) for licensed asset search
- [ ] Define `ImageGenProvider` interface
- [ ] Implement `ReplicateImageProvider` (paid fallback) and `StableDiffusionLocalProvider`/disabled stock-only path (free mode)
- [ ] Implement `collectMedia` activity: stock search first, generative fallback only on no-match, tagged with provenance metadata
- [ ] Define `TTSProvider` interface
- [ ] Implement `ElevenLabsProvider`/`PlayHtProvider` (paid) and `PiperTTSProvider`/`EdgeTTSProvider` (free)
- [ ] Implement `generateVoice` activity, respecting the `language` constraint from job input
- [ ] Confirm workflow runs `collectMedia` and `generateVoice` concurrently (`Promise.all` per architecture.md §3.3)
- [ ] Persist media and voice artifacts with metadata (duration, resolution, provider used)
- [ ] Frontend: show media thumbnails and an audio player in the job detail view as they land

**Exit criteria:** a job produces real narrated audio and a matched set of media assets, sourced correctly for licensing in both modes.

---

## Phase 5 — Composition & Rendering

*The most resource-hungry station in the whole line — isolated on its own queue precisely so a render spike doesn't starve everything upstream of it (README/architecture rationale for per-stage worker pools).*

- [ ] Set up Remotion project for code-driven timeline composition
- [ ] Implement `composeTimeline` activity: assembles captions, transitions, audio sync into a render spec
- [ ] Implement `renderVideo` activity: Remotion render + FFmpeg encode to platform-ready formats
- [ ] Deploy render workers on their own autoscaling group/queue, separate from lighter stages
- [ ] Add captions export (`.srt`) alongside the final video
- [ ] Store `final.mp4` and captions in the `render/` storage path (architecture.md §8)
- [ ] Load-test render duration/cost to inform the GPU-encoding decision flagged in architecture.md §14
- [ ] Frontend: video preview player in the job detail view once rendering completes

**Exit criteria:** a job produces a finished, watchable, platform-formatted video with captions, end to end.

---

## Phase 6 — Publishing Integrations

*The pipeline finally leaves the building: the last hand-off before content is live on someone else's platform, so idempotency here matters more than anywhere else.*

- [ ] `IntegrationsModule`: YouTube OAuth connect/disconnect flow
- [ ] `IntegrationsModule`: TikTok OAuth connect/disconnect flow
- [ ] Encrypt OAuth tokens at rest (KMS envelope encryption in paid mode; locally-managed key in free mode — architecture.md §12)
- [ ] Implement content safety filter activity between `composeTimeline` and `publishContent`
- [ ] Implement `publishContent` activity with a deterministic `idempotency_key` per `(jobId, provider)` (architecture.md §7)
- [ ] `POST /jobs/:id/publish` endpoint (per-provider selection)
- [ ] Insert `publish_records` row with remote content id/URL on success
- [ ] Handle and surface provider-specific upload errors distinctly from pipeline errors
- [ ] Frontend: publish button + resulting platform links in the job detail view

**Exit criteria:** a finished video can be published to a connected YouTube and/or TikTok account, with a retried publish never creating a duplicate upload.

---

## Phase 7 — Notifications, History & Retry UX

*Closing the loop back to the user — the parts of the README's "human-visible checkpoints" goal that don't live inside the pipeline itself.*

- [ ] Define `EmailProvider` interface
- [ ] Implement `ResendEmailProvider`/`PostmarkProvider` (paid) and `SmtpEmailProvider` (free)
- [ ] Send completion (and failure) email notifications
- [ ] Build job history dashboard: list, filter, search (README "Review Output and History")
- [ ] Job detail page: full artifact viewer (script, media, voice, video), timestamps, publish links
- [ ] Implement re-run and duplicate-job actions
- [ ] `POST /jobs/:id/retry`: signal the existing workflow to retry from the failed activity, preserving completed artifacts (architecture.md §6)
- [ ] Tune per-activity retry policies (initial interval, backoff coefficient, max attempts) — aggressive for LLM 429s, conservative for publish rate limits
- [ ] Surface clear, actionable failure reasons in the UI (not raw stack traces)

**Exit criteria:** a failed job can be retried from where it broke, a user can browse and search their full job history, and they get notified without watching the screen.

---

## Phase 8 — Observability & Reliability Hardening

*Instrumentation that turns "it feels slow sometimes" into an actual number you can act on — this is what makes the README's success metrics real rather than aspirational.*

- [ ] Instrument OpenTelemetry spans across API → workflow → activities
- [ ] Paid mode: export traces/metrics to Grafana Cloud or Datadog
- [ ] Free mode: stand up self-hosted Grafana + Prometheus + Loki in `docker-compose.free.yml`
- [ ] Build dashboards: per-stage duration, per-stage failure rate, external/local provider latency, quota consumption
- [ ] Alerting: stage failure-rate threshold, queue-depth-per-task-queue threshold
- [ ] Structured JSON logging correlated by `jobId` and Temporal `workflowId`/`runId`
- [ ] Chaos test: kill a worker mid-render, confirm Temporal reschedules and the job resumes without data loss
- [ ] Basic load test to find the free-mode single-VM ceiling (§14 in architecture.md)

**Exit criteria:** you can answer "what's our failure rate on the render stage this week" and "did that retry actually pick up where it left off" from dashboards, not guesswork.

---

## Phase 9 — Deployment & Launch Readiness

*Standing up both houses for real — the free-tier VM and the managed-services stack — and proving the migration path between them actually works.*

- [ ] Paid mode: deploy frontend to Vercel; API + workers to Fly.io/Railway (or ECS/Fargate); connect Temporal Cloud, managed Postgres, Redis, S3/R2
- [ ] Free mode: provision an Oracle Cloud "Always Free" VM, deploy full `docker-compose.free.yml` stack
- [ ] Dry-run the Free → Paid migration path: Postgres dump/restore, MinIO → S3/R2 object copy, `DEPLOYMENT_MODE` flip (architecture.md §11.2)
- [ ] Security review pass: token encryption, signed URL expiry, per-user isolation audit, content safety filter coverage
- [ ] Acceptance test against the README's MVP Scope checklist end to end, in both modes
- [ ] Write user-facing setup docs referencing `README.md` and `architecture.md`

**Exit criteria:** the MVP scope from the README works, unassisted, in both Free and Paid mode, and switching modes is a documented, tested procedure rather than a leap of faith.

---

## Post-MVP Backlog (Future Enhancements)

Not scheduled into a phase yet — pull forward as needed once the above is stable (README "Future Enhancements" / "Non-Goals for MVP"):

- [ ] Multi-style templates and brand kits
- [ ] Automatic A/B variant generation for the same idea
- [ ] Team collaboration and approval workflow (role-based access)
- [ ] Analytics sync for post-publish performance
- [ ] Multi-language localization pipeline
- [ ] Human-in-the-loop editing step before final publish
- [ ] Full professional timeline editor (explicit non-goal for MVP — revisit only if user demand is clear)
- [ ] Deep analytics/attribution suite (explicit non-goal for MVP)
- [ ] Enterprise organization management (explicit non-goal for MVP)