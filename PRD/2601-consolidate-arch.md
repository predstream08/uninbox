# PRD: Consolidate Deployment Architecture to Vercel

## Goal
Consolidate the current multi-service setup into a smaller set of Vercel-deployable apps, reduce operational complexity, and replace BullMQ worker processes with Vercel Workflow for durable, serverless background work.

## Scope
- Inventory all deployable apps and packages in this repo.
- Define a target deployment architecture on Vercel.
- Provide a migration plan with phased steps and risk notes.
- Map current BullMQ + cron workloads to Vercel Workflow.

## Non-goals
- Implementing the migration in this PRD.
- Rewriting product functionality.

## Current Architecture Inventory

### Deployable apps (runtime services)
1. `apps/web` (Next.js web app)
   - Entry: Next `app` router
   - Depends on Platform, Storage, Realtime, Billing, Mail-Bridge via env URLs

2. `apps/platform` (Hono API + tRPC)
   - Responsibilities: auth, orgs, spaces, workflows, services endpoints, realtime gateway
   - Exposes tRPC router used by web and other services

3. `apps/storage` (Hono API)
   - Responsibilities: S3 presign, proxy (avatars, attachments, inline), deletion APIs
   - Uses S3-compatible storage and Redis cache

4. `apps/mail-bridge` (Hono + worker mode)
   - Responsibilities: Postal webhooks, mail processing, routing rules
   - Has two modes: handler/worker/dual
   - Uses BullMQ for background mail processing

5. `apps/worker` (Hono + BullMQ + cron)
   - Responsibilities: DNS check jobs, session cleanup, job endpoints
   - Runs cron jobs inside a long-running process

6. `ee/apps/billing` (Hono API)
   - Responsibilities: Stripe billing, webhooks, plan management

7. `ee/apps/command` (Next.js EE web app)
   - Separate web app; keep as a separate Vercel project

### Shared packages
- `packages/database`: Drizzle ORM + MySQL/Planetscale
- `packages/realtime`: Pusher client/server helpers
- `packages/otel`: OpenTelemetry setup
- `packages/hono`: Hono server helpers
- `packages/utils`: utilities
- `packages/tiptap`: editor helpers

### External dependencies (inferred from env)
- MySQL/Planetscale (data)
- Redis (cache + queues)
- S3 (attachments/avatars)
- Pusher/Soketi (realtime)
- Postal (inbound mail + SMTP + webhooks)
- Stripe (billing)
- Unkey (rate limits)
- Turnstile, PostHog, OTEL exporters

## Current Runtime Flows (high-level)
1. Web UI -> Platform API (tRPC + REST)
2. Web UI -> Storage API (attachments, avatars)
3. Platform -> Mail-Bridge (postal org/domain management + outbound sending)
4. Mail-Bridge inbound webhook -> BullMQ -> Mail processor worker
5. Worker cron -> BullMQ -> DNS checks + session cleanup

## Target Architecture (Vercel)

### Vercel Projects
1. **Web App**: `apps/web` (Next.js)
2. **EE Command App**: `ee/apps/command` (Next.js)
3. **Services App**: consolidate Hono services into a single Vercel deployment
   - Combine `apps/platform`, `apps/storage`, `apps/mail-bridge` (handler mode only), `ee/apps/billing`
   - Route by path prefix inside a single Hono app

### Why 3 projects
- Keep EE command app separate as requested.
- Consolidate service APIs to reduce cross-service networking and env sprawl.
- Avoid long-running processes by moving background work to Vercel Workflow.

## Vercel Workflow Migration (replace BullMQ)

### Why Vercel Workflow
- Durable execution with retries and persistence
- No long-running workers
- Native observability in Vercel
- Supports step-based and resumable workflows

### Workload mapping
1. **Mail processing (currently BullMQ worker)**
   - Trigger: `POST /postal/mail/inbound/:orgId/:mailserverId`
   - New flow: handler enqueues a Workflow run
   - Workflow steps (example):
     - Parse inbound email payload
     - Resolve org/mailserver
     - Persist convo + entries
     - Presign/upload attachments to S3
     - Send realtime notifications

2. **DNS checks (currently Cron + BullMQ)**
   - Scheduled workflow (Vercel Cron or Workflow schedule)
   - Step to fetch active domains
   - Step per domain to run `checkDns`
   - Immediate DNS checks become Workflow trigger endpoint

3. **Session cleanup (currently Cron)**
   - Scheduled workflow with single step calling `cleanupExpiredSessions`

### Implementation shape
- Add `workflow` package to Services App
- Create `app/workflows/*` and `app/steps/*` for each background task
- Expose thin HTTP routes to start workflows

## Migration Plan (phased)

### Phase 1: Consolidate Services API
- Create a unified Hono app in Services project
- Mount routers with prefixes:
  - `/platform/*`
  - `/storage/*`
  - `/mail-bridge/*`
  - `/billing/*`
- Standardize auth middleware per route group
- Update Web and EE app env URLs to single Services base URL

### Phase 2: Introduce Vercel Workflow
- Add Workflow dependency to Services App
- Implement workflows:
  - `mailProcessorWorkflow`
  - `dnsCheckWorkflow`
  - `sessionCleanupWorkflow`
- Update inbound mail webhook to trigger `mailProcessorWorkflow`
- Update DNS job endpoints to trigger `dnsCheckWorkflow`

### Phase 3: Retire BullMQ + Worker App
- Remove BullMQ workers from `apps/worker` and `apps/mail-bridge`
- Remove long-running cron jobs
- Delete `apps/worker` deployment

### Phase 4: Cleanup and hardening
- Trim Redis usage to caching only
- Audit timeouts for large emails and attachments
- Add observability hooks where needed

## Risks and Mitigations
- **Long-running mail processing**: use Workflow steps to split work and avoid timeouts.
- **Attachment uploads**: use presigned URLs and resumable steps.
- **Postal webhooks**: keep handler as a lightweight trigger to workflow.
- **Queue semantics**: ensure idempotency for workflow replays.

## Open Decisions
- Whether to retain Redis for caching only, or replace with Vercel KV.
- Whether to replace Soketi/Pusher with managed service (Pusher, Ably) to reduce infra.
- Whether to keep Postal external or migrate to a managed email provider.

## Success Criteria
- All runtime services deploy to Vercel with at most 3 projects.
- Background jobs run via Vercel Workflow with no long-running workers.
- Reduced env variables and cross-service URL dependencies.
- Functional parity with current mail, storage, and billing flows.
