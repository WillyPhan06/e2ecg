# e2ecg

End to End Content Generation

## Overview

e2ecg is a web app that turns a user idea into a ready-to-publish short video.

A user submits a content idea, and the system orchestrates an automated pipeline:
- Plan and script the content with an AI model
- Gather media assets (images and video clips)
- Generate AI voice-over from the script
- Compose and render the final video
- Upload to social platforms (YouTube, TikTok)
- Notify the user when each stage is complete

The app is designed to be transparent: users can inspect live progress for every step, review past jobs, and manage their own account and generated content history.

## Product Goals

- Reduce the manual work required to create social-ready video content
- Provide end-to-end automation with human-visible checkpoints
- Keep users in control through progress views, editable job settings, and job history
- Support scalable, multi-user operation with secure account logic

## Core User Experience

### 1) Submit Idea

Users enter:
- Content idea or topic
- Optional audience and tone
- Target platform (YouTube, TikTok, both)
- Optional constraints (duration, style, call-to-action, language)

### 2) Track Generation Pipeline

Each job displays real-time status for every stage:
- Queued
- Planning
- Script Drafting
- Media Collection
- Voice Generation
- Video Composition
- Uploading
- Completed or Failed

Users can inspect per-step logs and outputs as they become available.

### 3) Review Output and History

Users can:
- Watch generated videos
- View script, media, and voice artifacts
- See upload links and timestamps
- Re-run, duplicate, or revise previous jobs
- Browse account-level history with filtering and search

## End-to-End System Flow

1. User submits idea via web app
2. API creates a generation job
3. Job enters async queue
4. AI planning service creates a content plan and script
5. Asset service gathers licensed or policy-compliant media
6. Voice service generates narration audio from script
7. Composition service assembles timeline, captions, transitions, and audio
8. Render service exports video in platform-ready formats
9. Publishing service uploads to YouTube and or TikTok
10. Notification service updates in-app progress and sends completion alerts

## Recommended Architecture

Think of the whole system like a restaurant that takes custom orders (ideas), routes them through several specialized kitchen stations (planning, media, voice, render, publish), and keeps the customer updated on a screen above the counter (progress UI) — while a separate "expo" role makes sure nothing gets lost between stations, even if one station briefly goes down.

- **Frontend** — Next.js (TypeScript), App Router. Handles job creation, live progress, and history browsing. Server Components fetch job history directly from the API for fast initial loads; client components handle the live progress stream.
- **Backend API** — A dedicated service (NestJS or Fastify on Node.js), separate from the Next.js app, rather than using Next.js API routes for orchestration.
  - *Why:* Next.js API routes are convenient for CRUD, but a multi-stage async pipeline needs its own lifecycle, scaling, and deploy cadence independent of the frontend. This is the difference between a food truck (one window does everything) and a real kitchen with stations — fine at tiny scale, but you outgrow the food truck fast once orders (jobs) get complex.
- **Orchestration Engine** — A durable workflow engine (Temporal, or the lighter-weight Inngest) coordinates the pipeline stages, instead of relying on a plain job queue alone.
  - *Why:* A basic queue (BullMQ) is like a relay race — each worker passes a baton to the next, but if a runner drops it mid-race (a crash, a timeout), nobody automatically knows how to pick up where it left off. A durable workflow engine is more like a conductor with the full score in front of them: it remembers exactly which stage each job was on, retries only the failed step, and survives a crash without losing the job's place. This maps directly to the "resume and retry from failed step" requirement already in this spec.
  - BullMQ (Redis-backed) still has a place for simple, single-hop background tasks (e.g., sending a notification email) where full durability is overkill.
- **Worker Services** — Independently deployable, independently scalable workers per stage: planning, media, voice, composition, render, publish.
  - *Why:* Rendering video is CPU/GPU-heavy; calling an LLM for a script is I/O-bound and cheap. Running them on the same fleet means you either over-provision for render spikes or starve the render stage. Separating them is like keeping the grill station and the salad station on different counters — each can be staffed (scaled) according to its own order volume.
- **Database** — PostgreSQL, accessed via Prisma (or Drizzle if you want lighter-weight, more SQL-like query control as the schema grows).
- **Object Storage** — S3-compatible storage for scripts, media, audio, and video exports. Cloudflare R2 is worth evaluating over raw S3 for its zero-egress-fee model, since users will be downloading/streaming finished videos often.
- **Real-time Updates** — Server-Sent Events (SSE), not WebSockets, for progress streaming.
  - *Why:* Progress updates only flow one direction — server to client. SSE is a one-way radio broadcast; WebSockets are a two-way walkie-talkie. Since the app never needs to send data back over that same channel, SSE gets the same live-update experience with simpler infrastructure (plain HTTP, no sticky-session load balancing, easy to proxy).
- **Auth** — Clerk for MVP speed (hosted UI, session handling, social login out of the box), with a documented path to a custom JWT/session layer later if pricing or control becomes a constraint.
- **Deployment** — Frontend on Vercel; backend API and workers on a container platform with per-service autoscaling (Fly.io, Railway for early stage; AWS ECS/Fargate or GKE once volume justifies it). Each worker type gets its own autoscaling group, so a spike in render jobs doesn't spin up unnecessary planning workers.
- **Observability** — OpenTelemetry tracing across API → orchestrator → workers, exported to a dashboard (Grafana Cloud or Datadog). This is what actually makes "failure rate per pipeline stage" and "external API latency and quota usage" (both already goals in this spec) measurable rather than aspirational.

## High-Level Components

- Auth and User Accounts
- Idea Intake and Prompt Builder
- AI Planning and Script Generation
- Media Discovery and Curation
- AI Voice Synthesis
- Video Composition and Rendering
- Platform Publishing Integrations
- Progress Timeline and History UI
- Notifications and Activity Feed
- Orchestration and Workflow State (new — the "expo" role that tracks every job across every station)

## Recommended Tech Stack

| Layer | Recommendation | Why |
|---|---|---|
| Frontend | Next.js + TypeScript | Server Components for fast history/dashboard loads; large ecosystem for auth, forms, streaming UI |
| Backend API | NestJS (or Fastify) on Node.js | Decoupled from frontend deploys; structured module system suits an orchestration-heavy domain |
| Orchestration | Temporal (or Inngest for a lighter self-hosted footprint) | Durable, resumable multi-step workflows with built-in retry/backoff — matches the reliability requirements already in this spec |
| Lightweight queue | BullMQ (Redis) | For simple, single-hop background jobs (emails, cleanup tasks) where full durability isn't needed |
| Database | PostgreSQL + Prisma | Relational integrity for users/jobs/artifacts/publish records; Prisma keeps schema and migrations in one place |
| Object Storage | S3-compatible (Cloudflare R2 or AWS S3) | Cheap, durable storage for scripts, media, audio, video; R2 avoids egress fees for frequent downloads |
| Real-time updates | Server-Sent Events | One-directional progress data doesn't need full-duplex WebSockets; simpler to scale and proxy |
| Auth | Clerk (or Auth0) | Fast to ship, handles session/token complexity so the team can focus on the pipeline |
| AI Planning/Script | Anthropic API (Claude) with tool use for structured plan/script output | Structured outputs make it easy to hand a clean script object to the next stage |
| Media Discovery | Stock APIs (Pexels, Pixabay) + AI image generation fallback (Replicate/fal.ai) when no licensed match is found | Keeps licensing compliant by default, with generative media as a fallback rather than a first resort |
| Voice Synthesis | ElevenLabs or Play.ht | Mature TTS APIs with multiple voices/languages, fits the "language" constraint already in the idea-submission form |
| Video Composition/Render | Remotion (React + TypeScript video framework) for template-driven composition, with FFmpeg workers for final encoding/format conversion | Since the whole stack is already React/TypeScript, composing video as code (like writing a webpage that becomes a video) keeps one shared language across frontend and render pipeline, rather than introducing an unrelated video-editing SDK |
| Publishing | YouTube Data API v3, TikTok Content Posting API | Official upload APIs; OAuth tokens stored encrypted per user |
| Notifications | SSE (in-app) + Resend or Postmark (email) | Matches the "notify user when each stage completes" requirement without adding a push-notification dependency for MVP |
| Deployment | Vercel (frontend) + Fly.io/Railway → ECS/Fargate (backend/workers) | Cheap to start, straightforward path to scale each worker type independently |

## Data Model (Conceptual)

### User
- id
- email
- password hash or external auth id
- profile metadata
- created at

### Project or Job
- id
- user id
- idea input
- target platforms
- status
- created at and updated at

### Job Step
- id
- job id
- step name
- status
- started at and finished at
- logs and error details

### Artifact
- id
- job id
- type (script, image, clip, voice, final video)
- uri
- metadata

### Platform Account
- id
- user id
- provider (youtube or tiktok)
- oauth tokens (encrypted)
- token expiry

### Publish Record
- id
- job id
- provider
- remote content id
- remote url
- published at
- response payload summary

## API Surface (Draft)

- POST /auth/register
- POST /auth/login
- POST /jobs
- GET /jobs
- GET /jobs/:id
- GET /jobs/:id/steps
- GET /jobs/:id/artifacts
- GET /jobs/:id/stream (SSE endpoint for live step/status updates)
- POST /jobs/:id/retry
- POST /jobs/:id/publish
- GET /integrations
- POST /integrations/:provider/connect
- DELETE /integrations/:provider

## Progress and History Requirements

- Every pipeline stage emits structured status events
- UI displays both stage-level and overall progress
- Persist all job events for later audit and history browsing
- Include failure reasons and retry metadata
- Show final publish links when available

## Security and Compliance Notes

- Enforce per-user data isolation
- Encrypt OAuth tokens and sensitive credentials (KMS-backed at rest)
- Apply role-based access if teams are introduced
- Add content safety filters before publishing
- Respect platform terms and media licensing requirements
- Maintain clear provenance for generated and sourced assets

## Failure Handling and Reliability

- Retry transient failures with exponential backoff (native to the orchestration engine's workflow retries)
- Mark unrecoverable failures with actionable user-facing errors
- Allow resume and retry from failed step where possible — this is the core reason a durable workflow engine (Temporal/Inngest) is recommended over a plain queue
- Add idempotency keys for upload and publish operations
- Track external API latency and quota usage via OpenTelemetry traces per stage

## MVP Scope

Phase 1 should focus on:
- User authentication and account management
- Idea submission and job creation
- Script generation
- Basic media retrieval
- Voice generation
- Simple video composition
- Manual review plus one-click publish
- Real-time progress screen
- Job history dashboard

## Future Enhancements

- Multi-style templates and brand kits
- Automatic A and B variants for the same idea
- Team collaboration and approval workflow
- Analytics sync for post performance
- Multi-language localization pipeline
- Human-in-the-loop editing before final publish

## Non-Goals for MVP

- Full professional timeline editor
- Deep analytics and attribution suite
- Enterprise organization management

## Development Roadmap

1. Project setup and auth foundation
2. Job orchestration and progress event model (stand up Temporal/Inngest + Postgres schema)
3. AI script generation integration
4. Media pipeline integration
5. Voice generation integration
6. Video composition and rendering pipeline (Remotion + FFmpeg workers)
7. YouTube and TikTok publishing integration
8. Notifications, history UI, and reliability hardening

## Success Metrics

- Time from idea submission to first publishable draft
- Job completion rate
- Failure rate per pipeline stage
- Publish success rate by platform
- User retention and repeat job creation

## Project Name

e2ecg means End to End Content Generation.

## Current Status

Planning and specification stage.

Implementation should begin with web app foundations, then progressively connect the generation and publishing pipeline.