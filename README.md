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

## Suggested Architecture

- Frontend: Web dashboard for job creation, progress tracking, and history
- Backend API: Auth, job management, orchestration endpoints
- Worker Services: Queue-based pipeline stages for plan, assets, voice, compose, upload
- Database: Users, jobs, artifacts, progress events, platform connections
- Object Storage: Generated scripts, media files, audio, video exports
- WebSocket or SSE: Live progress updates
- Scheduler and Retry Logic: Recover from transient API failures

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

## Recommended Tech Stack for Web App First

- Frontend: Next.js with TypeScript
- Backend: Node.js service layer or Next.js API routes
- Queue: BullMQ or similar Redis-backed queue
- Database: PostgreSQL
- ORM: Prisma
- Storage: S3-compatible object storage
- Real-time updates: Server-Sent Events or WebSocket
- Auth: Clerk, Auth0, or custom JWT plus session strategy
- Deployment: Vercel for frontend, container platform for workers

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
- Encrypt OAuth tokens and sensitive credentials
- Apply role-based access if teams are introduced
- Add content safety filters before publishing
- Respect platform terms and media licensing requirements
- Maintain clear provenance for generated and sourced assets

## Failure Handling and Reliability

- Retry transient failures with exponential backoff
- Mark unrecoverable failures with actionable user-facing errors
- Allow resume and retry from failed step where possible
- Add idempotency keys for upload and publish operations
- Track external API latency and quota usage

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
2. Job orchestration and progress event model
3. AI script generation integration
4. Media pipeline integration
5. Voice generation integration
6. Video composition and rendering pipeline
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
